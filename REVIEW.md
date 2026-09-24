# ZTWIM Code Review Instructions

## Review Priorities

1. **Singleton enforcement** — CRDs must have `.metadata.name == 'cluster'` (CEL XValidation). Never allow non-singleton operand CRs.
2. **Resource apply method** — This operator uses imperative Create/Update (NOT SSA). Changes must follow the `ResourceNeedsUpdate()` pattern in `pkg/controller/utils/resource_comparison.go`.
3. **Generated code** — Never hand-edit `zz_generated.deepcopy.go`, `config/crd/bases/*.yaml`, `pkg/operator/assets/bindata.go`. Verify regeneration via `make manifests generate update-bindata`.
4. **Vendor directory** — Must be committed after dependency changes (`make vendor`).
5. **RBAC markers** — kubebuilder RBAC comments on the ZTWIM controller generate ClusterRole rules. Missing markers cause runtime permission errors.

## Do Not Report

These are CI-enforced or generated — do not flag in reviews:

- `**/zz_generated*.go` — generated deepcopy
- `**/bindata.go` — generated go-bindata
- `config/crd/bases/**` — generated CRD YAML
- `bundle/manifests/**` — generated OLM bundle
- `vendor/**` — vendored dependencies
- `pkg/client/fakes/**` — counterfeiter-generated test fakes
- Formatting issues — `make fmt` and `make verify` enforce gofmt/goimports

## Path-Specific Rules

### `api/v1alpha1/` — CRD Type Definitions
- Verify kubebuilder validation markers match intended constraints
- Check CEL XValidation rules for singleton and immutability enforcement
- Ensure new fields have proper `+kubebuilder:validation:` annotations
- Watch for breaking changes to existing fields (especially immutable ones)

### `pkg/controller/*/` — Controller Logic
- Each controller must use `CustomCtrlClient` (not raw `client.Client`)
- New resources must be added to the `Watches()` chain in `SetupWithManager()`
- Resource comparison must go through `ResourceNeedsUpdate()` — not direct equality
- Verify create-only mode is respected (`handleCreateOnlyMode` / `createOnlyMode` parameter)
- Top-level conditions (`Ready`, `Degraded`, `Upgradeable`) use constants from `api/v1alpha1/conditions.go`; per-controller conditions are defined locally in each controller's `controller.go`

### `pkg/controller/utils/` — Shared Utilities
- Changes here affect ALL controllers — review impact broadly
- Error classification (`errors.go`) affects requeue behavior
- Label changes (`labels.go`) affect cache filtering and resource matching
- Resource comparison changes (`resource_comparison.go`) affect drift detection

### `bindata/` — Static Manifests
- Changes require `make update-bindata` to take effect
- Verify RBAC resources match kubebuilder markers
- Check that resource names match constants in `pkg/controller/utils/constants.go`

### `cmd/` — Entrypoint
- Scheme registration order matters — OpenShift API types before upstream
- Flag changes affect deployment manifests

## Platform Conventions

- Follow [OpenShift development conventions](https://github.com/openshift/enhancements/tree/master/dev-guide)
- Follow [OpenShift coding standards](https://github.com/openshift/enhancements/blob/master/CONVENTIONS.md)
- Status conditions should follow [API conventions](https://github.com/openshift/enhancements/blob/master/dev-guide/api-conventions.md)

## Key Behavioral Contracts to Verify

- Owner references: operand CRs must be owned by ZTWIM CR (cascade delete)
- Config hash annotations trigger StatefulSet/DaemonSet rolling updates
- Label predicates filter managed resources by component — cross-controller interference is a bug
- ZTWIM controller aggregates status only — it must NOT create operand CRs
