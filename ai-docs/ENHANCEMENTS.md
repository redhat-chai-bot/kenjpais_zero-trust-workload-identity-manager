# ZTWIM Enhancements & Design Documents

## Repository Design Documents

| Document | Status | Description |
|---|---|---|
| [ADR-0001: controller-runtime over library-go](../harness-evals/harness-docs/decisions/adr-0001-controller-runtime-over-library-go.md) | Implemented | Chose controller-runtime for multi-CRD operator pattern |
| [ADR-0002: Bindata for operand manifests](../harness-evals/harness-docs/decisions/adr-0002-bindata-for-operand-manifests.md) | Implemented | Embedded upstream YAML manifests via go-bindata |

## Upstream References

| Project | Link | Relevance |
|---|---|---|
| SPIFFE Specification | [github.com/spiffe/spiffe](https://github.com/spiffe/spiffe) | Identity framework specification |
| SPIRE Documentation | [spiffe.io/docs](https://spiffe.io/docs/latest/spire-about/) | Upstream SPIRE server/agent docs |
| SPIRE Controller Manager | [github.com/spiffe/spire-controller-manager](https://github.com/spiffe/spire-controller-manager) | CRD types (ClusterSPIFFEID, etc.) |
| SPIFFE CSI | [github.com/spiffe/spiffe-csi](https://github.com/spiffe/spiffe-csi) | CSI driver for SVID injection |

## Related Documentation

| Document | Location | Purpose |
|---|---|---|
| Domain types reference | [harness-evals/harness-docs/domain/](../harness-evals/harness-docs/domain/) | Per-CRD behavioral documentation |
| Architecture reference | [harness-evals/harness-docs/architecture/](../harness-evals/harness-docs/architecture/) | Component overview and patterns |
| Development reference | [harness-evals/harness-docs/ZTWIM_DEVELOPMENT.md](../harness-evals/harness-docs/ZTWIM_DEVELOPMENT.md) | Build and workflow details |
| Testing reference | [harness-evals/harness-docs/ZTWIM_TESTING.md](../harness-evals/harness-docs/ZTWIM_TESTING.md) | Test patterns and utilities |
| Ecosystem references | [harness-evals/harness-docs/references/](../harness-evals/harness-docs/references/) | Platform and upstream links |
