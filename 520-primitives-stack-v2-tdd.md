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

- Domain generator plugin source: `plugins/ameide_domain_go/`
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

   - `go build -o bin/protoc-gen-ameide-domain-go ./plugins/ameide_domain_go`

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

- a generator plugin under `plugins/`
- a `buf.gen.*.local.yaml` template under `packages/ameide_core_proto/`
- a runtime under `primitives/<kind>/<name>/` (if the primitive is runtime-based)
- GitOps workload + smoke components under `gitops/ameide-gitops/`

Projection and Integration MUST be included in the set of verticals; they MUST NOT be deferred by this TDD plan.

## Progress Trackers (MUST)

Each primitive vertical MUST be tracked in its own checklist document:

- [x] `backlog/520-primitives-stack-v2-tdd-domain.md`
- [ ] `backlog/520-primitives-stack-v2-tdd-process.md`
- [ ] `backlog/520-primitives-stack-v2-tdd-agent.md`
- [ ] `backlog/520-primitives-stack-v2-tdd-projection.md`
- [ ] `backlog/520-primitives-stack-v2-tdd-integration.md`
- [ ] `backlog/520-primitives-stack-v2-tdd-uisurface.md`
