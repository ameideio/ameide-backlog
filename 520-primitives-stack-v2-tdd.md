# 520 — Primitives Stack v2 (Repo-Aligned TDD Guide)

This document guides the end-to-end development of a single sample stack across all planes:

`proto shape` → `SDKs` → `skeleton generator` → `primitive runtime` → `operator reconcile` → `ArgoCD sync` → `in-cluster probe`

The only permitted deviation for this activity is: image publishing is done manually (by pushing `:dev` tags), while GitOps deployments still remain digest-pinned (see `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`).

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
- Use the CLI (`ameide primitive publish`) for dev runtime image build/push when CI publishing is skipped.
- Use `./scripts/build-all-images.sh dev operators/<operator>` for manual operator image build/push.

## One TDD Outer Loop

Every vertical primitive implementation follows this sequence:

1. Update the proto shape (if required).
2. Regenerate SDKs for all target languages (when proto changed).
3. Build the local generator binary (when a plugin changed).
4. Run `buf generate` with the primitive’s generation template (internal/gen glue, static outputs).
5. Scaffold runtime + GitOps wiring with `ameide primitive scaffold` (proto-driven for Go primitives).
6. Implement the implementation-owned runtime behavior until the in-cluster probe passes.
7. Build and publish the runtime image to a registry the cluster can pull from.
8. Deploy via ArgoCD (GitOps components + values).
9. Prove behavior via an in-cluster Job probe.

## Temporary: Skip CI Publishing

For this activity, treat CI image publishing as disabled.

- Every push to `dev` uses commit messages containing `[skip ci]`.
- No PR to `main` is opened for this activity.
- Images are pushed manually with the credentials in `.env`.
- After pushing a new `:dev` image, update GitOps by writing the resolved digest into `sources/values/env/{local,dev}/**`:
  - run `bash scripts/bump-local-dev-images.sh` (or trigger `.github/workflows/bump-local-dev-images.yaml`)
  - ArgoCD rollouts then happen because Git changed (digest-pinned ref changed)
- Do not use `spec.imagePullPolicy: Always` as a rollout mechanism; keep pull policy boring (`IfNotPresent`) and drive rollouts via digest updates.

### Smoke probes in ArgoCD (GitOps team feedback)

- A green ArgoCD application does not prove smokes ran recently. Smokes are PostSync hook Jobs and only run when the `*-smoke` application syncs.
- To run a smoke, sync the corresponding `*-smoke` application and inspect the Job result (Succeeded/Failed) and logs.

### Smoke image baseline (gRPC)

The gRPC smoke jobs run shell scripts via `/bin/sh -c`. Use the grpcurl runner image that includes a shell:

- `ghcr.io/ameideio/grpcurl-runner:dev`

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

### Manual Image Publish

Producer pushes `ghcr.io/ameideio/transformation-domain:dev`, and GitOps deploys the resolved digest-pinned ref after the local/dev bump PR lands (see `backlog/603-image-pull-policy.md`).

Publish with the CLI:

- `ameide primitive publish --kind domain --name transformation`

If you need to publish without primitive discovery (tooling images), provide `--image` and `--dockerfile`, for example:

- `ameide primitive publish --image ghcr.io/ameideio/grpcurl-runner:dev --dockerfile tools/images/grpcurl-runner/Dockerfile`

Equivalent manual commands:

- `REPO_ROOT="$(git rev-parse --show-toplevel)"`
- `set -a; source "$REPO_ROOT/.env"; set +a`
- `printf '%s' "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USERNAME" --password-stdin`
- `cd "$REPO_ROOT"`
- `docker build -t ghcr.io/ameideio/transformation-domain:dev -f primitives/domain/transformation/Dockerfile .`
- `docker push ghcr.io/ameideio/transformation-domain:dev`

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

## Appendix: Dev 1 Retrospective + Generator Improvements

This appendix documents what was learned while implementing Domain v0 and translates those learnings into generator requirements that remove developer guesswork.

### Observed Friction (FACTS)

- The runtime `go.mod` used a local `replace` to the Go SDK, which required the Docker build context to include `packages/ameide_sdk_go`.
- The gRPC surface was defined by proto shape, but the runtime still needed a deterministic and repeatable way to register services without hand-editing imports and `Register*Server` calls.
- The build/publish step was the only manual deviation; this surfaced environment drift (Docker daemon access) that is unrelated to the primitive/operator architecture.

### Generator Improvements Implemented in 520

- The Go registration generator auto-discovers services in `files_to_generate`; it does not require a manual service list.
- The Go registration generator generates compile-time safe wiring; it does not rely on `any` or type assertions.
- The Go registration generator reads Ameide options from `source_file_descriptors` when present; it does not assume runtime descriptors contain SOURCE-retained data.

### Generator Improvements Deferred to Follow-Up Work

- Generate a complete v0 runtime skeleton (server bootstrap + health + reflection) so a developer does not create `cmd/main.go` by hand.
- Generate “create-if-missing” implementation-owned extension points (handlers) without overwriting implementation-owned files.
- Emit deterministic file names and stable ordering to guarantee reproducible outputs and reliable regen-diff gating.

### Repo Automation Improvements

- Provide a single command that runs the vertical loop for each primitive (generate → build → push → GitOps sync → probe).
- Provide a single command that validates generator determinism via golden tests for each generator plugin.
