# 520 TDD — UISurface Vertical Progress

Complete all unchecked items in this document for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source

- [x] Keep the UISurface v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/uisurface/v1/hello_uisurface.proto`.

### Skeleton Generator

- [x] Keep the UISurface static generator at `plugins/ameide_uisurface_static/`.
- [x] Keep the UISurface generation template at `packages/ameide_core_proto/buf.gen.uisurface-hello.local.yaml`.
- [x] Write generated output to `primitives/uisurface/hello/internal/gen/` and keep it gitignored.

### Runtime

- [x] Keep the UISurface runtime at `primitives/uisurface/hello/`.
- [x] Serve the generated marker response `Hello UISurface v0`.
- [ ] Configuration authority: runtime wiring is GitOps/operator-provisioned only (env/secret/volume), request inputs are request-provisioned only, and the runtime has no fallback/override chains (see `backlog/520-primitives-stack-v2.md`).

### Operator + GitOps

- [x] Keep the UISurface CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_uisurfaces.yaml`.
- [x] Keep the UISurface operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/uisurface-operator-deployment.yaml`.
- [x] Keep the v0 UISurface workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/uisurface-hello-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/uisurface-hello-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/uisurface-hello-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/uisurface-hello-v0-smoke.yaml`

### Publish

- [x] CI publishes the runtime image to GHCR; GitOps deploys only digest-pinned refs (no floating tags). See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
