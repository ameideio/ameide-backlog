# 520 TDD — Agent Vertical Progress

Complete all unchecked items in this document for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source

- [x] Keep the Agent v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/agent/v1/echo_agent.proto`.
- [x] Keep `thread_id` in the request contract for persisted identity.
- [ ] If persistence/checkpointing is enabled, require `thread_id` (hard reject if missing) and map it to LangGraph `configurable.thread_id`.
- [ ] If replay/continue is supported, accept `checkpoint_id` in config and document the behavior.
- [ ] Ensure all persisted state fields have explicit reducers (default `REPLACE`).
- [ ] For DAG parallelism, use `Send` fan-out and reducer merges (no implicit “LLM parallelization”).
- [ ] Keep large outputs out-of-band (artifact refs only in state).
- [ ] If persistence/checkpointing is enabled, require `thread_id` (hard reject if missing) and map it to LangGraph `configurable.thread_id`.
- [ ] Document a canonical `thread_id` scheme (recommended: `<tenant>/<capability>/<work_item_id>`; keep a separate `run_id` for concurrency).

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
- [ ] Configuration authority: runtime wiring is GitOps/operator-provisioned only (env/secret/volume), request inputs are request-provisioned only, and the runtime has no fallback/override chains (see `backlog/520-primitives-stack-v2.md`).
- [ ] Interrupt safety: nodes may be re-run; side-effects must be idempotent (dedupe keys derived from `{thread_id, node_id, logical_task_key}`).
- [ ] Streaming: prefer machine-readable progress updates; keep token/message streaming opt-in.
- [ ] Keep agent state “resume essentials” only; store large outputs as out-of-band artifact references (`uri`, `checksum`, `size`, `content_type`).
- [ ] Emit at least one artifact reference per turn (even for Echo v0) to validate the pattern end-to-end.
- [ ] Add replay/conformance tests:
  - [ ] A fresh checkpointer per test and deterministic replay produces the same outputs for the same inputs.
  - [ ] Tool registry wiring exists (tools may be no-ops, but must be registered and validated).

### Operator + GitOps

- [x] Keep the Agent CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_agents.yaml`.
- [x] Keep the Agent operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/agent-operator-deployment.yaml`.
- [x] Keep the v0 Agent workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/agent-echo-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/agent-echo-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/agent-echo-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/agent-echo-v0-smoke.yaml`

### Manual Image Publish

- [x] Publish the runtime image to GHCR and deploy it via a digest-pinned ref in GitOps (local/dev bump PR writes the digest; do not deploy floating `:dev` directly). See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
- [x] Publish the operator image to GHCR and pin by digest in GitOps (operators are dependencies; do not deploy floating `:dev`/`:main` directly). See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
