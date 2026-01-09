# 520 TDD — Process Vertical Progress

Complete all unchecked items in this document for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source

- [x] Keep the Process v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/process/v1/process_v0.proto`.

### SDKs

- [x] Generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator

- [x] Use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] Keep the Process glue template at `packages/ameide_core_proto/buf.gen.process-ping.local.yaml`.
- [x] Write the generated glue to `primitives/process/ping/internal/gen/` and keep it gitignored.

### Runtime

- [x] Keep the Process runtime at `primitives/process/ping/`.
- [x] Expose gRPC on port `50051` and serve gRPC health.
- [x] Implement deterministic v0 behavior in `ameide_core_proto.process.v1.ProcessV0Service/Ping`.
- [ ] Configuration authority: runtime wiring is GitOps/operator-provisioned only (env/secret/volume), request inputs are request-provisioned only, and the runtime has no fallback/override chains (see `backlog/520-primitives-stack-v2.md`).

### Operator + GitOps

- [x] Keep the Process CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_processes.yaml`.
- [x] Keep the Process operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/process-operator-deployment.yaml`.
- [x] Keep the v0 Process workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/process-ping-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/process-ping-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/process-ping-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/process-ping-v0-smoke.yaml`

### Publish

- [x] CI publishes the runtime image to GHCR; GitOps deploys only digest-pinned refs (no floating tags). See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
- [x] CI publishes the operator image to GHCR; GitOps pins the operator image by digest (operators are dependencies). See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
