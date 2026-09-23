# ZTWIM Code Review Instructions

## Scope

These instructions apply to reviews of the zero-trust-workload-identity-manager repository. Use them alongside [openshift/enhancements](https://github.com/openshift/enhancements) `dev-guide/` and `CONVENTIONS.md` for platform-wide conventions.

## Do Not Report

Issues in generated or vendored files -- these are maintained by tooling:
- `vendor/**`
- `go.sum`
- `api/v1alpha1/zz_generated.deepcopy.go`
- `config/crd/bases/**`
- `pkg/operator/assets/bindata.go`
- `pkg/client/fakes/**`
- `bundle/manifests/**`

## Critical Checks (Severity: Error)

1. **Generated code consistency**: If `api/v1alpha1/*_types.go` changed, verify `make manifests generate` was run (check `zz_generated.deepcopy.go` and `config/crd/bases/` are updated)
2. **Bindata sync**: If files in `bindata/` changed, verify `make update-bindata` was run (check `pkg/operator/assets/bindata.go` is updated)
3. **Resource comparison coverage**: If new fields are added to workload specs (StatefulSet, DaemonSet, Deployment), verify `ResourceNeedsUpdate()` in `pkg/controller/utils/resource_comparison.go` handles them
4. **Singleton enforcement**: New CRDs must include CEL XValidation `self.metadata.name == 'cluster'`
5. **Owner reference**: New operand controllers must set ZTWIM as owner via `controllerutil.SetControllerReference()`
6. **Create-only mode**: Controllers must check `utils.IsInCreateOnlyMode()` before updating existing resources
7. **Vendor consistency**: If `go.mod` changed, verify `vendor/` is updated via `make vendor`

## Important Checks (Severity: Warning)

1. **Label consistency**: New managed resources should use `StandardizedLabels()` from `pkg/controller/utils/labels.go`
2. **Error classification**: API errors should use `utils.FromClientError()` for proper retry/irrecoverable classification
3. **Status conditions**: Controllers should use `status.Manager` and set meaningful conditions with standard reasons from `api/v1alpha1/conditions.go`
4. **Config hash annotations**: ConfigMap changes should trigger workload rolling updates via hash annotations
5. **Constant naming**: New constants should follow existing patterns in `pkg/controller/utils/constants.go`

## Path-Specific Rules

- `api/v1alpha1/*_types.go`: Verify kubebuilder markers, CEL validation rules, field documentation
- `pkg/controller/*/controller.go`: Verify watch predicates, status condition handling, create-only mode support
- `pkg/controller/utils/`: Shared code -- changes here affect all controllers; verify test coverage
- `bindata/`: Upstream YAML templates -- verify they match upstream SPIRE versions in `pkg/version/version.go`
- `test/e2e/`: Verify proper use of `Eventually`, `DeferCleanup`, and timeout constants

## Platform References

- [openshift/enhancements dev-guide/](https://github.com/openshift/enhancements/tree/master/dev-guide) -- Development conventions
- [openshift/enhancements CONVENTIONS.md](https://github.com/openshift/enhancements/blob/master/CONVENTIONS.md) -- Coding standards
