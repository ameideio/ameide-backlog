# 520 TDD — Domain Vertical (Transformation v0) Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Proto Shape (MUST)

- [x] MUST use shape source protos under `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/`.
- [x] MUST keep the Domain query service in `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation-scrum-query.proto`.
- [x] MUST keep `ScrumQueryService` as the generated Domain surface for v0.

### SDKs (MUST)

- [x] MUST have Go SDK stubs under `packages/ameide_sdk_go/gen/go/ameide_core_proto/transformation/scrum/v1/`.
- [ ] MUST confirm SDK generation is up-to-date with the shape source used by this vertical.

### Skeleton Generator (MUST)

- [x] MUST implement the Domain generator plugin at `plugins/ameide_domain_go/main.go:1`.
- [x] MUST keep a unit test that asserts `source_file_descriptors` is used when present: `plugins/ameide_domain_go/internal/generator/generator_test.go:1`.
- [x] MUST provide a Buf generation template for this vertical: `packages/ameide_core_proto/buf.gen.domain-transformation.local.yaml:1`.
- [x] MUST write generated glue only to `primitives/domain/transformation/internal/gen/`.
- [x] MUST gitignore generated glue at `primitives/**/internal/gen/`.

### Runtime (MUST)

- [x] MUST keep the sample Domain runtime at `primitives/domain/transformation/`.
- [x] MUST register all Domain services via generated glue in `primitives/domain/transformation/cmd/main.go:1`.
- [x] MUST expose gRPC on port `50051` and serve gRPC health.
- [x] MUST build from repo root with `docker build -f primitives/domain/transformation/Dockerfile .`.

### Operator (MUST)

- [x] MUST have the Domain CRD in GitOps at `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_domains.yaml:1`.
- [x] MUST have the Domain operator deployment in GitOps at `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/domain-operator-deployment.yaml:1`.
- [ ] MUST confirm the Domain operator is installed in the target cluster namespace via ArgoCD.

### GitOps (MUST)

- [x] MUST define the Domain v0 CR values at `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml:1`.
- [x] MUST define the Domain v0 workload component at `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0/component.yaml:1`.
- [x] MUST define the Domain v0 smoke component at `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0-smoke/component.yaml:1`.
- [x] MUST define the Domain v0 smoke values at `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0-smoke.yaml:1`.
- [ ] MUST confirm ArgoCD sync applies the Domain CR and creates the expected `Deployment` and `Service`.

### Manual Image Publish (MUST)

- [x] MUST publish `ghcr.io/ameideio/transformation-domain:dev` manually (CI publishing is disabled for this activity).

### In-Cluster Probe (MUST)

- [x] MUST probe via a PostSync Job that calls `grpc.health.v1.Health/Check` (defined in `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0-smoke.yaml:1`).
- [ ] MUST confirm the PostSync Job succeeds in the target cluster.

