# ZTWIM Architecture

## Repository Layout

```text
zero-trust-workload-identity-manager/
├── api/v1alpha1/              # CRD type definitions (ZTWIM, SpireServer, SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider)
├── bindata/                   # Static YAML manifests for upstream operands — NEVER hand-edit bindata.go
│   ├── spire-server/          # StatefulSet RBAC, service, service-account
│   ├── spire-agent/           # DaemonSet RBAC, service, service-account
│   ├── spiffe-csi/            # CSIDriver, privileged role-binding, service-account
│   ├── spire-controller-manager/  # Webhook service, validating webhook, RBAC
│   ├── spire-oidc-discovery-provider/  # Service, service-account, external-cert RBAC
│   └── spire-bundle/          # Bundle role, role-binding
├── bundle/                    # OLM bundle (CSV, CRD manifests, metadata)
├── cmd/zero-trust-workload-identity-manager/main.go  # Entrypoint — registers all 5 controllers
├── config/                    # Kustomize overlays (CRDs, RBAC, manager, samples, prometheus)
├── hack/                      # Build helpers, boilerplate
├── pkg/
│   ├── client/                # CustomCtrlClient — unified cache, Create/Update/Patch/Delete wrapper
│   │   └── fakes/             # counterfeiter-generated fakes for testing
│   ├── controller/
│   │   ├── zero-trust-workload-identity-manager/  # ZTWIM controller — status aggregator
│   │   ├── spire-server/      # SpireServer reconciler — StatefulSet, ConfigMaps, Routes, webhooks
│   │   ├── spire-agent/       # SpireAgent reconciler — DaemonSet, SCC, ConfigMap
│   │   ├── spiffe-csi-driver/ # SpiffeCSIDriver reconciler — DaemonSet, CSIDriver
│   │   ├── spire-oidc-discovery-provider/  # OIDC reconciler — Deployment, Route, ClusterSPIFFEID
│   │   ├── status/            # Status manager — condition aggregation
│   │   └── utils/             # Shared utilities — constants, errors, labels, resource comparison, validation
│   ├── operator/assets/       # go-bindata generated file — DO NOT hand-edit
│   └── version/               # Operator and component version constants
├── test/e2e/                  # Ginkgo E2E tests
├── tools/                     # Tool dependencies (controller-gen, etc.)
└── vendor/                    # Tracked vendor directory — commit after `make vendor`
```

## Key Domain Concepts

ZTWIM manages the SPIFFE/SPIRE identity infrastructure on OpenShift clusters. The core flow:

1. **User creates `ZeroTrustWorkloadIdentityManager` CR** (singleton named `cluster`) with `trustDomain` and `clusterName`
2. **User creates operand CRs** (`SpireServer`, `SpireAgent`, `SpiffeCSIDriver`, `SpireOIDCDiscoveryProvider` — each singleton named `cluster`)
3. **Operand controllers reconcile** — each renders static bindata manifests with runtime config (images, ConfigMaps, tolerations) and applies them imperatively
4. **ZTWIM controller aggregates status** — watches all operand CRs, computes overall Ready/Upgradeable conditions, syncs Upgradeable to OLM OperatorCondition

Key abstractions:
- **Trust Domain** — the SPIFFE trust domain (immutable after creation), set on ZTWIM CR
- **Operands** — the four SPIRE components, each with its own CRD and controller
- **Bindata Manifests** — static YAML templates in `bindata/` compiled into `pkg/operator/assets/bindata.go` via `make update-bindata`
- **RELATED_IMAGE_*** — env vars injecting upstream container images, defined in `pkg/controller/utils/constants.go:57-63`

## Controller Details

**Framework**: controller-runtime (NOT library-go). All controllers use `sigs.k8s.io/controller-runtime`.

**Startup sequence** (`cmd/zero-trust-workload-identity-manager/main.go`):
1. Register schemes (clientgo, operator API v1alpha1, securityv1, routev1, operatorv1, spire-controller-manager)
2. Build unified cache via `customClient.NewCacheBuilder()` — label-filtered for managed resources
3. Create controller-runtime manager with leader election, secure metrics (mTLS with OpenShift service CA)
4. Instantiate each controller via `New(mgr)` → `SetupWithManager(mgr)`

| Controller | CRD Watched | Key Resources Created | Predicate |
|---|---|---|---|
| **ZTWIM** | `ZeroTrustWorkloadIdentityManager` | None (status-only) | `GenerationChangedPredicate` on ZTWIM; `operandStatusChangedPredicate` on all operand CRs + OperatorCondition |
| **SpireServer** | `SpireServer` | StatefulSet, ConfigMaps (server, controller-manager, bundle), Services, RBAC, Route, ValidatingWebhook | `GenerationOrOwnerReferenceChangedPredicate` on SpireServer; `ControllerManagedResourcesForComponent(ComponentControlPlane)` on managed resources; `ZTWIMSpecChangedPredicate` on ZTWIM |
| **SpireAgent** | `SpireAgent` | DaemonSet, ConfigMap, SCC, Service, RBAC | Same pattern as SpireServer with `ComponentNodeAgent` |
| **SpiffeCSIDriver** | `SpiffeCSIDriver` | DaemonSet, CSIDriver, ServiceAccount, RBAC | Same pattern with `ComponentCSI` |
| **SpireOIDCDiscoveryProvider** | `SpireOIDCDiscoveryProvider` | Deployment, ConfigMaps, Route, ClusterSPIFFEID, RBAC | Same pattern with `ComponentDiscovery` |

**Controller naming convention**: `zero-trust-workload-identity-manager-<component>-controller` (defined in `pkg/controller/utils/constants.go:6-10`).

## Resource Management

**Apply method**: Imperative Create/Update — NOT Server-Side Apply (SSA).

Each operand controller follows this pattern:
1. Build desired resource from bindata template + runtime config (images, labels, tolerations, config hashes)
2. Check if resource exists via `ctrlClient.Get()`
3. If not found → `ctrlClient.Create()`
4. If found → compare via `ResourceNeedsUpdate()` → `ctrlClient.Update()` or `ctrlClient.UpdateWithRetry()` if changed

`ResourceNeedsUpdate` (`pkg/controller/utils/resource_comparison.go:23`) performs type-aware comparison:
- Labels and annotations: checks desired subset is present
- Type-specific: per-resource-kind spec comparison (e.g., StatefulSet compares replicas, containers, volumes, annotations for config hashes)

**Create-only mode**: When `CREATE_ONLY_MODE=true` env var is set, controllers create resources but skip updates. Checked via `utils.IsInCreateOnlyMode()` (`pkg/controller/utils/utils.go:237`).

**Image resolution**: All upstream container images are injected via `RELATED_IMAGE_*` environment variables read at runtime:

| Env Var | Component | Getter Function |
|---|---|---|
| `RELATED_IMAGE_SPIRE_SERVER` | SPIRE Server | `utils.GetSpireServerImage()` |
| `RELATED_IMAGE_SPIRE_AGENT` | SPIRE Agent | `utils.GetSpireAgentImage()` |
| `RELATED_IMAGE_SPIFFE_CSI_DRIVER` | SPIFFE CSI Driver | `utils.GetSpiffeCSIDriverImage()` |
| `RELATED_IMAGE_SPIRE_OIDC_DISCOVERY_PROVIDER` | OIDC Provider | `utils.GetSpireOIDCDiscoveryProviderImage()` |
| `RELATED_IMAGE_SPIRE_CONTROLLER_MANAGER` | Controller Manager | `utils.GetSpireControllerManagerImage()` |
| `RELATED_IMAGE_NODE_DRIVER_REGISTRAR` | Node Driver Registrar | `utils.GetNodeDriverRegistrarImage()` |
| `RELATED_IMAGE_SPIFFE_CSI_INIT_CONTAINER` | CSI Init Container | `utils.GetSpiffeCsiInitContainerImage()` |

## Feature Gates

No OpenShift feature gates are used. The operator's feature control is via:
- **Create-only mode**: `CREATE_ONLY_MODE` env var (`pkg/controller/utils/utils.go:237`)
- **Federation**: Optional `SpireServer.spec.federation` field — once set, cannot be removed (CEL validation on `spire_server_config_types.go`)
- **Upstream authority**: Optional `SpireServer.spec.upstreamAuthority` field

## Error Classification

Three error types defined in `pkg/controller/utils/errors.go`:

| Error Type | Constant | Effect |
|---|---|---|
| `IrrecoverableError` | Authorization, forbidden, invalid, bad-request, service-unavailable | No requeue — requires manual intervention |
| `RetryRequiredError` | Transient API errors | Requeue with backoff |
| `MultipleInstanceError` | Multiple instances of singleton CRs | Treated as irrecoverable |

`FromClientError()` auto-classifies Kubernetes API errors. `FromError()` checks if error wraps an irrecoverable error.

## OpenShift Integration Points

| Integration | How | Code Reference |
|---|---|---|
| **Routes** | `routev1.Route` for SPIRE server federation and OIDC discovery | `pkg/controller/spire-server/routes.go`, `pkg/controller/spire-oidc-discovery-provider/routes.go` |
| **SecurityContextConstraints** | Custom SCC for SPIRE agent (hostPath, hostPID access) | `pkg/controller/spire-agent/scc.go` |
| **Privileged SCC binding** | CSI driver gets privileged SCC via RoleBinding | `pkg/controller/spiffe-csi-driver/rbac.go` |
| **Service CA** | `service.beta.openshift.io/serving-cert-secret-name` annotation for auto TLS | `pkg/controller/utils/constants.go:53-54` |
| **OLM** | `OperatorCondition` resource for Upgradeable status sync | `pkg/controller/zero-trust-workload-identity-manager/controller.go:575` |
| **Proxy** | `ValidateProxyConfiguration()` checks HTTP_PROXY/HTTPS_PROXY/NO_PROXY and CA bundle | `pkg/controller/utils/proxy.go` |
| **Metrics** | mTLS metrics endpoint via OpenShift service CA, `filters.WithAuthenticationAndAuthorization` | `cmd/zero-trust-workload-identity-manager/main.go:151` |

## Generated Code Inventory

| Path | Generator | Regenerate Command |
|---|---|---|
| `api/v1alpha1/zz_generated.deepcopy.go` | controller-gen | `make generate` |
| `config/crd/bases/*.yaml` | controller-gen | `make manifests` |
| `pkg/operator/assets/bindata.go` | go-bindata | `make update-bindata` |
| `pkg/client/fakes/` | counterfeiter | `go generate ./pkg/client/...` |
| `bundle/manifests/` | operator-sdk | `make bundle` |

**NEVER hand-edit** any of these files. Always use the corresponding make target.

## API Behavioral Contracts

- **Singleton constraint**: All five ZTWIM-owned CRDs enforce `.metadata.name == 'cluster'` via CEL XValidation. Creating a second instance is rejected by the API server.
- **Immutable fields**: `trustDomain`, `clusterName`, `bundleConfigMap` on ZTWIM CR; `persistence.size`, `persistence.accessMode`, `persistence.storageClass` on SpireServer; federation config cannot be removed once set.
- **Owner references**: Each operand controller sets ZTWIM as the owner of its operand CR via `controllerutil.SetControllerReference`. This means deleting ZTWIM cascades to all operand CRs.
- **Status aggregation**: ZTWIM controller does NOT create operand CRs. It only aggregates status from existing operand CRs using generics (`getOperandStatus[T]()` at `pkg/controller/zero-trust-workload-identity-manager/controller.go:417`).
- **Operand state classification**: Uses structured state classification via `classifyOperandState()` with three states: `operandReady`, `operandProgressing`, `operandFailed`. Falls back to message substring matching for backward compatibility.
- **Upgradeable condition**: Set ONLY on OLM `OperatorCondition`, NOT on the ZTWIM CR. False when any existing operand is not ready or create-only mode is enabled. Non-existent CRs are OK for upgrade.
- **Config hash annotations**: SpireServer StatefulSet uses `ztwim.openshift.io/spire-server-config-hash` and `ztwim.openshift.io/spire-controller-manager-config-hash` annotations to trigger rolling updates on config changes.
- **Label predicates**: Managed resources are filtered by `app.kubernetes.io/managed-by=zero-trust-workload-identity-manager` AND `app.kubernetes.io/component=<component>` to prevent cross-controller interference.
- **Unified cache**: All controllers share a single manager cache with label-filtered informers (`pkg/client/client.go:224`). The cache uses `ReaderFailOnMissingInformer: true`.

## Status Management

The `pkg/controller/status/status.go` module provides `status.Manager` for condition aggregation:

- Each controller creates a `status.NewManager(ctrlClient)` at the start of `Reconcile()`
- Conditions are added via `statusMgr.AddCondition(type, reason, message, status)`
- A deferred `statusMgr.ApplyStatus()` call persists conditions on the CR
- The auto-aggregation logic computes an overall `Ready` condition from component conditions

**Condition types per controller**:

| Controller | Condition Types |
|---|---|
| SpireServer | `StatefulSetAvailable`, `ServerConfigMapAvailable`, `ControllerManagerConfigAvailable`, `BundleConfigAvailable`, `TTLConfigurationValid`, `ConfigurationValid`, `ServiceAccountAvailable`, `ServiceAvailable`, `RBACAvailable`, `ValidatingWebhookAvailable`, `RouteAvailable` |
| SpireAgent | `DaemonSetAvailable`, `ConfigMapAvailable`, `SecurityContextConstraintsAvailable`, `ServiceAvailable`, `ServiceAccountAvailable`, `RBACAvailable`, `ConfigurationValid` |
| SpiffeCSIDriver | `DaemonSetAvailable`, `SecurityContextConstraintsAvailable`, `ServiceAccountAvailable`, `CSIDriverAvailable` |
| SpireOIDCDiscoveryProvider | `DeploymentAvailable`, `ConfigMapAvailable`, `RouteAvailable`, `ClusterSPIFFEIDAvailable`, `ServiceAvailable`, `ServiceAccountAvailable`, `RBACAvailable`, `ConfigurationValid` |
| ZTWIM | `OperandsAvailable`, `CreateOnlyMode`, `Ready` |

## Custom Predicates

The operator defines several custom predicates to minimize unnecessary reconciliations:

| Predicate | Behavior | Used By |
|---|---|---|
| `ZTWIMSpecChangedPredicate` | Triggers on Create/Delete only, ignores Update | Operand controllers watching ZTWIM |
| `OwnerReferenceChangedPredicate` | Triggers when owner references are added/removed/modified | Operand controllers |
| `GenerationOrOwnerReferenceChangedPredicate` | Combines `GenerationChangedPredicate` + `OwnerReferenceChangedPredicate` | All operand controllers for their primary CRD |
| `ControllerManagedResourcesForComponent(component)` | Filters by `app.kubernetes.io/managed-by` + `app.kubernetes.io/component` labels | All controllers for their managed resources |
| `operandStatusChangedPredicate` | Triggers only on status changes (not spec changes) | ZTWIM controller watching operand CRs |

## Labeling Strategy

All managed resources use standardized Kubernetes app labels (`pkg/controller/utils/labels.go`):

```
app.kubernetes.io/name:       <resource-name>
app.kubernetes.io/instance:   cluster-zero-trust-workload-identity-manager
app.kubernetes.io/part-of:    zero-trust-workload-identity-manager
app.kubernetes.io/component:  <control-plane|node-agent|csi|discovery>
app.kubernetes.io/managed-by: zero-trust-workload-identity-manager
app.kubernetes.io/version:    <component-version-from-pkg/version>
```

Component-specific label generators: `SpireServerLabels()`, `SpireAgentLabels()`, `SpiffeCSIDriverLabels()`, `SpireOIDCDiscoveryProviderLabels()`, `SpireControllerManagerLabels()` — all accept custom labels that are merged (standard labels take precedence).

## Design References — ZTWIM uses controller-runtime exclusively rather than library-go. This simplifies the reconciliation model (standard Reconcile loop vs. library-go's sync pattern) and allows using standard kubebuilder tooling for CRD generation, webhook scaffolding, and testing with envtest. Trade-off: no access to library-go's built-in operator status helpers — custom `status.Manager` implemented instead.

**2. Bindata for operand manifests** — Upstream SPIRE resource definitions are stored as static YAML in `bindata/` and compiled into Go via go-bindata. Controllers decode these at runtime, overlay runtime configuration (images, labels, config hashes), and apply imperatively. This avoids embedding upstream Go code and allows updating manifests independently of the operator binary.

**3. Custom client with unified cache** — `pkg/client/client.go` wraps controller-runtime client with retry logic (`UpdateWithRetry`, `StatusUpdateWithRetry`) and a pre-configured cache builder that registers informers for all watched resources. The unified cache prevents race conditions between manager and reconciler caches that occurred with separate cache instances.

## Platform Documentation

For generic OpenShift development patterns, conventions, and guidelines:
- [openshift/enhancements `dev-guide/`](https://github.com/openshift/enhancements/tree/master/dev-guide) — development conventions
- [openshift/enhancements `guidelines/`](https://github.com/openshift/enhancements/tree/master/guidelines) — enhancement process
- [openshift/enhancements `CONVENTIONS.md`](https://github.com/openshift/enhancements/blob/master/CONVENTIONS.md) — coding standards

## SME Review Recommended

- Federation workflow end-to-end behavior (ClusterFederatedTrustDomain lifecycle, cross-cluster communication)
- Upstream authority plugin configuration and cert-manager integration details
- SPIRE server HA behavior and data persistence strategy
- E2E test infrastructure requirements and CI environment setup
