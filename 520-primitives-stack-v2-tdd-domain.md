# 520 TDD — Domain Vertical (Transformation v0) Progress

Complete all unchecked items in this document for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source

- [x] Keep the Domain query proto at `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation-scrum-query.proto`.

### SDKs

- [x] Generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator

- [x] Use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] Keep the Domain glue template at `packages/ameide_core_proto/buf.gen.domain-transformation.local.yaml`.
- [x] Write the generated glue to `primitives/domain/transformation/internal/gen/` and keep it gitignored.

### Runtime

- [x] Keep the Domain runtime at `primitives/domain/transformation/`.
- [x] Register services via generated glue in `primitives/domain/transformation/cmd/main.go`.
- [x] Expose gRPC on port `50051` and serve gRPC health.

### Operator + GitOps

- [x] Keep the Domain CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_domains.yaml`.
- [x] Keep the Domain operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/domain-operator-deployment.yaml`.
- [x] Keep the v0 Domain workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0-smoke.yaml`

### Manual Image Publish

- [x] Publish the runtime image `ghcr.io/ameideio/transformation-domain:dev`.
