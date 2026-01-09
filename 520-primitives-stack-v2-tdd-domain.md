# 520 TDD — Domain Vertical (Example Instance: transformation v0)

Note: this is an example instance checklist for one concrete Domain vertical in this repo. The canonical, domain-agnostic TDD checklist is `backlog/520-primitives-stack-v2-tdd.md`, and the consolidated spec is `backlog/520-primitives-stack-v2.md`.

Complete all unchecked items in this document for this vertical primitive instance to be considered complete.

## Full-Stack Checklist

### Shape Source

- [x] Example instance (`transformation`): keep the Domain query proto at `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_query.proto`.

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
- [ ] Configuration authority: runtime wiring is GitOps/operator-provisioned only (env/secret/volume), request inputs are request-provisioned only, and the runtime has no fallback/override chains (see `backlog/520-primitives-stack-v2.md`).

### Operator + GitOps

- [x] Keep the Domain CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_domains.yaml`.
- [x] Keep the Domain operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/domain-operator-deployment.yaml`.
- [x] Example instance (`transformation`): keep the v0 workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0-smoke.yaml`

### Publish

- [x] CI publishes the runtime image to GHCR; GitOps deploys only digest-pinned refs (no floating tags). See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
