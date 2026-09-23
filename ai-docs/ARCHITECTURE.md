# ZTWIM Architecture

## Repository Layout

```text
api/v1alpha1/                     # CRD types -- DO NOT edit zz_generated.deepcopy.go
  conditions.go                   # Shared condition constants (Ready, Degraded, Upgradeable)
  meta.go                         # ConditionalStatus, GetConditionalStatus helpers
  zero_trust_workload_identity_manager_types.go  # Parent CR + CommonConfig
  spire_server_config_types.go    # SpireServer CR
  spire_agent_config_types.go     # SpireAgent CR
  spiffe_csi_config_types.go      # SpiffeCSIDriver CR
  spire_oidc_discovery_provider_types.go  # SpireOIDCDiscoveryProvider CR
bindata/                          # Static upstream YAML manifests -- DO NOT hand-edit
  spire-server/                   # StatefulSet, RBAC, Service templates
  spire-agent/                    # DaemonSet, RBAC, SCC templates
  spiffe-csi/                     # CSIDriver, DaemonSet templates
  spire-controller-manager/       # Webhook, RBAC templates
  spire-oidc-discovery-provider/  # Deployment, Service templates
  spire-bundle/                   # Bundle Role/RoleBinding templates
bundle/                           # OLM bundle (CSV, CRDs, metadata)
cmd/zero-trust-workload-identity-manager/  # Entrypoint (main.go)
config/                           # Kustomize overlays (CRDs, RBAC, manager, samples)
hack/                             # Helper scripts (update-bindata, etc.)
pkg/client/                       # CustomCtrlClient wrapper + label-filtered cache
  fakes/                          # Counterfeiter-generated fakes for testing
pkg/controller/
  zero-trust-workload-identity-manager/  # ZTWIM controller (status aggregator)
  spire-server/                          # SpireServer controller
  spire-agent/                           # SpireAgent controller
  spiffe-csi-driver/                     # SpiffeCSIDriver controller
  spire-oidc-discovery-provider/         # SpireOIDCDiscoveryProvider controller
  status/                               # Status manager (condition aggregation, health checks)
  utils/                                 # Shared constants, errors, labels, predicates, resource comparison
pkg/operator/assets/              # bindata.go -- DO NOT hand-edit, use `make update-bindata`
pkg/version/                      # Operator and per-component version strings
test/e2e/                         # Ginkgo E2E tests (Ordered, real cluster)
vendor/                           # Tracked vendored dependencies
```

## Key Domain Concepts

ZTWIM is a **meta-operator**: it manages a family of 5 CRDs that together deploy the SPIFFE/SPIRE stack. The primary workflow:

1. User creates `ZeroTrustWorkloadIdentityManager` CR (global config: trust domain, cluster name, bundle ConfigMap)
2. User creates operand CRs (`SpireServer`, `SpireAgent`, `SpiffeCSIDriver`, `SpireOIDCDiscoveryProvider`)
3. Each operand controller reconciles its CR, creating Kubernetes resources from bindata YAML templates
4. ZTWIM controller aggregates status from all operand CRs and syncs Upgradeable condition to OLM

All five ZTWIM-owned CRDs are **cluster-scoped singletons** named `cluster`, enforced by CEL XValidation on each type. The three upstream `spire.spiffe.io` CRDs (ClusterSPIFFEID, ClusterFederatedTrustDomain, ClusterStaticEntry) are NOT singletons.

## Controller Details

All controllers use **controller-runtime** (not library-go). Each follows the same pattern:

| Controller | Package | CR | Workload | Component Label |
|---|---|---|---|---|
| ZTWIM | `pkg/controller/zero-trust-workload-identity-manager/` | `ZeroTrustWorkloadIdentityManager` | None | N/A |
| SpireServer | `pkg/controller/spire-server/` | `SpireServer` | StatefulSet `spire-server` | `control-plane` |
| SpireAgent | `pkg/controller/spire-agent/` | `SpireAgent` | DaemonSet `spire-agent` | `node-agent` |
| SpiffeCSIDriver | `pkg/controller/spiffe-csi-driver/` | `SpiffeCSIDriver` | DaemonSet `spire-spiffe-csi-driver` | `csi` |
| SpireOIDCDP | `pkg/controller/spire-oidc-discovery-provider/` | `SpireOIDCDiscoveryProvider` | Deployment `spire-spiffe-oidc-discovery-provider` | `discovery` |

**Startup sequence** (`cmd/zero-trust-workload-identity-manager/main.go`):
1. Register schemes (core, OpenShift security/route, OLM operator, spire-controller-manager)
2. Create `CacheBuilder` with label-filtered informers (`app.kubernetes.io/managed-by=zero-trust-workload-identity-manager`)
3. Create controller-runtime Manager with unified cache
4. Instantiate all 5 controllers via `New(mgr)` + `SetupWithManager(mgr)`
5. Start manager with signal handler

**Controller names** follow the pattern `zero-trust-workload-identity-manager-<component>-controller` (constants in `pkg/controller/utils/constants.go:6-10`).

## Resource Management

**Apply method**: Get+Create/Update (NOT Server-Side Apply). All controllers:
1. Decode desired state from bindata YAML via `Decode*ObjBytes()` helpers (`pkg/controller/utils/utils.go:69-131`)
2. Set labels, annotations, namespace, image references on the decoded object
3. Attempt `Get` -- if not found, `Create`; if found, check `ResourceNeedsUpdate()` then `Update`
4. `ResourceNeedsUpdate()` (`pkg/controller/utils/resource_comparison.go:23`) does type-aware diff (labels, annotations, type-specific fields)

**Create-only mode**: When `CREATE_ONLY_MODE=true` env var is set, controllers create resources but skip updates (`pkg/controller/utils/utils.go:237-255`). Each controller checks this and sets a `CreateOnlyMode` status condition.

**Image resolution**: All operand images come from `RELATED_IMAGE_*` env vars (`pkg/controller/utils/constants.go:57-63`):

| Env Var | Component |
|---|---|
| `RELATED_IMAGE_SPIRE_SERVER` | SPIRE Server |
| `RELATED_IMAGE_SPIRE_AGENT` | SPIRE Agent |
| `RELATED_IMAGE_SPIFFE_CSI_DRIVER` | SPIFFE CSI Driver |
| `RELATED_IMAGE_SPIRE_OIDC_DISCOVERY_PROVIDER` | OIDC Discovery Provider |
| `RELATED_IMAGE_SPIRE_CONTROLLER_MANAGER` | SPIRE Controller Manager (sidecar) |
| `RELATED_IMAGE_NODE_DRIVER_REGISTRAR` | Node Driver Registrar |
| `RELATED_IMAGE_SPIFFE_CSI_INIT_CONTAINER` | CSI init container (defaults to `ubi9:latest`) |

OLM injects these via the ClusterServiceVersion `relatedImages` section.

## Error Classification

Three error types (`pkg/controller/utils/errors.go:12-17`):

| Type | Requeue | Usage |
|---|---|---|
| `IrrecoverableError` | No | Permission denied, invalid config, forbidden |
| `RetryRequiredError` | Yes | Transient API errors, conflicts |
| `MultipleInstanceError` | No | Singleton violation |

`FromClientError()` auto-classifies: Unauthorized/Forbidden/Invalid/BadRequest/ServiceUnavailable -> Irrecoverable; everything else -> RetryRequired.

## Status Management

`pkg/controller/status/status.go` provides `Manager`:
- Collects conditions during reconciliation via `AddCondition()`
- Auto-aggregates `Ready` condition from all other conditions (unless explicitly set)
- Distinguishes `Progressing` (StatefulSetNotReady, DaemonSetNotReady, DeploymentNotReady) from `Failed`
- `ApplyStatus()` only writes if status actually changed (semantic deep-equal)
- Health check helpers: `CheckStatefulSetHealth()`, `CheckDaemonSetHealth()`, `CheckDeploymentHealth()`

**ZTWIM controller status aggregation** (`pkg/controller/zero-trust-workload-identity-manager/controller.go:382-408`):
- Reads status from all 4 operand CRs
- Classifies each via `classifyOperandState()`: `ready`, `progressing`, or `failed`
- Sets `OperandsAvailable` and `Ready` conditions on the parent CR
- Syncs `Upgradeable` condition to OLM OperatorCondition

## Watch Predicates

| Predicate | Purpose | File |
|---|---|---|
| `ZTWIMSpecChangedPredicate` | Operand controllers react to ZTWIM CR create/delete only | `utils/utils.go:259-272` |
| `GenerationOrOwnerReferenceChangedPredicate` | Standard for operand CR watches | `utils/utils.go:343-346` |
| `ControllerManagedResourcesForComponent(component)` | Filter managed resources by both `managed-by` and `component` labels | `utils/labels.go:82-94` |
| `operandStatusChangedPredicate` | ZTWIM controller only reconciles on operand status changes | `controller.go:547-571` |

## Labeling

All managed resources get standardized Kubernetes labels (`pkg/controller/utils/labels.go:27-43`):

```
app.kubernetes.io/name: <resource-name>
app.kubernetes.io/instance: cluster-zero-trust-workload-identity-manager
app.kubernetes.io/part-of: zero-trust-workload-identity-manager
app.kubernetes.io/component: <control-plane|node-agent|csi|discovery>
app.kubernetes.io/managed-by: zero-trust-workload-identity-manager
app.kubernetes.io/version: <component-version>
```

Component-specific label generators: `SpireServerLabels()`, `SpireAgentLabels()`, etc. (`labels.go:47-65`).

## OpenShift Integration Points

| Integration | How | Code Reference |
|---|---|---|
| OLM OperatorCondition | ZTWIM syncs `Upgradeable` to OperatorCondition CR | `controller.go:575-636` |
| OpenShift Routes | SpireServer (federation) and OIDC use `route.openshift.io/v1` | `spire-server/routes.go`, `spire-oidc/routes.go` |
| SecurityContextConstraints | SpireAgent creates custom SCC; CSI uses privileged SCC RoleBinding | `spire-agent/scc.go`, `spiffe-csi-driver/rbac.go` |
| Service CA | Annotates Services with `service.beta.openshift.io/serving-cert-secret-name` | `utils/constants.go:53-54` |
| Proxy | Validates proxy env vars (HTTP_PROXY, HTTPS_PROXY, NO_PROXY) and CA bundle | `utils/proxy.go` |
| Metrics | Secure metrics endpoint at :8443 with authn/authz via ServiceAccount token | `main.go:140-187` |

## Generated Code Inventory

| File/Dir | Generator | Regenerate Command |
|---|---|---|
| `api/v1alpha1/zz_generated.deepcopy.go` | controller-gen | `make generate` |
| `config/crd/bases/*.yaml` | controller-gen | `make manifests` |
| `pkg/operator/assets/bindata.go` | go-bindata | `make update-bindata` |
| `pkg/client/fakes/` | counterfeiter | `go generate ./pkg/client/...` |
| `bundle/manifests/` | operator-sdk | `make bundle` |

**NEVER hand-edit any of these files.**

## API Behavioral Contracts

- **Singleton enforcement**: All 5 ZTWIM CRDs enforce `.metadata.name == 'cluster'` via CEL XValidation
- **Immutable fields**: `trustDomain`, `clusterName`, `bundleConfigMap` on ZTWIM CR use `self == oldSelf` CEL rules
- **Owner references**: Each operand CR gets `ZeroTrustWorkloadIdentityManager` as owner (garbage collection cascade)
- **Config hash annotations**: StatefulSet/DaemonSet/Deployment pods get config hash annotations (e.g., `ztwim.openshift.io/spire-server-config-hash`) to trigger rolling updates on ConfigMap changes
- **Resource conflict detection**: `CheckResourceConflict()` verifies existing resources have the `managed-by` label before updating, preventing conflicts with pre-existing resources (`utils/resource_ownership.go:21-27`)
- **CommonConfig pattern**: All operand CRs embed `CommonConfig` struct providing `labels`, `resources`, `affinity`, `tolerations`, `nodeSelector` fields with validation

## Design References

**controller-runtime over library-go**: ZTWIM chose controller-runtime because SPIRE lifecycle management requires managing multiple independent components (server, agent, CSI, OIDC) as separate CRDs rather than a single monolithic operator. controller-runtime's multi-controller manager pattern with per-controller watches maps naturally to this architecture. See `harness-evals/harness-docs/decisions/adr-0001-controller-runtime-over-library-go.md`.

**Bindata for operand manifests**: Upstream SPIRE YAML manifests are embedded as bindata rather than templated or generated. This preserves upstream compatibility while allowing runtime customization (namespace, images, labels) during decoding. See `harness-evals/harness-docs/decisions/adr-0002-bindata-for-operand-manifests.md`.

**Unified cache with label filtering**: The `CacheBuilder` (`pkg/client/client.go:224-269`) pre-registers informers with `managed-by` label selectors to reduce API server load. Resources without the label are invisible to the cache, which is why `CheckResourceConflict()` exists for conflict detection on Create.

## Platform Documentation

For generic operator patterns, testing practices, security guidelines, and coding standards, see the [openshift/enhancements](https://github.com/openshift/enhancements) repository:
- `dev-guide/` -- Development conventions
- `guidelines/` -- Enhancement process
- `CONVENTIONS.md` -- Coding standards

## SME Review Recommended

- Federation configuration details and cross-cluster trust domain behavior
- Upstream authority plugin configuration and cert-manager integration
- FIPS compliance status (no explicit FIPS build tags or OpenShift fork references detected in `go.mod`)
- Form factor behavior (no explicit SNO/HCP/MicroShift detection found in code)
