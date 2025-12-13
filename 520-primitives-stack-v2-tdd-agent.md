# 520 TDD — Agent Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Proto Shape (MUST)

- [ ] MUST add or select the Agent shape source protos under `packages/ameide_core_proto/src/ameide_core_proto/`.
- [ ] MUST define a v0 Agent surface that includes an explicit durable identity (`thread_id`) in the request contract.

### SDKs (MUST)

- [ ] MUST have Python SDK stubs under `packages/ameide_sdk_python/` for the Agent surface.
- [ ] MUST have Go SDK stubs under `packages/ameide_sdk_go/gen/` for the Agent surface.
- [ ] MUST have TS SDK stubs under `packages/ameide_sdk_ts/` for the Agent surface.

### Skeleton Generator (MUST)

- [ ] MUST implement the Agent generator plugin under `plugins/`.
- [ ] MUST keep deterministic output and MUST keep golden or unit tests for the generator.
- [ ] MUST provide a Buf generation template under `packages/ameide_core_proto/` named `buf.gen.*.local.yaml`.
- [ ] MUST write generated output only to generated-only roots and MUST gitignore them.

### Runtime (MUST)

- [ ] MUST implement the Agent runtime under `primitives/agent/<name>/` (or the repo’s chosen Agent runtime location).
- [ ] MUST implement a deterministic v0 mode that does not require external LLM providers.
- [ ] MUST persist minimal state keyed by `thread_id` and MUST store large artifacts by reference only.

### Operator (MUST)

- [ ] MUST reconcile the Agent CR into the required Deployments/Services and set `Ready=True` only when the runtime is reachable.

### GitOps (MUST)

- [ ] MUST define an Agent v0 workload component under `gitops/ameide-gitops/environments/_shared/components/apps/primitives/`.
- [ ] MUST define an Agent v0 smoke component under the same hierarchy using an in-cluster Job probe.

### Manual Image Publish (MUST)

- [ ] MUST publish the Agent runtime image manually to GHCR while CI publishing is disabled.

### In-Cluster Probe (MUST)

- [ ] MUST prove “same `thread_id` causes persisted state changes” from inside the cluster (Job probe MUST fail when persistence is missing and MUST pass when it is correct).

