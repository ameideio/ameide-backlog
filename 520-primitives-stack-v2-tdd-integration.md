# 520 TDD — Integration Vertical Progress

Complete all unchecked items in this document for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source

- [x] Keep the Integration v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/integration/v1/integration_v0.proto`.
- [x] Keep all environment-specific secrets out of proto shape.

### SDKs

- [x] Generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator

- [x] Use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] Keep the Integration glue template at `packages/ameide_core_proto/buf.gen.integration-echo.local.yaml`.
- [x] Write the generated glue to `primitives/integration/echo/internal/gen/` and keep it gitignored.

### Runtime

- [x] Keep the Integration runtime at `primitives/integration/echo/`.
- [x] Implement deterministic v0 behavior in `ameide_core_proto.integration.v1.IntegrationV0Service/Execute`.
- [x] Expose gRPC on port `50051` and serve gRPC health.

### Operator + GitOps

- [x] Keep the Integration CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_integrations.yaml`.
- [x] Keep the Integration operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/integration-operator-deployment.yaml`.
- [x] Keep the v0 Integration workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/integration-echo-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/integration-echo-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/integration-echo-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/integration-echo-v0-smoke.yaml`

### Manual Image Publish

- [x] Publish the runtime image `ghcr.io/ameideio/integration-echo:dev` (use `ameide primitive publish --kind integration --name echo`).
- [x] Publish the operator image `ghcr.io/ameideio/integration-operator:dev` (use `ameide primitive publish --image ghcr.io/ameideio/integration-operator:dev --dockerfile operators/integration-operator/Dockerfile.dev`).
