# ZTWIM Testing Guide

## Test Suites

| Suite | Location | Framework | Command |
|---|---|---|---|
| Unit tests | `pkg/controller/*/` (per-controller) | Ginkgo/Gomega + envtest | `make test` |
| E2E tests | `test/e2e/` | Ginkgo/Gomega (real cluster) | `make test-e2e` (CI) |

## Unit Tests

Unit tests use controller-runtime's **envtest** framework with a fake Kubernetes API server. Each controller package has `*_test.go` files testing individual reconciliation steps.

### Test Client: FakeCustomCtrlClient

Tests use counterfeiter-generated fakes of `CustomCtrlClient` (`pkg/client/fakes/`):

```go
// Regenerate fakes after changing the interface:
go generate ./pkg/client/...

// The fake is at pkg/client/fakes/fake_custom_ctrl_client.go
```

The `CustomCtrlClient` interface (`pkg/client/client.go:94-107`) wraps controller-runtime's client with retry semantics (`UpdateWithRetry`, `StatusUpdateWithRetry`), `Exists()`, and `CreateOrUpdateObject()`.

### Test Patterns

**Controller reconciliation tests** (`pkg/controller/spire-server/controller_test.go`, etc.):
- Create a fake client with pre-seeded objects
- Call `Reconcile(ctx, req)` directly
- Assert status conditions were set correctly
- Assert expected resources were created/updated

**Resource comparison tests** (`pkg/controller/utils/resource_comparison_test.go`):
- Test `ResourceNeedsUpdate()` with various resource types
- Verify label/annotation matching semantics
- Test type-specific field comparisons

**Error classification tests** (`pkg/controller/utils/errors_test.go`):
- Test `FromClientError()` classification
- Test `IsIrrecoverableError()` / `IsRetryRequiredError()` / `IsMultipleInstanceError()`

**Status manager tests** (`pkg/controller/status/status_test.go`):
- Test condition aggregation
- Test Ready condition auto-derivation
- Test progressing vs failed distinction

### Running Unit Tests

```bash
make test                    # All unit tests
go test ./pkg/controller/... # Controller tests only
go test ./pkg/controller/spire-server/... -run TestReconcile  # Specific test
```

## E2E Tests

E2E tests (`test/e2e/e2e_test.go`) run against a real OpenShift cluster with the operator installed via OLM. They are **Ordered** Ginkgo suites.

### Test Structure

Tests follow the installation → configuration → attestation → cleanup flow:

1. **Installation**: Create ZTWIM CR, then each operand CR, wait for all conditions True
2. **OperatorCondition**: Verify Upgradeable condition syncs correctly on pod failures/recovery
3. **Attestation**: Create test workloads with ClusterSPIFFEID, verify SVID certificates
4. **Configuration**: Test CR field updates (resources, nodeSelector, tolerations, affinity, labels, log level)
5. **Cleanup**: Delete operand CRs, verify resource cleanup

### E2E Test Utilities

`test/e2e/utils/` provides helpers:
- `WaitFor*` functions (StatefulSet, DaemonSet, Deployment, conditions)
- `SetupAttestationTest()` -- Creates namespace, ServiceAccount, ClusterSPIFFEID, and test pod
- `ReadAndParseSVIDCertificate()` -- Reads and parses X.509 SVID from test pod
- `UpdateCRWithRetry()` -- Retry-on-conflict CR updates
- `ExecInPod()` -- Execute commands in running pods

### E2E Constants

Key test constants (`test/e2e/utils/constants.go`):
- `OperatorNamespace`: `zero-trust-workload-identity-manager`
- `ShortTimeout`: 2 minutes, `DefaultTimeout`: 5 minutes
- `DefaultX509SVIDTTL`: 1 hour

## Adding a New Test

### Unit Test for a Controller Method

1. Add test in the controller's `*_test.go` file
2. Set up fake client with required objects
3. Call the method being tested
4. Assert status conditions and resource state
5. Follow existing patterns in the same file

### E2E Test for a New Feature

1. Add test in `test/e2e/e2e_test.go` within the appropriate `Context` block
2. Use `DeferCleanup()` for teardown
3. Use `Eventually().WithTimeout().WithPolling()` for async assertions
4. Use `utils.UpdateCRWithRetry()` for CR modifications
5. Print debug info with `fmt.Fprintf(GinkgoWriter, ...)`

## Platform Testing Practices

For generic testing patterns, test pyramid guidance, and E2E framework conventions, see [openshift/enhancements](https://github.com/openshift/enhancements) `dev-guide/`.

## SME Review Recommended

- E2E test execution environment and CI pipeline configuration
- Coverage requirements and test coverage thresholds
