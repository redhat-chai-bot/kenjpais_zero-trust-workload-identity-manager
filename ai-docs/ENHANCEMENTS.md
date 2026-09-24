# ZTWIM Enhancement Proposals & Design References

## Local Design Documents

| Document | Location | Status |
|---|---|---|
| ADR-0001: Controller-runtime over library-go | `harness-evals/harness-docs/decisions/adr-0001-controller-runtime-over-library-go.md` | Implemented |
| ADR-0002: Bindata for operand manifests | `harness-evals/harness-docs/decisions/adr-0002-bindata-for-operand-manifests.md` | Implemented |
| ADR Template | `harness-evals/harness-docs/decisions/adr-template.md` | Template |

## Upstream References

| Project | Repository | Relevance |
|---|---|---|
| SPIFFE Specification | [github.com/spiffe/spiffe](https://github.com/spiffe/spiffe) | Identity standard implemented by SPIRE |
| SPIRE | [github.com/spiffe/spire](https://github.com/spiffe/spire) | Core upstream project (server, agent) |
| SPIRE Controller Manager | [github.com/spiffe/spire-controller-manager](https://github.com/spiffe/spire-controller-manager) | API types used as Go module dependency; runs as sidecar |
| SPIFFE CSI Driver | [github.com/spiffe/spiffe-csi](https://github.com/spiffe/spiffe-csi) | CSI driver for SVID delivery |

## Related OpenShift Enhancement Areas

- SPIFFE/SPIRE integration for workload identity
- Zero Trust security model for OpenShift

## Release Repository

The operator release process is managed via [openshift/zero-trust-workload-identity-manager-release](https://github.com/openshift/zero-trust-workload-identity-manager-release) — Konflux-based, dual-branch model (`release-*` for images, `main` for FBC catalogs).
