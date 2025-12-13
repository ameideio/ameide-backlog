# 520 TDD — Agent Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source (MUST)

- [x] MUST keep the Agent v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/agent/v1/echo_agent.proto`.
- [x] MUST keep `thread_id` in the request contract for persisted identity.

### SDKs (MUST)

- [x] MUST generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator (MUST)

- [x] MUST use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] MUST keep the Agent glue template at `packages/ameide_core_proto/buf.gen.agent-echo.local.yaml`.
- [x] MUST write the generated glue to `primitives/agent/echo/internal/gen/` and MUST keep it gitignored.

### Runtime (MUST)

- [x] MUST keep the Agent runtime at `primitives/agent/echo/`.
- [x] MUST implement deterministic v0 behavior with no external LLM dependencies.
- [x] MUST persist minimal state keyed by `thread_id` (turn counter).

### Operator + GitOps (MUST)

- [x] MUST keep the Agent CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_agents.yaml`.
- [x] MUST keep the Agent operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/agent-operator-deployment.yaml`.
- [x] MUST keep the v0 Agent workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/agent-echo-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/agent-echo-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/agent-echo-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/agent-echo-v0-smoke.yaml`

### Manual Image Publish (MUST)

- [x] MUST publish the runtime image `ghcr.io/ameideio/agent-echo:dev`.
- [x] MUST publish the operator image `ghcr.io/ameideio/agent-operator:dev`.
