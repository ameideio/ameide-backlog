# 520 TDD — Process Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source (MUST)

- [x] MUST keep the Process v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/process/v1/process_v0.proto`.

### SDKs (MUST)

- [x] MUST generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator (MUST)

- [x] MUST use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] MUST keep the Process glue template at `packages/ameide_core_proto/buf.gen.process-ping.local.yaml`.
- [x] MUST write the generated glue to `primitives/process/ping/internal/gen/` and MUST keep it gitignored.

### Runtime (MUST)

- [x] MUST keep the Process runtime at `primitives/process/ping/`.
- [x] MUST expose gRPC on port `50051` and MUST serve gRPC health.
- [x] MUST implement deterministic v0 behavior in `ameide_core_proto.process.v1.ProcessV0Service/Ping`.

### Operator + GitOps (MUST)

- [x] MUST keep the Process CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_processes.yaml`.
- [x] MUST keep the Process operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/process-operator-deployment.yaml`.
- [x] MUST keep the v0 Process workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/process-ping-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/process-ping-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/process-ping-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/process-ping-v0-smoke.yaml`

### Manual Image Publish (MUST)

- [x] MUST publish the runtime image `ghcr.io/ameideio/process-ping:dev`.
- [x] MUST publish the operator image `ghcr.io/ameideio/process-operator:dev`.
