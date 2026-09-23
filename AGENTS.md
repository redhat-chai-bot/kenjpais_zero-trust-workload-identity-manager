# Zero Trust Workload Identity Manager (ZTWIM) - Agent Guide

**Component**: Zero Trust Workload Identity Manager (ZTWIM)
**Repository**: openshift/zero-trust-workload-identity-manager

> **Read ai-docs/ first**: Start with [ai-docs/ARCHITECTURE.md](ai-docs/ARCHITECTURE.md) for internals, then [ai-docs/DEVELOPMENT.md](ai-docs/DEVELOPMENT.md) for workflow, then [ai-docs/TESTING.md](ai-docs/TESTING.md) for test patterns.

## What is ZTWIM?

A controller-runtime operator that deploys upstream SPIFFE/SPIRE components on OpenShift so workloads get dynamic, rotatable X.509/JWT identities. It manages upstream resources via static YAML manifests (bindata), deploying upstream container images via `RELATED_IMAGE_*` env vars.

## Critical Warnings

1. **NEVER hand-edit generated files** -- `zz_generated.deepcopy.go`, `config/crd/bases/*.yaml`, `pkg/operator/assets/bindata.go`. Use `make generate`, `make manifests`, `make update-bindata`.
2. **All five CRDs are cluster-scoped singletons named `cluster`** -- enforced by CEL XValidation. Upstream `spire.spiffe.io` CRDs are NOT singletons.
3. **Vendor is tracked** -- after dependency changes, always `make vendor` and commit the vendor dir. Build uses `-mod=vendor`.
4. **Resource apply uses Get+Create/Update, NOT SSA** -- each controller uses `ResourceNeedsUpdate()` for diff detection (`pkg/controller/utils/resource_comparison.go`).
5. **Run `make verify` before pushing** -- CI enforces vet + fmt + golangci-lint.

## Controllers

| Controller | CR | Workload | Key Resources |
|---|---|---|---|
| ZTWIM | `ZeroTrustWorkloadIdentityManager` | None (aggregator) | Watches all operand CRs, syncs Upgradeable to OLM |
| SpireServer | `SpireServer` | StatefulSet | ConfigMaps, Routes, webhooks, RBAC, federation |
| SpireAgent | `SpireAgent` | DaemonSet | Custom SCC, ConfigMap, ClusterRole |
| SpiffeCSIDriver | `SpiffeCSIDriver` | DaemonSet | CSIDriver, privileged SCC RoleBinding |
| SpireOIDCDP | `SpireOIDCDiscoveryProvider` | Deployment | Route, ClusterSPIFFEID, ConfigMap |

## Documentation

| Need | Start here |
|---|---|
| Architecture, internals, contracts | [ai-docs/ARCHITECTURE.md](ai-docs/ARCHITECTURE.md) |
| Build, workflow, common tasks | [ai-docs/DEVELOPMENT.md](ai-docs/DEVELOPMENT.md) |
| Test patterns (unit + E2E) | [ai-docs/TESTING.md](ai-docs/TESTING.md) |
| Design docs, enhancements | [ai-docs/ENHANCEMENTS.md](ai-docs/ENHANCEMENTS.md) |
| CRD type definitions | [api/v1alpha1/](api/v1alpha1/) |
| Shared controller utilities | [pkg/controller/utils/](pkg/controller/utils/) |
| Platform patterns | [openshift/enhancements](https://github.com/openshift/enhancements) (`dev-guide/`, `CONVENTIONS.md`) |

## Key Build Commands

| Command | Purpose |
|---|---|
| `make build` | Full build (manifests + generate + fmt + vet + binary) |
| `make test` | Unit tests with envtest |
| `make verify` | vet + fmt + golangci-lint |
| `make manifests generate update-bindata` | Regenerate all generated files |

## Upstream

| Project | Integration |
|---|---|
| [spire](https://github.com/spiffe/spire) | Server, agent, OIDC images |
| [spire-controller-manager](https://github.com/spiffe/spire-controller-manager) | API types (Go module); sidecar with server |
| [spiffe-csi](https://github.com/spiffe/spiffe-csi) | CSI driver DaemonSet image |

**Platform Documentation**: [openshift/enhancements](https://github.com/openshift/enhancements) -- `dev-guide/`, `guidelines/`, `CONVENTIONS.md`
