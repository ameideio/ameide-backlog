> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 356 – Buf Managed Mode Migration

> **Superseded for v6 service consumption:** v6 converged on **wrapper SDK-only** contract consumption for runtime services and scenario runners (no direct `@buf/*` or `buf.build/gen/*` imports outside the SDK build pipeline). Use:
> - `backlog/715-v6-contract-spine-doctrine.md` (doctrine: SDK-only; Kafka + CloudEvents)
> - `backlog/300-400/393-ameide-sdk-import-policy.md` (enforced import rules)
> - `backlog/410-bsr-native.md` and `backlog/408-workspace-first-ring-2.md` (workspace-first rings)
>
> This document is retained as historical context from the earlier “services consume Buf artifacts directly” phase.

## Goal

Adopt Buf’s “Managed Mode” and Generated SDK workflows across the stack so every
consumer builds or downloads language artefacts directly from the canonical proto
module, without vendored code or custom install scripts.

## Current State

- `packages/ameide_core_proto` is Buf-managed (`buf.lock` in place) and `.github/workflows/ci-proto.yml:49`
  already pushes `buf.build/ameideio/ameide` on `main`. Every SDK sync script now
  pulls Generated SDK artefacts straight from BSR and aborts on failure
  (`packages/ameide_sdk_go/scripts/sync-proto.mjs:148-236`,
  `packages/ameide_sdk_ts/scripts/sync-proto.mjs:180-271`,
  `packages/ameide_sdk_python/scripts/sync_proto.py:300-404`). The manifests under
  each package (`packages/ameide_sdk_go/gen/.proto-manifest.json#L4`,
  `packages/ameide_sdk_ts/src/proto/.proto-manifest.json#L6`,
  `packages/ameide_sdk_python/src/ameide_sdk/proto/.proto-manifest.json#L4`)
  record `source: "bsr"`, confirming the new download path.
- Verdaccio/devpi/Athens Helm layers have been removed from infra (for example
  `infra/kubernetes/charts/external-services` and `infra/kubernetes/helmfiles`
  no longer contain registry charts) and the release tooling has been retargeted
  to Buf. The language publish helpers now download from Buf registries
  (`release/publish.sh`, `release/publish_go.sh`, `release/publish_python.sh`)
  and the new `sdk-smoke` job in `.github/workflows/ci-proto.yml:83` runs them on
  every push to ensure artefacts stay fetchable.
- Go services already depend on Buf modules (`services/agents/go.mod:6-8`,
  `services/inference_gateway/go.mod:6-8`). The CLI now imports the Buf-generated
  stubs directly and ships a tiny internal dialer so it no longer requires the
  workspace SDK (`packages/ameide_core_cli/internal/commands/workflow.go:11`,
  `packages/ameide_core_cli/internal/sdk/dial.go:7`).
- TypeScript services moved to Buf npm packages (`services/platform/package.json:17`,
  `services/graph/package.json:17`, `services/threads/package.json:17`,
  `services/www_ameide/package.json:16`). The public portal now re-exports proto
  namespaces locally (`services/www_ameide/src/proto/index.ts:1`) and builds
  `@connectrpc/connect` clients (via `createClient` + `connect-web` transports) using
  Buf descriptors, while Dockerfiles default to the public
  npm + Buf registries. The legacy `@ameideio/ameide-sdk-ts` package still exists in
  the workspace and needs a deprecation plan.
- Threads service now mirrors the Buf-only pattern end-to-end: it re-exports the
  required descriptors from `services/threads/src/proto/index.ts`, creates
  Connect clients locally via `createClient`
  (`services/threads/src/proto/clients.ts`), and ships a
  first-party telemetry bootstrap (`services/threads/src/telemetry.ts`) so the
  runtime no longer imports the legacy workspace telemetry packages.
- Transformation service now consumes Buf-published artefacts end-to-end. The
  service depends on `@buf/ameideio_ameide.bufbuild_es`
  (`services/transformation/package.json:17`),
  re-exports the required descriptors from a local `src/proto` facade
  (`services/transformation/src/proto/index.ts`) and instantiates clients with
  Connect directly (`services/transformation/src/proto/clients.ts`). The server
  was converted from manual gRPC/proto-loader wiring to a Connect HTTP/2 router
  that uses the Buf descriptors (`services/transformation/src/server.ts`), and
  all runtime/test imports now resolve through the local proto module instead of
  `@ameideio/ameide-sdk-ts`. Container builds append the Buf registry scope so no
  Verdaccio entries are required (`services/transformation/Dockerfile.release:32`,
  `services/transformation/Dockerfile.dev:28`).
- Workflows service now consumes Buf packages end-to-end. Runtime deps point at
  `@buf/ameideio_ameide` scopes (`services/workflows/package.json:18`), a local
  proto facade re-exports descriptors plus enum helpers and Connect clients
  (`services/workflows/src/proto/index.ts:1`,
  `services/workflows/src/proto/enums.ts:1`,
  `services/workflows/src/proto/clients.ts:1`), and the service/tests import
  through that layer (`services/workflows/src/grpc.ts:6`,
  `services/workflows/src/grpc-handlers.ts:5`,
  `services/workflows/__tests__/integration/workflows-service.e2e.test.ts:5`).
  Docker builds pin the Buf registry scope (`services/workflows/Dockerfile.release:11`,
  `services/workflows/Dockerfile.dev:9`), and TS tooling is configured for the
  ESM outputs (`services/workflows/tsconfig.json:13`,
  `services/workflows/jest.config.mjs:39`).
- Transformation service now consumes Buf-published artefacts end-to-end. The
  service depends on `@buf/ameideio_ameide.bufbuild_es`
  (`services/transformation/package.json:17`),
  re-exports the descriptors via `src/proto/index.ts` and new client helpers
  (`services/transformation/src/proto/clients.ts`), and all runtime/tests import
  from that local module (for example `services/transformation/src/common/context.ts:1`,
  `services/transformation/src/transformations/service.ts:7`,
  `services/transformation/__tests__/integration/transformation-service.grpc.test.ts:5`).
  The server now registers Connect handlers instead of manually loading raw protos
  (`services/transformation/src/server.ts`), and Docker builds configure the Buf
  registry scope (`services/transformation/Dockerfile.release:32`,
  `services/transformation/Dockerfile.dev:28`).
- Python runtime services continue to consume the workspace `ameide-sdk`
  wheel (`services/workflows_runtime/pyproject.toml:9`), so no Buf wheel is in use yet.
- Validation is unchanged: protos continue to import `buf/validate`
  (`packages/ameide_core_proto/src/ameide_core_proto/graph/v1/graph_service.proto:7`)
  and SDK manifests/configs show no Protovalidate runtime dependencies
  (`packages/ameide_sdk_go/go.mod`, `packages/ameide_sdk_ts/package.json`,
  `packages/ameide_sdk_python/pyproject.toml`).

## Latest Progress (2025-11-06)

- **Buf-only runtime import path proven:** Transformation service dropped
  `@ameideio/ameide-sdk-ts` and now resolves all descriptors through Buf packages,
  validating the Managed Mode “download don’t commit” workflow for backend gRPC
  services.
- **Connect bootstrap alignment:** The legacy proto-loader logic was replaced
  with Connect’s HTTP/2 server, confirming that descriptor-based routing works
  with existing observability/metrics hooks (`services/transformation/src/server.ts`).
- **Tooling and build parity:** `pnpm --filter @ameide/transformation-service run build`
  and Jest suites (`pnpm --filter @ameide/transformation-service run test`) pass
  after the migration, showing no hidden Verdaccio/devpi dependencies.
- **Container pipeline ready:** Dockerfiles append the Buf registry scope,
  proving CI/CD images can fetch the new packages without the retired package
  mirrors.
- **Backlog impact:** Demonstrates how remaining TypeScript services can follow
  the same pattern; informs the plan for removing the workspace SDK entirely.

## Target Architecture

1. **Single Proto Module**
   - Keep `.proto` files, `buf.yaml`, and `buf.gen.yaml` in `packages/ameide_core_proto`.
   - Enable Managed Mode (`managed.enabled: true`) to lock plugin versions.
   - Add Buf lint/format hooks (CI + pre-commit) to enforce schema hygiene.

2. **Language Generation via Managed Mode**
   - Remove generated code from the repo (`packages/ameide_sdk_go/gen`, TS `dist`,
     Python `generated` modules).
   - Only the SDK packaging pipelines (and Buf’s Generated SDK jobs) invoke `buf generate`.
     Application services download language artefacts from Buf; only `.proto`
     files remain in Git.
   - Buf caches the generation config; plugin updates happen centrally in
     `buf.gen.yaml`.
   - ✅ Transformation service now validates the download-only model by relying
     solely on Buf npm packages (no generated TS stubs checked into the repo)
     and a thin local re-export layer.

3. **Published SDK Artefacts**
   - Use Buf’s Generated SDK feature (or equivalent CI jobs) to produce
     versioned packages for Go, TypeScript, and Python.
   - Publish to BSR and document any optional mirroring strategy (Verdaccio/devpi
     stay retired) if internal registries are required.
   - Services consume the published packages; local builds pull from the
     registry instead of regenerating in the repo.
   - ✅ Transformation service build/test/Docker workflows now exercise this
     pattern by installing Buf npm packages directly and verifying they are
     available during CI (see `pnpm --filter @ameide/transformation-service run build`).

4. **Protovalidate Adoption**
   - Migrate schemas from `buf.validate` annotations to Protovalidate.
   - Update code generation to use Protovalidate plugins.
   - Add Protovalidate runtimes via official packages (Go: module import,
     TypeScript: `@bufbuild/protovalidate`, Python: `protovalidate` package) so
     no manual vendoring is needed.

5. **CI & Tooling**
   - Replace bespoke scripts (e.g., `ensure-build.sh`) with Buf commands:
     `buf format`, `buf lint`, `buf breaking`, `buf generate`.
   - Ensure CI runs Buf lint + breaking change checks before publishing.
   - Simplify Tilt/bootstrapping: services consume published Buf SDK packages
     only—no container build or dev workflow may call `buf generate` or protoc.

6. **Developer Workflow**
   - Developers update `.proto` files and rerun `buf generate` locally when
     needed (Managed Mode ensures consistent outputs). No generated code is
     committed.
   - Services pull published SDK packages (or Buf-generated artefacts) as part of
     their builds; changes are isolated to proto definitions.
   - Tilt now requires a valid `BUF_TOKEN` (for `buf push`) before reconciling infra layers; document how engineers obtain/store the token so local loops never fall back to ad-hoc generation.

## Migration Steps

1. **Preparation & Tooling Hardening** — ✅ (2025-11-06)
   - Enable Managed Mode (`managed.enabled: true`) in every Buf template (`packages/ameide_core_proto/buf.gen*.yaml`) and align `buf.lock`.
   - Register `buf.build/ameideio/ameide`; ensure BSR credentials are available locally and in CI (`BUF_TOKEN`).
   - Upgrade local tooling (`buf`, `pnpm`, `uv`, Go) to supported versions; install Buf CLI, `curl`, and `tar` everywhere the SDK sync scripts run.
   - Inventory bespoke proto-generation scripts (`ensure-build.sh`, language-specific `sync-proto`) and flag those destined for removal.

2. **Repository Cleanup** — ✅ (2025-11-06)
   - Delete committed generated artefacts (Go `packages/ameide_sdk_go/gen`, TS `packages/ameide_sdk_ts/src/proto/gen`, TS `dist/proto`, Python `packages/ameide_sdk_python/src/ameide_sdk/generated`).
   - Add `.gitignore` rules under each package (`ameide_core_proto`, `ameide_sdk_go`, `ameide_sdk_ts`, `ameide_sdk_python`) so regenerated assets never reappear.
   - Remove Bazel/Bazelisk configs or legacy proto vendor directories once confirmed unused; update lint/pre-commit rules blocking proto imports from application code.

3. **Protovalidate Adoption (Greenfield)** — 🚧 (no runtime coverage in any language yet)
   - The platform is not live, so we are not juggling legacy validators. All client SDKs must ship with Protovalidate wired in before the first release.
   - Current readiness matrix:

     | Aspect | Go SDK | TypeScript SDK | Python SDK | Evidence |
     | --- | --- | --- | --- | --- |
     | Proto annotations prepared for Protovalidate | ❌ Shared protos still import `buf/validate/validate.proto`; no generated descriptors checked in. | Same shared schema; nothing Protovalidate-specific yet. | Same shared schema; nothing Protovalidate-specific yet. | `packages/ameide_core_proto/src/ameide_core_proto/graph/v1/graph_service.proto:7` |
     | Buf generation emits Protovalidate artefacts | ❌ `packages/ameide_core_proto/buf.gen.yaml` only invokes protoc/grpc/connect plugins; no Protovalidate plugin configured. | ❌ `packages/ameide_core_proto/buf.gen.yaml` only calls `protoc-gen-es`; no Protovalidate hooks. | ❌ `packages/ameide_core_proto/buf.gen.yaml` targets python/grpc only. | `packages/ameide_core_proto/buf.gen.yaml` |
     | Runtime dependency added | ❌ `packages/ameide_sdk_go/go.mod` has no `github.com/bufbuild/protovalidate-go`. | ❌ `packages/ameide_sdk_ts/package.json` lacks `@bufbuild/protovalidate`. | ❌ `packages/ameide_sdk_python/pyproject.toml` lacks a `protovalidate` dependency. | File refs as listed. |
     | Validation exercised in code/tests | ❌ No Protovalidate usage anywhere in repo (`rg "protovalidate"` returns only backlog mentions). | Same. | Same. | `rg "protovalidate" -n` |
     | CI / tooling guards | ❌ No Protovalidate CLI invocations in workflows or scripts. | Same. | Same. | Workflows/scripts search. |

   - Implementation tasks (deliver all languages together):
     - Use Buf CLI to regenerate schema metadata with Protovalidate-aware annotations (`buf beta protovalidate migrate`) and commit the updated proto definitions.
     - Extend all Buf templates (`buf.gen*.yaml`) so Managed Mode produces Protovalidate artefacts alongside the existing protobuf/grpc outputs.
     - Add official Protovalidate libraries to each SDK (`github.com/bufbuild/protovalidate-go`, `@bufbuild/protovalidate`, `protovalidate`) and wire validation into request/response helpers.
     - Introduce golden-path validation tests per SDK (e.g., expected error payloads) and gate them in CI so regressions surface immediately.
     - Update CLI/local workflow checks to fail fast if the Protovalidate runtime or generated assets are missing—there is no fallback mode.

4. **Generated SDK Pivot** — 🚧 (BSR downloads live; publish + consumer rollout pending)
   - Current readiness matrix:

     | Checkpoint | Go SDK | TypeScript SDK | Python SDK | Evidence |
     | --- | --- | --- | --- | --- |
    | Sync script pulls exclusively from BSR | ✅ | ✅ | ✅ | `packages/ameide_sdk_go/scripts/sync-proto.mjs:148-236`, `packages/ameide_sdk_ts/scripts/sync-proto.mjs:180-271`, `packages/ameide_sdk_python/scripts/sync_proto.py:300-404` |
    | Manifest/source pinned to BSR | ✅ | ✅ | ✅ | `packages/ameide_sdk_go/gen/.proto-manifest.json#L4`, `packages/ameide_sdk_ts/src/proto/.proto-manifest.json#L6`, `packages/ameide_sdk_python/src/ameide_sdk/proto/.proto-manifest.json#L4` |
    | Primary consumers using Buf artefacts | ✅ Services and CLI import Buf modules (`services/agents`, `services/inference_gateway`, `packages/ameide_core_cli/internal/...`). | ✅ Node apps depend on Buf-published npm packages (`services/platform/package.json:17`, `services/graph/package.json:17`, `services/threads/package.json:17`) and proxy descriptors locally (`services/threads/src/proto/index.ts`). | ❌ Runtime services depend on `ameide-sdk` wheels built locally (`services/workflows_runtime/pyproject.toml:9`). | Go and TypeScript use Buf; Python still pending. |
    | Publish config retargeted to Buf | ✅ `release/publish_go.sh` verifies Buf proxy access. | ✅ `release/publish.sh` downloads tarballs from Buf’s npm registry. | ✅ `release/publish_python.sh` verifies Buf’s pip index. | Release README documents the Buf-first flow. |

   - Follow-up actions to complete the pivot:
     - Track Buf SDK releases and bump dependent Go services (`services/agents`, `services/inference_gateway`, CLI) whenever versions change; document this in the release checklist.
     - Publish TypeScript and Python SDKs through Buf (or designate a mirror job) and switch downstream package.json/pyproject dependencies from the workspace tarballs to the Buf registries.
     - ✅ Tilt/local workflows now depend on Buf: the `ameide_core_proto` Tilt resource runs `infra/kubernetes/scripts/buf-build-push.sh`, and the former `ameide_sdk_*` generators were removed, so dev loops pull/push through BSR just like CI.

5. **CI / Publishing** — 🚧 (Buf push live; release orchestration catching up)
   - `.github/workflows/ci-proto.yml` now includes the `sdk-smoke` job that runs
     `release/publish.sh`, `release/publish_go.sh`, and `release/publish_python.sh`
     to download Buf artefacts on every push. We still need to mirror those
     checks in the language-specific release workflows so the smoke step blocks
     a publish if Buf access breaks.
   - The release helpers themselves now target Buf registries; track down any
     callers that still expect Verdaccio/devpi/Athens hooks and delete the dead
     references from Tilt or infra once consumers are switched.
   - Tilt enforces Buf publication before any Helm work: `ameide_core_proto` shells out to `infra/kubernetes/scripts/buf-build-push.sh`, which fails fast unless `BUF_TOKEN` is present and `buf build/push` succeed. Document this requirement in the publish runbooks so local/CI paths export the secret.
   - Add gating around version bumps (e.g., ensure Buf module is up to date and
     versions are recorded in changelog tooling) to keep the Buf-first flow
     audit-friendly.

6. **Consumer & Pipeline Updates** — 🚧
   - **Go:** Application services (`services/agents/go.mod:6-8`,
     `services/inference_gateway/go.mod:6-8`) already import `buf.build/gen/...`
     modules. The CLI now does the same through `internal/sdk`, so the remaining
     cleanup is to drop the `packages/ameide_sdk_go` workspace dependency from
     generator commands (`packages/ameide_core_cli/internal/commands/generate.go:48`)
     and delete the legacy module once no scripts reference it.
  - **TypeScript:** Platform, graph, threads, workflows, and web apps all point
    to `@buf/ameideio_ameide.*` packages (`services/platform/package.json:17`,
    `services/graph/package.json:17`, `services/threads/package.json:17`,
    `services/www_ameide/package.json:16`,
    `services/www_ameide_platform/package.json:31`). The public portal now owns
    a local proto barrel (`services/www_ameide/src/proto/index.ts:1`) and
    transport helpers (`services/www_ameide/src/proto/clients.ts:1`), plus its
    Dockerfiles reference the public npm registry and Buf scope by default.
    Workflows service is partially migrated—local facades live in
    `services/workflows/src/proto/index.ts` and tests/client helpers consume
    them—but `pnpm --filter @ameide/workflows-service build` currently fails
    because the Buf TS descriptors do not expose several `*ResponseSchema`
    symbols expected by `services/workflows/src/grpc-handlers.ts:562-666`, and
    integration tests require a running Postgres instance to apply migrations.
    Remaining work: deprecate the workspace `@ameideio/ameide-sdk-ts` package,
    migrate any consumers still importing helpers from it (notably
    `www_ameide_platform`), and scrub Tilt/docs of Verdaccio references.
   - **Python:** Service packages such as
     `services/workflows_runtime/pyproject.toml:9` still install `ameide-sdk`
     from the repo. Switch them to the Buf wheel, update imports to the Buf
     module layout, and remove the `uv run scripts/sync_proto.py` hooks and
     Dockerfile mounts once consumers are migrated.
   - **Tooling cleanup:** With all consumers on Buf, retire the remaining local
     bootstrap helpers (e.g., `packages/ameide_core_proto/scripts/ensure-build.sh` and
     the SDK sync scripts) so CI/Tilt no longer stage generated artefacts locally.

7. **Documentation, Training & Guardrails** — 🚧
   - Refresh developer docs (`packages/ameide_core_proto/README.md`, `.github/workflows/README.md`, `docs/sdk-only-policy.md`) to describe BSR-first workflows.
   - Service-level READMEs start reflecting the Buf-only flow (for example
     `services/www_ameide/README.md:42` now explains the local proto barrel and
     Buf registry usage); replicate across remaining services.
   - Update onboarding/backlog notes to cover Buf login, module push expectations, and failure scenarios (e.g., how to handle 404s when BSR access is missing).
   - Add runtime checks: CLI targets or preflight scripts should verify `.proto-manifest.json` provenance (`source: bsr`) and fail if language SDKs are missing.
   - Communicate the new contract to service owners: all proto changes require `buf push`, and SDK consumers must refresh via the published packages, not local generation.

## Open Questions

- Do we still need the curated SDK wrappers (`ameide_sdk_go`, `@ameideio/ameide-sdk-ts`,
  `ameide-sdk`) once every consumer can lean on Buf artefacts, or can we replace
  them with thin composition libraries?
- Should we mirror Buf outputs into a single internal registry for compliance,
  or is direct Buf consumption acceptable for prod environments?
- How do we describe versioning once Buf is the source of truth (semver tags vs.
  timestamped pseudo-versions) so downstream automation can pin releases?

**v6 resolution (715):**

- Keep wrapper SDKs as the only supported import surface for services and scenario runners.
- Direct Buf artifact consumption by runtime services is not permitted; Buf/BSR is confined to the SDK build/publish pipeline.

## Risks & Mitigations

- **Consumer drift:** Without committed generated code, services must regenerate
  or pull packages on each build. Mitigate with build steps orchestrated by CI
  and Managed Mode.
- **Plugin/runtime changes:** Buf-managed versions reduce risk, but we need
  breakage checks in CI (`buf breaking`).
- **Tooling adoption:** Document and automate the Buf workflows to avoid ad-hoc
  scripts and ensure new developers don’t regenerate with mismatched plugins.

## Success Criteria

- No generated Go/TS/Python code committed; Buf is the single source for stubs.
- `buf generate` runs cleanly with Managed Mode and Protovalidate support.
- SDK packages are published automatically and consumed without custom scripts.
- Tilt/CI pipelines stop manipulating symlinks or workspace installs; builds rely
  purely on Buf and package registries.
