# 520 TDD — Process Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Proto Shape (MUST)

- [ ] MUST add or select the Process shape source protos under `packages/ameide_core_proto/src/ameide_core_proto/`.
- [ ] MUST define a v0 Process service/event surface that is sufficient to prove end-to-end execution (ingress → Temporal workflow).

### SDKs (MUST)

- [ ] MUST have Go SDK stubs for the Process surface under `packages/ameide_sdk_go/gen/go/`.
- [ ] MUST have Python SDK stubs for the Process surface under `packages/ameide_sdk_python/`.
- [ ] MUST have TS SDK stubs for the Process surface under `packages/ameide_sdk_ts/`.

### Skeleton Generator (MUST)

- [ ] MUST implement the Process generator plugin under `plugins/`.
- [ ] MUST keep golden or unit tests that assert deterministic output.
- [ ] MUST provide a Buf generation template under `packages/ameide_core_proto/` named `buf.gen.*.local.yaml`.
- [ ] MUST write generated output only to generated-only roots and MUST gitignore them.

### Runtime (MUST)

- [ ] MUST implement the Process runtime under `primitives/process/<name>/` (or the repo’s chosen Process runtime location).
- [ ] MUST enforce Temporal determinism in generated and human-owned workflow code.
- [ ] MUST expose an ingress surface that can start or signal a workflow for the v0 scenario.

### Operator (MUST)

- [ ] MUST reconcile the Process CR into the required Deployments/Services and set `Ready=True` only when the runtime is reachable and configured.
- [ ] MUST keep reconciliation idempotent and bounded (no long-running work in reconcile).

### GitOps (MUST)

- [ ] MUST define a Process v0 workload component under `gitops/ameide-gitops/environments/_shared/components/apps/primitives/`.
- [ ] MUST define a Process v0 smoke component under the same hierarchy using an in-cluster Job probe.

### Manual Image Publish (MUST)

- [ ] MUST publish the Process runtime image(s) manually to GHCR while CI publishing is disabled.

### In-Cluster Probe (MUST)

- [ ] MUST prove the v0 behavior from inside the cluster (Job probe MUST fail when the Process is not running and MUST pass when it is).

