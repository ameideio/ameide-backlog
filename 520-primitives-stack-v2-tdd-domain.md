# 520 TDD — Domain Vertical (Transformation v0) Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source (MUST)

- [x] MUST keep the Domain query proto at `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation-scrum-query.proto`.

### SDKs (MUST)

- [x] MUST generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator (MUST)

- [x] MUST use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] MUST keep the Domain glue template at `packages/ameide_core_proto/buf.gen.domain-transformation.local.yaml`.
- [x] MUST write the generated glue to `primitives/domain/transformation/internal/gen/` and MUST keep it gitignored.

### Runtime (MUST)

- [x] MUST keep the Domain runtime at `primitives/domain/transformation/`.
- [x] MUST register services via generated glue in `primitives/domain/transformation/cmd/main.go`.
- [x] MUST expose gRPC on port `50051` and MUST serve gRPC health.

### Operator + GitOps (MUST)

- [x] MUST keep the Domain CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_domains.yaml`.
- [x] MUST keep the Domain operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/domain-operator-deployment.yaml`.
- [x] MUST keep the v0 Domain workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0-smoke.yaml`

### Manual Image Publish (MUST)

- [x] MUST publish the runtime image `ghcr.io/ameideio/transformation-domain:dev`.
