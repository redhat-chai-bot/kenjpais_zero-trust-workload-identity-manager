# Zero Trust Workload Identity Manager (ZTWIM)

**Component**: Zero Trust Workload Identity Manager (ZTWIM)
**Repository**: `openshift/zero-trust-workload-identity-manager` | **Branch**: `main` | **Go**: 1.25.7

A controller-runtime operator managing upstream SPIFFE/SPIRE components on OpenShift. Deploys SPIRE server, agent, OIDC discovery provider, and SPIFFE CSI driver via static YAML bindata manifests. Upstream images injected through `RELATED_IMAGE_*` env vars.

## Critical Warnings

1. **Never hand-edit generated files** — `zz_generated.deepcopy.go`, `config/crd/bases/*.yaml`, `pkg/operator/assets/bindata.go`. Use `make generate`, `make manifests`, `make update-bindata`.
2. **All five ZTWIM-owned CRDs are cluster-scoped singletons named `cluster`** — enforced by CEL XValidation. The three upstream `spire.spiffe.io` CRDs (ClusterSPIFFEID, ClusterFederatedTrustDomain, ClusterStaticEntry) are NOT singletons.
3. **Imperative Create/Update** — controllers use `CustomCtrlClient.Create` + `Update` (NOT SSA). Resource drift is detected via `ResourceNeedsUpdate` in `pkg/controller/utils/resource_comparison.go`.
4. **Vendor is tracked** — after dependency changes, always `make vendor` and commit the vendor directory.
5. **Run `make verify` before pushing** — runs vet + fmt + golangci-lint.

## Architecture at a Glance

| Controller | Manages | Key Resource |
|---|---|---|
| ZTWIM | Status aggregation, OLM Upgradeable | Watches all operand CRs |
| SpireServer | SPIRE server lifecycle | StatefulSet, ConfigMaps, Routes, webhooks |
| SpireAgent | SPIRE agent lifecycle | DaemonSet, custom SCC, ConfigMap |
| SpiffeCSIDriver | CSI driver lifecycle | DaemonSet, CSIDriver, privileged SCC |
| SpireOIDCDiscoveryProvider | OIDC discovery | Deployment, Route, ClusterSPIFFEID |

## Documentation

```
ai-docs/
├── ARCHITECTURE.md   # Internals, integrations, behavioral contracts
├── DEVELOPMENT.md    # Build, common tasks, mistakes
├── TESTING.md        # Test suites and patterns
└── ENHANCEMENTS.md   # Enhancement/design doc catalog
```

## Key Files

| Need | Start here |
|---|---|
| API types | `api/v1alpha1/` (`*_types.go`) |
| Controller logic | `pkg/controller/<component>/controller.go` |
| Shared utilities | `pkg/controller/utils/` |
| Static manifests | `bindata/` → `pkg/operator/assets/bindata.go` |
| Image env vars | `pkg/controller/utils/constants.go` (lines 57-63) |
| Custom client | `pkg/client/client.go` |

## Build Commands

| Command | Purpose |
|---|---|
| `make build` | Full build (manifests + generate + fmt + vet + binary) |
| `make test` | Unit tests (counterfeiter fakes) |
| `make verify` | vet + fmt + golangci-lint |
| `make manifests generate update-bindata` | Regenerate all generated files |

## External References

- [SPIFFE Specification](https://github.com/spiffe/spiffe) | [SPIRE Docs](https://spiffe.io/docs/latest/spire-about/)
- [OpenShift Docs](https://docs.openshift.com/) | [README](README.md)
- **Platform Documentation**: [openshift/enhancements](https://github.com/openshift/enhancements) (`dev-guide/`, `guidelines/`, `CONVENTIONS.md`)
