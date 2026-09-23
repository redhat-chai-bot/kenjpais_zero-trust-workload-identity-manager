# ZTWIM Development Guide

## Prerequisites

- Go 1.25.7+ (`go.mod`)
- Default branch: `main`
- Operator namespace: `zero-trust-workload-identity-manager` (env `OPERATOR_NAMESPACE`)

## Build Commands

| Command | Purpose |
|---|---|
| `make build` | Full build pipeline: manifests + generate + fmt + vet + binary |
| `make test` | Unit tests with envtest (controller-runtime test framework) |
| `make verify` | vet + fmt + golangci-lint (CI-enforced) |
| `make lint` | golangci-lint only |
| `make fmt` | gofmt formatting |
| `make vet` | go vet |
| `make manifests` | Regenerate CRD YAMLs from types (`config/crd/bases/`) |
| `make generate` | Regenerate deepcopy functions (`zz_generated.deepcopy.go`) |
| `make update-bindata` | Regenerate `pkg/operator/assets/bindata.go` from `bindata/` |
| `make vendor` | Update vendored dependencies |
| `make docker-build` | Build container image |
| `make bundle` | Generate OLM bundle |

## Common Tasks

### Add a new field to a CRD

1. Edit the type file in `api/v1alpha1/` (e.g., `spire_server_config_types.go`)
2. Add kubebuilder validation markers
3. Run `make manifests generate`
4. Update the controller to read and use the new field
5. Update tests
6. Run `make verify` before pushing

### Add a new bindata manifest

1. Add the YAML file to the appropriate `bindata/<component>/` directory
2. Run `make update-bindata` to regenerate `pkg/operator/assets/bindata.go`
3. Add a constant for the asset path in `pkg/controller/utils/constants.go`
4. Reference the new asset in the controller's reconcile functions
5. Run `make verify`

### Add a new RELATED_IMAGE

1. Add the env var constant to `pkg/controller/utils/constants.go`
2. Add the getter function to `pkg/controller/utils/relatedImages.go`
3. Add the env var to the Makefile (look for existing `RELATED_IMAGE_*` patterns)
4. Add to the CSV relatedImages in `bundle/manifests/`
5. Use the getter in the controller when building container specs

### Add a new controller

1. Create package under `pkg/controller/<name>/`
2. Define the reconciler struct with `customClient.CustomCtrlClient`, `logr.Logger`, `record.EventRecorder`, `runtime.Scheme`
3. Implement `New(mgr) (*Reconciler, error)` and `Reconcile(ctx, req) (Result, error)`
4. Use `status.NewManager()` for condition management (defer `ApplyStatus`)
5. Register controller in `cmd/zero-trust-workload-identity-manager/main.go`
6. Add the controller name constant to `pkg/controller/utils/constants.go`
7. Use `utils.ControllerManagedResourcesForComponent()` for managed resource watches
8. Watch the parent ZTWIM CR with `utils.ZTWIMSpecChangedPredicate`

### Update vendored dependencies

1. Update `go.mod` / `go.sum`
2. Run `make vendor`
3. Commit the vendor directory changes

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `OPERATOR_NAMESPACE` | Namespace for operator resources | Required (empty = fail) |
| `OPERATOR_CONDITION_NAME` | OLM OperatorCondition CR name | Required (empty = fail) |
| `CREATE_ONLY_MODE` | Skip updates to existing resources | `false` |
| `RELATED_IMAGE_*` | OLM-injected operand container images | None (required) |
| `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` | Proxy configuration | None |

## Common Mistakes

1. **Editing generated files directly** -- Always use `make generate`, `make manifests`, or `make update-bindata`
2. **Forgetting `make vendor`** after modifying `go.mod` -- CI will fail
3. **Not running `make verify`** before pushing -- CI enforces vet + fmt + golangci-lint
4. **Using SSA or Patch for resource updates** -- ZTWIM uses Get+Create/Update with `ResourceNeedsUpdate()`. Follow existing controller patterns
5. **Forgetting config hash annotations** -- ConfigMap changes must update the hash annotation on the workload pod template to trigger rolling updates
6. **Missing owner reference** -- Each operand CR must be owned by ZTWIM CR via `controllerutil.SetControllerReference()`
7. **Ignoring create-only mode** -- Controllers must check `utils.IsInCreateOnlyMode()` and skip updates accordingly
8. **Not updating `ResourceNeedsUpdate()`** -- When adding new fields to workload specs, add corresponding comparison logic to `resource_comparison.go`

## Platform Practices

For generic development conventions, coding standards, and enhancement guidelines, see [openshift/enhancements](https://github.com/openshift/enhancements):
- `dev-guide/` for development conventions
- `CONVENTIONS.md` for coding standards

## SME Review Recommended

- CI-specific Konflux/Prow configuration details
- OLM bundle release workflow and channel management
- Metrics and debugging operational procedures
