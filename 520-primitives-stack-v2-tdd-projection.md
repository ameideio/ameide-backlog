# 520 TDD — Projection Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source (MUST)

- [x] MUST keep the Projection v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/projection/v1/projection_v0.proto`.

### SDKs (MUST)

- [x] MUST generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator (MUST)

- [x] MUST use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] MUST keep the Projection glue template at `packages/ameide_core_proto/buf.gen.projection-foo.local.yaml`.
- [x] MUST write the generated glue to `primitives/projection/foo/internal/gen/` and MUST keep it gitignored.

### Runtime (MUST)

- [x] MUST keep the Projection runtime at `primitives/projection/foo/`.
- [x] MUST implement deterministic v0 behavior in `ameide_core_proto.projection.v1.ProjectionV0QueryService/GetFoo`.
- [x] MUST expose gRPC on port `50051` and MUST serve gRPC health.

### Operator + GitOps (MUST)

- [x] MUST keep the Projection CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_projections.yaml`.
- [x] MUST keep the Projection operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/projection-operator-deployment.yaml`.
- [x] MUST keep the v0 Projection workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/projection-foo-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/projection-foo-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/projection-foo-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/projection-foo-v0-smoke.yaml`

### Manual Image Publish (MUST)

- [x] MUST publish the runtime image `ghcr.io/ameideio/projection-foo:dev`.
- [x] MUST publish the operator image `ghcr.io/ameideio/projection-operator:dev`.
