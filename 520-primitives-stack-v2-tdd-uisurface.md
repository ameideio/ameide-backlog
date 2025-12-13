# 520 TDD — UISurface Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source (MUST)

- [x] MUST keep the UISurface v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/uisurface/v1/hello_uisurface.proto`.

### Skeleton Generator (MUST)

- [x] MUST keep the UISurface static generator at `plugins/ameide_uisurface_static/`.
- [x] MUST keep the UISurface generation template at `packages/ameide_core_proto/buf.gen.uisurface-hello.local.yaml`.
- [x] MUST write generated output to `primitives/uisurface/hello/internal/gen/` and MUST keep it gitignored.

### Runtime (MUST)

- [x] MUST keep the UISurface runtime at `primitives/uisurface/hello/`.
- [x] MUST serve the generated marker response `Hello UISurface v0`.

### Operator + GitOps (MUST)

- [x] MUST keep the UISurface CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_uisurfaces.yaml`.
- [x] MUST keep the UISurface operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/uisurface-operator-deployment.yaml`.
- [x] MUST keep the v0 UISurface workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/uisurface-hello-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/uisurface-hello-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/uisurface-hello-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/uisurface-hello-v0-smoke.yaml`

### Manual Image Publish (MUST)

- [x] MUST publish the runtime image `ghcr.io/ameideio/uisurface-hello:dev`.
