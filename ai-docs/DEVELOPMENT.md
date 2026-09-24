# ZTWIM Development Guide

## Prerequisites

- Go 1.25.7 (`go.mod`: `go 1.25.7`)
- Docker 17.03+
- kubectl v1.11.3+
- Access to OpenShift 4.19+ cluster (for E2E)
- operator-sdk (installed via `make operator-sdk`)

## Build Commands

| Command | Purpose |
|---|---|
| `make build` | Full build: manifests + generate + fmt + vet + binary |
| `make build-operator` | Build the operator binary only |
| `make test` | Unit tests with envtest |
| `make verify` | vet + fmt + golangci-lint — **run before every push** |
| `make lint` | golangci-lint only |
| `make lint-fix` | golangci-lint with auto-fix |
| `make fmt` | `go fmt ./...` |
| `make vet` | `go vet ./...` |
| `make manifests` | Regenerate CRD YAML from kubebuilder markers |
| `make generate` | Regenerate deepcopy functions |
| `make update-bindata` | Regenerate `pkg/operator/assets/bindata.go` from `bindata/` YAML |
| `make vendor` | `go mod vendor` — **always commit vendor after dep changes** |
| `make docker-build IMG=<image>` | Build container image |
| `make docker-push IMG=<image>` | Push container image |
| `make bundle` | Generate OLM bundle |
| `make deploy IMG=<image>` | Deploy operator to cluster |
| `make undeploy` | Remove operator from cluster |

## Common Tasks

### Adding a new managed resource to a controller

1. Create the YAML manifest in `bindata/<component>/`
2. Run `make update-bindata` to regenerate `pkg/operator/assets/bindata.go`
3. Add the asset name constant to `pkg/controller/utils/constants.go`
4. Add a `reconcile<Resource>()` method in the controller package
5. Call the new reconcile method from the main `Reconcile()` function
6. Add the resource type to the `Watches()` chain in `SetupWithManager()`
7. Add necessary RBAC markers (kubebuilder comments) on the ZTWIM controller
8. Run `make manifests generate` to update CRD and RBAC

### Adding a new RELATED_IMAGE env var

1. Add the env var constant to `pkg/controller/utils/constants.go`
2. Add the getter function to `pkg/controller/utils/relatedImages.go`
3. Update the CSV in `bundle/manifests/` with the new relatedImage entry
4. Update the Makefile if a `*_VERSION` variable is needed

### Modifying CRD types

1. Edit the types in `api/v1alpha1/<crd>_types.go`
2. Add kubebuilder validation markers as needed
3. Run `make manifests generate` to regenerate CRDs and deepcopy
4. Run `make update-bindata` if bindata references changed
5. Update `bundle/` with `make bundle`
6. Run `make verify` before pushing

### Updating upstream SPIRE component versions

1. Update version constants in `pkg/version/version.go`
2. Update `RELATED_IMAGE_*` defaults if applicable
3. Update any changed YAML manifests in `bindata/`
4. Run `make update-bindata`
5. Update the CSV and bundle: `make bundle`

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `OPERATOR_NAMESPACE` | Namespace where operator runs | Required, no default |
| `OPERATOR_CONDITION_NAME` | OLM OperatorCondition CR name | Required, no default |
| `CREATE_ONLY_MODE` | When `true`, skip resource updates | `false` |
| `RELATED_IMAGE_SPIRE_SERVER` | SPIRE server image | None (required) |
| `RELATED_IMAGE_SPIRE_AGENT` | SPIRE agent image | None (required) |
| `RELATED_IMAGE_SPIFFE_CSI_DRIVER` | SPIFFE CSI driver image | None (required) |
| `RELATED_IMAGE_SPIRE_OIDC_DISCOVERY_PROVIDER` | OIDC provider image | None (required) |
| `RELATED_IMAGE_SPIRE_CONTROLLER_MANAGER` | Controller manager image | None (required) |
| `RELATED_IMAGE_NODE_DRIVER_REGISTRAR` | Node driver registrar image | None (required) |
| `RELATED_IMAGE_SPIFFE_CSI_INIT_CONTAINER` | CSI init container image | `registry.access.redhat.com/ubi9:latest` |
| `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` | Proxy configuration | None |

## Common Mistakes

1. **Forgetting `make update-bindata`** after editing YAML files in `bindata/`. The controller reads from the compiled Go file, not the YAML directly.
2. **Not running `make vendor`** after changing `go.mod`. The build uses `-mod=vendor`.
3. **Editing `zz_generated.deepcopy.go` manually** — always use `make generate`.
4. **Creating operand CRs with names other than `cluster`** — CEL validation will reject them.
5. **Assuming SSA apply** — this operator uses imperative Create/Update, not server-side apply. Use `ResourceNeedsUpdate()` pattern.
6. **Missing RBAC markers** — kubebuilder RBAC markers on the ZTWIM controller (`pkg/controller/zero-trust-workload-identity-manager/controller.go`) generate ClusterRole rules. Forgetting them causes permission errors at runtime.
7. **Editing the CSV directly** — use `make bundle` which regenerates from kubebuilder markers and kustomize overlays.

## Linting

golangci-lint configuration in `.golangci.yml`:
- **Enabled linters**: errcheck, govet, gofmt, goimports, staticcheck, revive, misspell, dupl, ineffassign, unconvert, unparam, unused, gocyclo, goconst, lll, nakedret, prealloc, ginkgolinter, exportloopref, typecheck, gosimple
- **Exclusions**: `lll` disabled for `api/*`, `dupl` and `lll` disabled for `pkg/*`
- **Timeout**: 5 minutes

## Platform Documentation

For generic operator patterns, testing practices, and coding standards, see [openshift/enhancements](https://github.com/openshift/enhancements) (`dev-guide/`, `guidelines/`, `CONVENTIONS.md`).

## SME Review Recommended

- Cluster deployment workflow for non-standard environments
- Image mirroring and disconnected cluster support details
- Konflux release pipeline configuration (managed in [zero-trust-workload-identity-manager-release](https://github.com/openshift/zero-trust-workload-identity-manager-release))
