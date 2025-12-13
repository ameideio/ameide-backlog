# 520 TDD — Agent Vertical Progress

Complete all unchecked items in this document for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source

- [x] Keep the Agent v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/agent/v1/echo_agent.proto`.
- [x] Keep `thread_id` in the request contract for persisted identity.

### SDKs

- [x] Generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator

- [x] Use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] Keep the Agent glue template at `packages/ameide_core_proto/buf.gen.agent-echo.local.yaml`.
- [x] Write the generated glue to `primitives/agent/echo/internal/gen/` and keep it gitignored.

### Runtime

- [x] Keep the Agent runtime at `primitives/agent/echo/`.
- [x] Implement deterministic v0 behavior with no external LLM dependencies.
- [x] Persist minimal state keyed by `thread_id` (turn counter).

### Operator + GitOps

- [x] Keep the Agent CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_agents.yaml`.
- [x] Keep the Agent operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/agent-operator-deployment.yaml`.
- [x] Keep the v0 Agent workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/agent-echo-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/agent-echo-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/agent-echo-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/agent-echo-v0-smoke.yaml`

### Manual Image Publish

- [x] Publish the runtime image `ghcr.io/ameideio/agent-echo:dev`.
- [x] Publish the operator image `ghcr.io/ameideio/agent-operator:dev`.
