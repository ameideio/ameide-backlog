# 520 TDD — Integration Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Proto Shape (MUST)

- [ ] MUST add or select the Integration shape source protos under `packages/ameide_core_proto/src/ameide_core_proto/`.
- [ ] MUST define a v0 Integration surface that is sufficient to prove a deployed integration produces an observable effect.
- [ ] MUST keep all environment-specific secrets out of proto shape.

### SDKs (MUST)

- [ ] MUST have SDK stubs for any Integration control surfaces that are part of the shape source.

### Skeleton Generator (MUST)

- [ ] MUST implement the Integration generator plugin under `plugins/`.
- [ ] MUST generate artifacts that contain no sensitive literals.
- [ ] MUST provide a Buf generation template under `packages/ameide_core_proto/` named `buf.gen.*.local.yaml`.
- [ ] MUST write generated output only to generated-only roots and MUST gitignore them.

### Runtime / Artifacts (MUST)

- [ ] MUST define how the integration artifacts are packaged (ConfigMap, OCI artifact, or image) and MUST keep the packaging deterministic.

### Operator (MUST)

- [ ] MUST reconcile the Integration CR into the underlying integration control-plane resources and set `Ready=True` only when the integration is deployed and running.

### GitOps (MUST)

- [ ] MUST define an Integration v0 workload component under `gitops/ameide-gitops/environments/_shared/components/apps/primitives/`.
- [ ] MUST define an Integration v0 smoke component under the same hierarchy using an in-cluster Job probe.

### Manual Publish (MUST)

- [ ] MUST publish integration artifacts manually while CI publishing is disabled.

### In-Cluster Probe (MUST)

- [ ] MUST prove the integration’s observable effect from inside the cluster (Job probe MUST fail when the flow is not running and MUST pass when it is).

