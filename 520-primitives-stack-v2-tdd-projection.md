# 520 TDD — Projection Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Proto Shape (MUST)

- [ ] MUST add or select the Projection shape source protos under `packages/ameide_core_proto/src/ameide_core_proto/`.
- [ ] MUST define a v0 Projection query surface that is sufficient to prove end-to-end behavior.

### SDKs (MUST)

- [ ] MUST have Go SDK stubs under `packages/ameide_sdk_go/gen/` for the Projection surface.
- [ ] MUST have Python SDK stubs under `packages/ameide_sdk_python/` for the Projection surface.
- [ ] MUST have TS SDK stubs under `packages/ameide_sdk_ts/` for the Projection surface.

### Skeleton Generator (MUST)

- [ ] MUST implement the Projection generator plugin under `plugins/`.
- [ ] MUST generate an offset/checkpoint contract and MUST make processing semantics explicit (at-least-once vs exactly-once).
- [ ] MUST provide a Buf generation template under `packages/ameide_core_proto/` named `buf.gen.*.local.yaml`.
- [ ] MUST write generated output only to generated-only roots and MUST gitignore them.

### Runtime (MUST)

- [ ] MUST implement the Projection runtime under `primitives/projection/<name>/` (or the repo’s chosen Projection runtime location).
- [ ] MUST implement idempotent sink writes for at-least-once processing.
- [ ] MUST implement a rebuild/replay path for v0 verification.

### Operator (MUST)

- [ ] MUST reconcile the Projection CR into migrations + runtime and set `Ready=True` only when both are successful.

### GitOps (MUST)

- [ ] MUST define a Projection v0 workload component under `gitops/ameide-gitops/environments/_shared/components/apps/primitives/`.
- [ ] MUST define a Projection v0 smoke component under the same hierarchy using an in-cluster Job probe.

### Manual Image Publish (MUST)

- [ ] MUST publish the Projection runtime image manually to GHCR while CI publishing is disabled.

### In-Cluster Probe (MUST)

- [ ] MUST prove query correctness from inside the cluster (Job probe MUST fail without migrations/seed data and MUST pass when the projection is correct).

