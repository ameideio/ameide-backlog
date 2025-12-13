# 520 — Primitives Stack v2 (Repo-Aligned TDD Guide)

This document MUST guide the end-to-end development of a single sample stack across all planes:

`proto shape` → `SDKs` → `skeleton generator` → `primitive runtime` → `operator reconcile` → `ArgoCD sync` → `in-cluster probe`

The only permitted deviation for this activity is: image publishing MUST be done manually.

## Non-Negotiables (MUST / MUST NOT)

- Protobuf files under `packages/ameide_core_proto/src/` MUST be the shape source.
- SDK packages MUST be the only language-specific SDK roots:
  - `packages/ameide_sdk_go`
  - `packages/ameide_sdk_python`
  - `packages/ameide_sdk_ts`
- Operators MUST NOT build binaries, build images, push images, or perform network fetches for build tooling.
- ArgoCD/GitOps MUST NOT build images. ArgoCD MUST only apply manifests and reconcile desired state.
- Skeleton generators MUST be deterministic (no timestamps, random IDs, environment-dependent output).
- Ameide-specific proto options MUST be SOURCE-retained, and generators MUST read options from `source_file_descriptors` when present.
- Generated skeleton output MUST be written only into generated-only directories and MUST be gitignored.

## Repository Locations (MUST)

- Shape source protos: `packages/ameide_core_proto/src/ameide_core_proto/**/*.proto`
- Skeleton generators (Buf/protoc plugins): `plugins/**`
- Primitive runtimes (sample implementations): `primitives/<kind>/<name>/`
- Operators: `operators/*-operator/`
- GitOps deployment + smoke probes (ArgoCD): `gitops/ameide-gitops/` (submodule)

## One TDD Outer Loop (MUST)

Every vertical primitive implementation MUST follow this sequence:

1. Update the proto shape (if required).
2. Regenerate or sync SDKs for all target languages (if proto changed).
3. Build the local skeleton generator binary (if generator changed).
4. Run `buf generate` with the primitive’s generation template.
5. Implement the human-owned runtime behavior until the in-cluster probe passes.
6. Build and publish the runtime image to a registry the cluster can pull from.
7. Deploy via ArgoCD (GitOps components + values).
8. Prove behavior via an in-cluster Job probe.

## Temporary: Skip CI Publishing (MUST)

For this activity, CI image publishing MUST be treated as disabled.

- Every push to `dev` MUST use commit messages containing `[skip ci]`.
- No PR to `main` MUST be opened for this activity.
- Images MUST be pushed manually with the credentials in `.env`.

## Domain Vertical (Dev 1) — Transformation v0 (MUST)

### Deliverables (MUST)

- Go service registration generator plugin source: `plugins/ameide_register_go/`
- Domain generation template: `packages/ameide_core_proto/buf.gen.domain-transformation.local.yaml`
- Domain runtime: `primitives/domain/transformation/`
- GitOps workload component:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`
- GitOps smoke probe component:
  - `gitops/ameide-gitops/environments/_shared/components/apps/primitives/domain-transformation-v0-smoke/component.yaml`
  - `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0-smoke.yaml`

### Generation (MUST)

1. Build the plugin binary:

   - `go build -o bin/protoc-gen-ameide-register-go ./plugins/ameide_register_go`

2. Generate the Domain registration glue (this output MUST NOT be committed):

   - `REPO_ROOT="$(git rev-parse --show-toplevel)"`
   - `cd "$REPO_ROOT/packages/ameide_core_proto"`
   - `PATH="$REPO_ROOT/bin:$PATH" buf generate --template buf.gen.domain-transformation.local.yaml --path src/ameide_core_proto/transformation/scrum/v1/transformation-scrum-query.proto`

The Domain runtime MUST import the generated registration glue from:

- `primitives/domain/transformation/internal/gen/domain_services.generated.go`

### Manual Image Publish (MUST)

The cluster MUST pull `ghcr.io/ameideio/transformation-domain:dev` (as referenced by GitOps values).

Run:

- `REPO_ROOT="$(git rev-parse --show-toplevel)"`
- `set -a; source "$REPO_ROOT/.env"; set +a`
- `printf '%s' "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USERNAME" --password-stdin`
- `cd "$REPO_ROOT"`
- `docker build -t ghcr.io/ameideio/transformation-domain:dev -f primitives/domain/transformation/Dockerfile .`
- `docker push ghcr.io/ameideio/transformation-domain:dev`

### GitOps Sync + Probe (MUST)

- ArgoCD MUST apply the Domain CR from `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`.
- The PostSync smoke Job MUST pass by calling `grpc.health.v1.Health/Check` against `transformation-v0-domain.<ns>.svc.cluster.local:50051`.

## Remaining Verticals (Dev 2–Dev 6) (MUST)

Each developer MUST implement exactly one primitive vertical with the same outer loop and MUST add:

- a generator plugin under `plugins/` OR an extension to an existing generator plugin
- a `buf.gen.*.local.yaml` template under `packages/ameide_core_proto/`
- a runtime under `primitives/<kind>/<name>/` (if the primitive is runtime-based)
- GitOps workload + smoke components under `gitops/ameide-gitops/`

Projection and Integration MUST be included in the set of verticals; they MUST NOT be deferred by this TDD plan.

All Go gRPC primitives (Domain, Process, Agent, Projection, Integration) MUST use `plugins/ameide_register_go/` for service registration glue and MUST NOT introduce per-primitive Go registration plugins.

## Progress Trackers (MUST)

Each primitive vertical MUST be tracked in its own checklist document:

- [x] `backlog/520-primitives-stack-v2-tdd-domain.md`
- [x] `backlog/520-primitives-stack-v2-tdd-process.md`
- [x] `backlog/520-primitives-stack-v2-tdd-agent.md`
- [x] `backlog/520-primitives-stack-v2-tdd-projection.md`
- [x] `backlog/520-primitives-stack-v2-tdd-integration.md`
- [x] `backlog/520-primitives-stack-v2-tdd-uisurface.md`

## Appendix: Dev 1 Retrospective + Generator Improvements (MUST)

This appendix MUST document what was learned while implementing Domain v0 and MUST translate those learnings into generator requirements that remove developer guesswork.

### Observed Friction (FACTS)

- The runtime `go.mod` used a local `replace` to the Go SDK, which required the Docker build context to include `packages/ameide_sdk_go`.
- The gRPC surface was defined by proto shape, but the runtime still needed a deterministic and repeatable way to register services without hand-editing imports and `Register*Server` calls.
- The build/publish step was the only manual deviation; this surfaced environment drift (Docker daemon access) that is unrelated to the primitive/operator architecture.

### Generator Improvements Implemented in 520 (MUST)

- The Go registration generator MUST auto-discover services in `files_to_generate` and MUST NOT require a manual service list.
- The Go registration generator MUST generate compile-time safe wiring and MUST NOT rely on `any` + type assertions.
- The Go registration generator MUST read Ameide options from `source_file_descriptors` when present and MUST NOT assume runtime descriptors contain SOURCE-retained data.

### Generator Improvements Deferred to Follow-Up Work (MUST)

- A generator MUST generate a complete v0 runtime skeleton (server bootstrap + health + reflection) so a developer MUST NOT create `cmd/main.go` by hand.
- A generator MUST generate “create-if-missing” human-owned extension points (handlers) and MUST NOT overwrite human-owned files.
- A generator MUST emit deterministic file names and stable ordering to guarantee reproducible outputs and reliable regen-diff gating.

### Repo Automation Improvements (MUST)

- The repo MUST provide a single command that runs the vertical loop for each primitive (generate → build → push → GitOps sync → probe).
- The repo MUST provide a single command that validates generator determinism via golden tests for each generator plugin.
