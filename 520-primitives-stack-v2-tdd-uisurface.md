# 520 TDD — UISurface Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Proto Shape (MUST)

- [ ] MUST add or select the UISurface shape source protos under `packages/ameide_core_proto/src/ameide_core_proto/`.
- [ ] MUST define a v0 UISurface surface that is sufficient to prove routing end-to-end.

### SDKs (MUST)

- [ ] MUST have TS SDK stubs under `packages/ameide_sdk_ts/` if the UISurface uses generated clients.

### Skeleton Generator (MUST)

- [ ] MUST implement the UISurface generator plugin under `plugins/`.
- [ ] MUST provide a Buf generation template under `packages/ameide_core_proto/` named `buf.gen.*.local.yaml`.
- [ ] MUST write generated output only to generated-only roots and MUST gitignore them.

### Runtime (MUST)

- [ ] MUST implement the UISurface runtime under `primitives/uisurface/<name>/` (or the repo’s chosen UISurface runtime location).
- [ ] MUST serve a v0 “marker response” that is asserted by the in-cluster probe.

### Operator (MUST)

- [ ] MUST reconcile the UISurface CR into `Deployment` + `Service` and `HTTPRoute` when Gateway API is available.
- [ ] MUST set `Ready=True` only when routing and workload are ready.

### GitOps (MUST)

- [ ] MUST define a UISurface v0 workload component under `gitops/ameide-gitops/environments/_shared/components/apps/primitives/`.
- [ ] MUST define a UISurface v0 smoke component under the same hierarchy using an in-cluster Job probe.

### Manual Image Publish (MUST)

- [ ] MUST publish the UISurface runtime image manually to GHCR while CI publishing is disabled.

### In-Cluster Probe (MUST)

- [ ] MUST prove routability from inside the cluster (Job probe MUST fail when the route is absent and MUST pass when it is correct).

