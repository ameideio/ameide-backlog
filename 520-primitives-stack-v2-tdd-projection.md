# 520 TDD — Projection Vertical Progress

Complete all unchecked items in this document for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source

- [x] Keep the Projection v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/projection/v1/projection_v0.proto`.

### SDKs

- [x] Generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator

- [x] Use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] Keep the Projection glue template at `packages/ameide_core_proto/buf.gen.projection-foo.local.yaml`.
- [x] Write the generated glue to `primitives/projection/foo/internal/gen/` and keep it gitignored.

### Runtime

- [x] Keep the Projection runtime at `primitives/projection/foo/`.
- [x] Implement deterministic v0 behavior in `ameide_core_proto.projection.v1.ProjectionV0QueryService/GetFoo`.
- [x] Expose gRPC on port `50051` and serve gRPC health.
- [ ] Configuration authority: runtime wiring is GitOps/operator-provisioned only (env/secret/volume), request inputs are request-provisioned only, and the runtime has no fallback/override chains (see `backlog/520-primitives-stack-v2.md`).

### Operator + GitOps

- [x] Keep the Projection CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_projections.yaml`.
- [x] Keep the Projection operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/projection-operator-deployment.yaml`.
- [x] Keep the v0 Projection workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/projection-foo-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/projection-foo-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/projection-foo-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/projection-foo-v0-smoke.yaml`

### Publish

- [x] CI publishes the runtime image to GHCR; GitOps deploys only digest-pinned refs (no floating tags). See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
- [x] Publish the operator image to GHCR and pin by digest in GitOps (operators are dependencies; do not deploy floating tags directly). See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
