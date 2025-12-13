# 520 TDD — Integration Vertical Progress

All unchecked items in this document MUST be completed for this vertical primitive to be considered complete.

## Full-Stack Checklist

### Shape Source (MUST)

- [x] MUST keep the Integration v0 proto at `packages/ameide_core_proto/src/ameide_core_proto/integration/v1/integration_v0.proto`.
- [x] MUST keep all environment-specific secrets out of proto shape.

### SDKs (MUST)

- [x] MUST generate Go SDK stubs via `packages/ameide_core_proto/buf.gen.sdk-go.local.yaml`.

### Skeleton Generator (MUST)

- [x] MUST use `plugins/ameide_register_go/` to generate Go service registration glue.
- [x] MUST keep the Integration glue template at `packages/ameide_core_proto/buf.gen.integration-echo.local.yaml`.
- [x] MUST write the generated glue to `primitives/integration/echo/internal/gen/` and MUST keep it gitignored.

### Runtime (MUST)

- [x] MUST keep the Integration runtime at `primitives/integration/echo/`.
- [x] MUST implement deterministic v0 behavior in `ameide_core_proto.integration.v1.IntegrationV0Service/Execute`.
- [x] MUST expose gRPC on port `50051` and MUST serve gRPC health.

### Operator + GitOps (MUST)

- [x] MUST keep the Integration CRD in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_integrations.yaml`.
- [x] MUST keep the Integration operator deployment in `gitops/ameide-gitops/sources/charts/platform/ameide-operators/templates/integration-operator-deployment.yaml`.
- [x] MUST keep the v0 Integration workload + smoke components:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/integration-echo-v0/component.yaml`
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/integration-echo-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/integration-echo-v0.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/integration-echo-v0-smoke.yaml`

### Manual Image Publish (MUST)

- [x] MUST publish the runtime image `ghcr.io/ameideio/integration-echo:dev`.
- [x] MUST publish the operator image `ghcr.io/ameideio/integration-operator:dev`.
