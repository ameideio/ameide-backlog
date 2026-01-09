# 520 — Primitives Stack v2 (Repo-Aligned TDD Guide)

This document guides the end-to-end development of a single sample stack across all planes:

`proto shape` → `SDKs` → `skeleton generator` → `primitive runtime` → `operator reconcile` → `ArgoCD sync` → `in-cluster probe`

## Non-Negotiables

- Protobuf files under `packages/ameide_core_proto/src/` are the shape source.
- SDK packages are the only language-specific SDK roots:
  - `packages/ameide_sdk_go`
  - `packages/ameide_sdk_python`
  - `packages/ameide_sdk_ts`
- Operators do not build binaries, build images, push images, or fetch build tooling.
- ArgoCD/GitOps applies manifests and reconciles desired state; it does not build or push images.
- Skeleton generators are deterministic (no timestamps, random IDs, or environment-dependent output).
- Ameide-specific proto options are SOURCE-retained; generators read options from `source_file_descriptors` when present.
- Generated output is written only into generated-only directories and is gitignored.
- Runtime configuration follows the v2 “single authority” rule (cluster-derived vs GitOps/operator-provisioned vs request-provisioned); runtimes do not implement fallback/override chains (see `backlog/520-primitives-stack-v2.md`).

## Repository Locations

- Shape source protos: `packages/ameide_core_proto/src/ameide_core_proto/**/*.proto`
- Skeleton generators (Buf/protoc plugins): `plugins/**`
- Primitive runtimes (sample implementations): `primitives/<kind>/<name>/`
- Operators: `operators/*-operator/`
- GitOps deployment + smoke probes (ArgoCD): `gitops/ameide-gitops/` (submodule)
- CLI orchestrator (scaffold, dev publishing, drift checks): `packages/ameide_core_cli/`

## Responsibilities Split (Consolidated Approach)

Internal generation:
- Run `buf generate` for SDKs and per-primitive generated glue.
- Keep generated glue in generated-only directories (gitignored).

External wiring:
- Use the CLI (`ameide primitive scaffold`) for repo wiring (runtime skeleton, GitOps components/values).
- Image publishing is performed by CI to GHCR and consumed by GitOps via digest-pinned refs.

## One TDD Outer Loop

Every vertical primitive implementation follows this sequence:

1. Update the proto shape (if required).
2. Regenerate SDKs for all target languages (when proto changed).
3. Build the local generator binary (when a plugin changed).
4. Run `buf generate` with the primitive’s generation template (internal/gen glue, static outputs).
5. Scaffold runtime + GitOps wiring with `ameide primitive scaffold` (proto-driven for Go primitives).
6. Implement the implementation-owned runtime behavior until the in-cluster probe passes.
7. Publish images to GHCR via CI.
8. Deploy via ArgoCD (GitOps components + values).
9. Prove behavior via an in-cluster Job probe.

## Domain Vertical (Dev 1) — Transformation v0

### Deliverables

- Go service registration generator plugin source: `plugins/ameide_register_go/`
- Domain generation template: `packages/ameide_core_proto/buf.gen.domain-transformation.local.yaml`
- Domain runtime: `primitives/domain/transformation/`
- GitOps workload component:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`
- GitOps smoke probe component:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0-smoke.yaml`

### Generation

1. Build the plugin binary:

   - `go build -o bin/protoc-gen-ameide-register-go ./plugins/ameide_register_go`

2. Generate the Domain registration glue (do not commit this output):

   - `REPO_ROOT="$(git rev-parse --show-toplevel)"`
   - `cd "$REPO_ROOT/packages/ameide_core_proto"`
   - `PATH="$REPO_ROOT/bin:$PATH" buf generate --template buf.gen.domain-transformation.local.yaml --path src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_query.proto`

The Domain runtime imports the generated registration glue from:

- `primitives/domain/transformation/internal/gen/domain_services.generated.go`

### Publish

Image publishing is performed by CI and consumed by GitOps via digest-pinned refs. For PR preview environments, CI publish MUST ensure `ghcr.io/<org>/<image>:<HEAD_SHA>` exists for every previewed image (build changed components; retag unchanged from `:main`).

### GitOps Sync + Probe

- ArgoCD applies the Domain CR from `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`.
- The PostSync smoke Job passes by calling `grpc.health.v1.Health/Check` against `transformation-v0-domain.<ns>.svc.cluster.local:50051`.

## Remaining Verticals (Dev 2–Dev 6)

Each developer implements exactly one primitive vertical with the same outer loop and adds:

- a generator plugin under `plugins/` OR an extension to an existing generator plugin
- a `buf.gen.*.local.yaml` template under `packages/ameide_core_proto/`
- a runtime under `primitives/<kind>/<name>/` (if the primitive is runtime-based)
- GitOps workload + smoke components under `gitops/ameide-gitops/`

Projection and Integration are included in the set of verticals; they are not deferred by this plan.

All Go gRPC primitives (Domain, Process, Agent, Projection, Integration) use `plugins/ameide_register_go/` for service registration glue and do not introduce per-primitive Go registration plugins.

## Progress Trackers

Each primitive vertical is tracked in its own checklist document:

- [x] `backlog/520-primitives-stack-v2-tdd-domain.md`
- [x] `backlog/520-primitives-stack-v2-tdd-process.md`
- [x] `backlog/520-primitives-stack-v2-tdd-agent.md`
- [x] `backlog/520-primitives-stack-v2-tdd-projection.md`
- [x] `backlog/520-primitives-stack-v2-tdd-integration.md`
- [x] `backlog/520-primitives-stack-v2-tdd-uisurface.md`
