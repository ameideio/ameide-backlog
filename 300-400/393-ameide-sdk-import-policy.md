# 393 – SDK Import & Proto Consumption Policy

**Status:** Draft (enforce across services)  
**Owner:** Platform DX / SDK maintainers  
**Updated:** 2025-02-21

Doc relationships: backlog/388-ameide-sdks-north-star.md (why/versioning goals), backlog/390-ameide-sdk-versioning.md and backlog/391-resilient-cd-packages-workflow.md (how versions are computed/published), backlog/389-ameide-sdk-ameideclient.md (runtime client surface), backlog/394-dockerfile-tilt.md + backlog/395-sdk-build-docker-tilt-north-star.md (how Docker/Tilt honor this policy). Script names in this doc (`sdk_policy_*`) are canonical; older names such as `enforce_no_proto_imports.sh` / `ensure_proto_workspace_usage.sh` are deprecated.

## Intent

Eliminate the constant flip-flop between “import proto types from the SDK” and “import from the generated package” by defining a single, boring policy that works for:

- Local inner loop (devcontainer, Tilt)
- CI workflows
- Docker builds
- Helm / Argo runtimes

This document is referenced from backlog/388 (SDK publishing) and backlog/373 (Argo + Tilt north star). Treat it as the canonical rulebook for where proto/schemas/clients come from in every environment.

**Scope:** Ring 1 (devcontainer/Tilt/PR CI/prod images built in-repo) uses workspace stubs + workspace AmeideClient implementations per 402/403/404/405. Ring 2 (publish smoke + external consumers) uses the published SDK artifacts. When older wording implied “published-only everywhere,” treat that as Ring 2; inner loop is workspace-first.

### Contract-first alignment

- `packages/ameide_core_proto` is our internal schema registry module. All `.proto` files land there first, so contracts evolve independently of any single service implementation.
- CI treats that package as the single source of truth: Buf generates TS/Go/Python stubs, SDK packages are rebuilt from those outputs, and tests/imports inside the monorepo resolve the generated artifacts locally. External consumers get the exact same contract via the published SDKs.
- This mirrors the “contract-first / schema-first” approach advocated by the Protobuf and gRPC ecosystem (Spartner, Medium, Red Hat Developer) and by Buf’s registry guidance: define the schema once, publish it centrally, and generate language-specific clients/servers from that source.
- **External runtime consumers + release images** use the published SDKs (`@ameideio/ameide-sdk-ts`, `ameide-sdk-go`, `ameide-sdk-python`) as the public surface. **Internal monorepo services** depend on `packages/ameide_core_proto` + the workspace AmeideClient implementations in dev/PR CI and prod builds; no registry availability is required to fail on proto drift.
- Treating `packages/ameide_core_proto` as the only schema input (no service runs `buf generate` on its own, no one reads proto output from SDK bundles) effectively gives us a Git-backed schema registry, which is the same architectural model Buf’s Schema Registry promotes.
- The workspace vs. published-artifact split (“workspace:^” ranges / `go.work` / editable installs for inner-loop dev, versioned SDKs for external consumers) reflects the common monorepo pattern of fast local iteration with workspace wiring and strict artifact pinning in the outer loop so partners/helm see what we ship.
- Conceptually this is a hexagonal/ports-and-adapters setup: `packages/ameide_core_proto` defines the port (messages + service signatures), `ameide-sdk-*` act as adapters that wrap auth/retries/transport details per language, and services/applications depend on the adapter instead of hand-rolling clients.

## Policy

1. **One proto source:** `packages/ameide_core_proto` is the only on-repo source of generated proto artifacts. When protos change, run `pnpm --filter @ameide/core-proto build` (Buf + tsc) and commit the outputs.

2. **Workspace imports for dev/tests/prod builds:** Internal services import proto namespaces from `packages/ameide_core_proto/dist/src/index.js` (TS), `packages/ameide_core_proto/gen/go/...` via `go.work` (Go), and editable installs of `packages/ameide_core_proto` (Python). That same workspace wiring is used for PR CI and prod service images so proto drift fails immediately without registry access.

3. **SDKs are runtime surfaces:** Services instantiate clients from the AmeideClient implementations (TS/Go/Python) in the workspace SDK packages; external consumers use the published SDKs. Tests that need proto-only helpers still import from `packages/ameide_core_proto`; tests that exercise RPC clients instantiate them via the AmeideClient surface.

4. **Devcontainer/Tilt installs:** `pnpm install --no-frozen-lockfile`, `go work sync && go list ./...`, and `uv sync --frozen` hydrate the workspace without touching lockfiles. All services point to `workspace:` specs for the SDK/config packages locally. The devcontainer never rewrites manifests to `npm:@…@dev`; CI handles version stamping.

5. **CI + publishing:** `.github/workflows/cd-packages.yml` computes the canonical `(BASE_VERSION, DATE, SHA)` tuple, rewrites **temp copies** of each package to that version, builds/publishes:
   - npm packages (`@ameideio/ameide-sdk-ts`, `@ameideio/configuration`)
   - Go module tags (`vX.Y.Z` or dev pseudo-version)
   - PyPI wheels (`ameide-sdk-python`)
   - GHCR images: dev → `dev` + SHA, main → `latest` + SHA, SemVer tags only on release refs

   **pnpm workspace-first pivot (clarity):** Service manifests stay on `workspace:^`/`workspace:*` for SDK/config, so monorepo installs always resolve to the local workspace. Publish uses stamped temp copies only; pnpm’s publish rewrite converts `workspace:` to semver in the packed artifacts. To validate what consumers see, CI runs an out-of-tree smoke project (`pnpm init -y && pnpm add @ameideio/ameide-sdk-ts@$NPM_VERSION && node -e "require('@ameideio/ameide-sdk-ts')"`) instead of rewriting service manifests or locks.

6. **Docker builds:** Prod/CI service images build from the repo using `packages/ameide_core_proto` + workspace SDKs (pnpm/go.work/editable uv) so proto edits break images without registry availability. Outer-loop publish smoke tests build out-of-tree images that install the published SDK artifacts to validate what external consumers see.

7. **Helm / Argo:** Charts reference the image tags produced in the publish step. Services should not mount proto directories or install extra dependencies at runtime. If a service needs a proto helper, it must come from the versioned SDK baked into the image.

## Practical Checklist

- [x] When adding a new service/module, import proto types from `packages/ameide_core_proto/dist/...`.
- [x] Workspace scripts/tests use `workspace:^` ranges for Ameide packages; lockfiles pin the last published version only in stamped temp copies (monorepo keeps workspace links).
- [x] Before publishing, run `pnpm --filter @ameide/core-proto build` and commit generated outputs. (`.github/workflows/cd-packages.yml` now runs the build and fails if `packages/ameide_core_proto/dist` drifts.)
- [x] Never import from `@ameideio/ameide-sdk-ts/proto.js`; use the proto package locally and the SDK package at runtime.
- [x] Prod Dockerfiles/Helm charts (cd-service-images) must use stamped locks + published SDKs; dev/Tilt `.dev` Dockerfiles stay workspace-first with relaxed installs.
- [x] Proto generation is centralized: only `packages/ameide_core_proto` owns `buf.gen*.yaml`; generated outputs in `packages/ameide_core_proto/dist` are committed for dev/test, and CI (`scripts/sdk_policy_enforce_proto_generation.sh`) fails if other buf.gen files appear.

## Implementation Gaps (Feb 2025 audit)

> **Go SDK decision (Nov 2025, updated per 404)**: Inner loop, PR CI, and prod service images use workspace stubs (`packages/ameide_core_proto/gen/go/...` via `go.work`) plus the workspace Go SDK; no Buf GOPROXY or `buf.build/gen` imports in services. Outer-loop publish still tags `github.com/ameideio/ameide-sdk-go` and smoke-installs it out of tree for external consumers.

1. **TypeScript proto shims now import from `packages/ameide_core_proto`.** All service barrels point at `dist/src/index.js` (graph/platform/threads/workflows/www apps) so local tests pick up proto edits immediately (`services/graph/src/proto/index.ts`, `services/platform/src/proto/index.ts`, `services/threads/src/proto/index.ts`, `services/workflows/src/proto/index.ts`, `services/www_ameide*/src/proto/index.ts`). Runtime clients still come from the SDK surface, so the original drift is resolved.

2. **Guardrails enforce the workspace-import contract.** `scripts/sdk_policy_enforce_proto_imports.sh` now fails when runtime code reaches for `@ameideio/ameide-sdk-ts/proto.js`, when barrels stop importing `packages/ameide_core_proto`, or when Dockerfiles bypass lockfiles. A companion script, `scripts/sdk_policy_check_proto_workspace_barrels.sh`, runs in CI to make sure every service’s `src/proto/index.*` continues to re-export the committed bundle. ESLint rules were updated accordingly (`eslint.config.mjs`, `services/www_ameide_platform/eslint.config.mjs`).

3. **SDK lock validation is enforced.** `scripts/sdk_policy_check_lock_hygiene.sh` now runs in `ci-core-quality` and fails if release-time Go modules omit `github.com/ameideio/ameide-sdk-go` tags or if Python services omit an exact `ameide-sdk-python==<version>` pin in `pyproject.toml` / `uv.lock`. Workspace `replace` / editable wiring is expected for Ring 1; publish smoke uses temp-stamped copies. For TypeScript, `pnpm-lock.yaml` intentionally carries `link:../../packages/...` entries in the monorepo so CI installs resolve the workspace copy; publish uses temp-stamped copies to produce the versioned tarballs that the smoke test installs.

4. **Node Dockerfiles now copy the repo lockfile before frozen installs.** Each builder stage copies `pnpm-lock.yaml` alongside `pnpm-workspace.yaml` so `pnpm install --frozen-lockfile` can enforce the stamped versions (`services/graph/Dockerfile.release:25-36`, `services/platform/Dockerfile.release:25-36`, `services/chat/Dockerfile.release:17-28`, `services/www_ameide_platform/Dockerfile.release:20-36`). The remaining TODO is ensuring CI stamps the lock prior to container builds.

5. **Go services now resolve workspace stubs + SDK in repo builds.** `go.work` maps `packages/ameide_core_proto` and `packages/ameide_sdk_go`, and `go test ./...` (dev/CI/prod images built in-repo) uses those workspace modules—no Buf GOPROXY or `buf.build/gen` imports in services. Outer-loop publish still tags `github.com/ameideio/ameide-sdk-go` and runs an out-of-tree smoke with `GOWORK=off` + tagged module for external consumers. The old workspace cache + sync script path is retired; CI guards block reintroducing replaces (`scripts/sdk_policy_check_go_replaces.sh`).

6. **Python runtimes are mostly migrated to the SDK facade.** Agents runtime (`services/agents_runtime`) now routes outbound calls through `AmeideClient` with async interceptors enabled. Workflows_runtime and workflow_worker callbacks also use SDK clients with tenant/source metadata. The main inference server still constructs raw `ameide_core_proto` stubs and custom transports, so it remains the last Python holdout.

7. **Python Dockerfiles now install the lockfile version of `ameide-sdk-python`.** Agents-runtime, inference, workflows_runtime, and workflow_worker images all run `uv sync --frozen` against their `uv.lock` and never pull ad-hoc GHCR wheels (`services/*/Dockerfile`). Each service also pins `ameide-sdk-python==2.10.1` in `pyproject.toml`, so Docker, CI, and runtimes consume the same published wheel.

8. **`@ameideio/ameide-sdk-ts/proto.js` has been removed from service code.** The lint rules now forbid it, and `rg` shows no remaining imports in service/runtime code. New regressions will fail `scripts/sdk_policy_enforce_proto_imports.sh`.

## Service & Package Checklists

The following per-package/service checklists break down the concrete activities required to implement the policy and close the gaps catalogued above. Use them during backlog refinement and PR reviews when touching a given component.

### Proto + SDK Packages

#### `packages/ameide_core_proto`

- [ ] Keep `dist/` outputs regenerated (`pnpm --filter @ameide/core-proto build`) and committed before any SDK publishing so dev/test consumers import the latest descriptors (`packages/ameide_core_proto/dist/src/index.js:1`).
- [ ] Maintain Buf generator configs (`packages/ameide_core_proto/buf.gen.yaml:1`, `packages/ameide_core_proto/buf.gen.sdk-go.yaml:1`, `packages/ameide_core_proto/buf.gen.sdk-python.yaml:1`) and bump plugin references when Buf releases breaking changes.
- [ ] Ensure Tilt/local_resource mirrors the CI steps and fails fast when Buf credentials are missing (`Tiltfile:332-373`).

#### `packages/ameide_sdk_ts`

- [x] Keep `package.json` scripts building/test running so `pnpm --filter @ameideio/ameide-sdk-ts test` is a viable guardrail before every publish (`.github/workflows/cd-packages.yml` now runs the suite before packing).
- [ ] Run `scripts/sync-proto.mjs` whenever `packages/ameide_core_proto` changes to refresh bundled descriptors and update `.proto-manifest.json`.
- [ ] Validate Node/Browser bundles continue to export `proto` and runtime helpers so services can rely on published artifacts during Docker builds.

#### `packages/ameide_sdk_go`

- [ ] Regenerate stubs via `packages/ameide_core_proto` manifests using Buf/BSR when protos change (`packages/ameide_sdk_go/go.mod:1`); no local `sdk/go` cache or sync script remains.
- [x] Keep the integration/unit test suites (`packages/ameide_sdk_go/__tests__/...`) green before cutting tags to `github.com/ameideio/ameide-sdk-go`. (`cd-packages` runs `go test ./...` in packages/ameide_sdk_go.)
- [ ] Ensure Go SDK depends on published Buf modules directly (`buf.build/gen/go/ameideio/ameide/...`); no local codegen or workspace caches are used.

#### `packages/ameide_sdk_python`

- [ ] Regenerate bundled wheels (`uv build`) after every proto change so the published `ameide-sdk-python` contains current descriptors.
- [x] Keep the retry/interceptor suites under `packages/ameide_sdk_python/__tests__` running and enforce version bumps through CI smoke (`cd-packages` runs `pytest` before packing and smoke-installs from PyPI/GHCR).
- [ ] Maintain the alias registration logic so importing `ameide_sdk` registers `ameide_core_proto` (`packages/ameide_sdk_python/src/ameide_sdk/__init__.py:1-26`).

### TypeScript / Node.js Services

#### `services/graph`

- [x] Point `src/proto/index.ts` at `../../packages/ameide_core_proto/dist/src/index.js` for dev/test imports so proto edits flow immediately (`services/graph/src/proto/index.ts:1`).
- [x] Configure Jest/TS path aliases (see `services/graph/tsconfig.json:3-18` and `services/graph/jest.config.mjs:15-22`) to resolve the workspace proto bundle and block `@ameideio/ameide-sdk-ts/proto.js` imports outside runtime surfaces.
- [ ] Keep runtime client creation (`services/graph/src/...`) on the published SDK and enforce that lint rule with a custom ESLint plugin.
- [x] Copy `pnpm-lock.yaml` into the Docker build context before each frozen install so container builds stay pinned (`services/graph/Dockerfile.release:25-52`).

#### `services/platform`

- [x] Rework `src/proto/index.ts` to reference `packages/ameide_core_proto/dist` for dev/test contexts and only export runtime clients via the SDK (`services/platform/src/proto/index.ts:1-27`).
- [x] Ensure `tsconfig.json` and Jest configs pick up the workspace proto path when `NODE_ENV=test` (`services/platform/tsconfig.json:3-18`, `services/platform/jest.config.mjs:15-22`).
- [x] Copy `pnpm-lock.yaml` into the Docker build context before running the frozen installs so container builds stay pinned (`services/platform/Dockerfile.release:25-46`).

#### `services/threads`

- [x] Replace the `@ameideio/ameide-sdk-ts` proto exports used in `src/proto/index.ts` with direct imports from `packages/ameide_core_proto/dist` for tests/mocks (`services/threads/src/proto/index.ts:1`).
- [x] Keep the runtime Connect clients (`src/proto/clients.ts`) on the SDK to benefit from shared interceptors (`services/threads/src/proto/clients.ts:1-45`).
- [x] Copy `pnpm-lock.yaml` into the Docker build context before running the frozen installs so container builds stay pinned (`services/threads/Dockerfile.release:20-38`).

#### `services/chat`

- [x] Update server code to build request/response proto types from the workspace package during tests, while continuing to dial runtime clients via the SDK (`services/chat/src/proto/index.ts:1-4`, `services/chat/src/server.ts:1-30`).
- [x] Ensure Jest/unit tests reference the workspace proto tree instead of `@ameideio/ameide-sdk-ts/proto.js` (`rg` shows no imports under `services/chat/__tests__`).
- [x] Copy `pnpm-lock.yaml` into the Docker build context so the existing `--frozen-lockfile` installs actually pin versions (`services/chat/Dockerfile.release:17-36`).

#### `services/repository`

- [x] Point all test-only imports (e.g., `src/__tests__/unit/**/*.ts`) at `packages/ameide_core_proto/dist/src/index.js` instead of the SDK (`services/repository/src/proto/index.ts:1`, `services/repository/src/__tests__/unit/metamodel-store.test.ts:1`).
- [ ] Ensure runtime code creates clients via `@ameideio/ameide-sdk-ts` but never reaches into `_proto.js`.
- [x] Copy `pnpm-lock.yaml` into the Docker build context so the frozen installs in both builder passes stay pinned (`services/repository/Dockerfile.release:23-44`).

#### `services/transformation`

- [x] Mirror the proto import swap used in other services so unit tests hit the workspace package (`services/transformation/src/proto/index.ts:1-15`).
- [x] Keep runtime client creation on the SDK and add lint guards to block proto-only imports outside tests (`services/transformation/src/proto/clients.ts:1-126`, `eslint.config.mjs`).
- [x] Copy `pnpm-lock.yaml` into the Docker build context so the frozen installs stay pinned (`services/transformation/Dockerfile.release:25-49`).

#### `services/workflows`

- [x] Repoint `services/workflows/src/proto/index.ts` and helper enums to import dev/test proto types from the workspace package (`services/workflows/src/proto/index.ts:1-10`, `services/workflows/src/proto/enums.ts:1-9`).
- [x] Ensure runtime clients (`services/workflows/src/proto/clients.ts:1-64`) use the published SDK and exercise them in integration tests.
- [x] Copy `pnpm-lock.yaml` into the Docker build context so the frozen installs stay pinned (`services/workflows/Dockerfile.release:25-47`).

#### `services/www_ameide`

- [x] Modify `src/proto/index.ts` to consume `packages/ameide_core_proto/dist` during development while preserving SDK-based runtime hooks for production builds (`services/www_ameide/src/proto/index.ts:1-30`).
- [x] Update lint rules to allow importing the workspace proto shim inside the app layer and forbid `@ameideio/ameide-sdk-ts/proto.js` (`services/www_ameide/eslint.config.mjs:41-63`).
- [x] Copy `pnpm-lock.yaml` into both the deps and builder stages so the frozen installs stay pinned (`services/www_ameide/Dockerfile.release:18-54`).

#### `services/www_ameide_platform`

- [x] Replace the proto imports in `src/proto/index.ts` and `lib/sdk/**` with references to `packages/ameide_core_proto/dist` for dev/test helpers, keeping runtime API routes on SDK clients (`services/www_ameide_platform/src/proto/index.ts:1-33`, `services/www_ameide_platform/lib/sdk/element-service.ts:1-37`).
- [x] Rework the ESLint rule that blocks `@buf/...` imports so it permits the workspace proto barrel but continues to warn on direct Buf registry references (`services/www_ameide_platform/eslint.config.mjs:45-82`).
- [x] Copy `pnpm-lock.yaml` into both the deps and builder stages so the frozen installs use the stamped versions (`services/www_ameide_platform/Dockerfile.release:18-66`).
- [x] Update any API routes that still import proto schemas from `@ameideio/ameide-sdk-ts/proto.js` (`services/www_ameide_platform/app/api/chat/stream/route.ts:1-60` now imports from '@/src/proto').

### Go Services

#### `services/agents`

- [x] Remove the local `replace github.com/ameideio/ameide-sdk-go => <workspace cache>` override once the module is published, and depend on tagged versions instead (`services/agents/go.mod:5-23`).
- [x] Update the Dockerfile to fetch the published SDK artifact during build instead of copying `/sdk/go` from the workspace so CI/Docker prove the tagged module works (`services/agents/Dockerfile.release:5-70`).
- [x] Keep tests importing only from the SDK module and never from `packages/ameide_core_proto` (`services/agents/internal/service/service.go:10`).

#### `services/inference_gateway`

- [x] Remove local replaces, require published tags, and ensure `go.mod` references the public module path (`services/inference_gateway/go.mod:5-20`).
- [x] Update the Dockerfile to fetch the published SDK artifact instead of reusing `/sdk/go` from the workspace so builds match the release flow (`services/inference_gateway/Dockerfile.release:5-69`).
- [x] Maintain integration tests so they run against the downloaded SDK instead of workspace copies (`services/inference_gateway/__tests__/integration/gateway_integration_test.go:13-24`).

### Python Services

#### `services/agents_runtime`

- [x] Stop importing `ameide_core_proto` directly inside the runtime service and switch to the published SDK clients for RPC calls (`services/agents_runtime/src/agents_service/sdk_clients.py:1-66`, `services/agents_runtime/src/agents_service/registry.py:23-145`).
- [x] Keep the `ameide_sdk` import that registers proto aliases but route all client construction through SDK helpers.
- [x] Update the Dockerfile to install the exact wheel version specified in `uv.lock` rather than an arbitrary `PYTHON_SDK_REF` (`services/agents_runtime/Dockerfile:1-73`).

#### `services/inference`

- [ ] Replace imports from `ameide_core_proto` in `app.py` and helper modules with SDK-provided stubs so retry/interceptor logic is shared (`services/inference/app.py:80-105`).
- [x] Ensure `pyproject.toml` pins `ameide-sdk-python==<version>` once lockfiles are stamped, and drop editable/path references (`services/inference/pyproject.toml:17`).
- [x] Update `services/inference/Dockerfile` to install dependencies via `uv sync --frozen` and then install the locked SDK wheel (`services/inference/Dockerfile:22-58`).

#### `services/workflows_runtime`

- [x] Switch callback clients and activities to use the SDK wrappers rather than creating gRPC channels manually (`services/workflows_runtime/src/workflows_runtime/callbacks.py:8-88`).
- [x] Maintain the `ameide_sdk` import for alias registration but audit the rest of the package for remaining direct `ameide_core_proto` imports (`services/workflows_runtime/src/workflows_runtime/__init__.py:5`).
- [x] Align the Dockerfile’s `uv sync` step with the pinned SDK version just like agents runtime (`services/workflows_runtime/Dockerfile:14-52`).

#### `services/workflow_worker`

- [x] Ensure worker callbacks and clients leverage the SDK wrappers to avoid duplicating transport logic and proto imports (`services/workflow_worker/src/workflow_worker/callbacks.py:9-72`).
- [x] Update `pyproject.toml` and `uv.lock` to depend on the published wheel and strip any editable `packages/ameide_sdk_python` references (`services/workflow_worker/pyproject.toml:13`).
- [x] Mirror the Dockerfile changes applied to agents runtime/workflows_runtime so builds install the pinned SDK artifact (`services/workflow_worker/Dockerfile:20-52`).

## Enforcement & Control Script Upgrades

- [x] **Invert the proto guard script:** `scripts/sdk_policy_enforce_proto_imports.sh` now fails on `@ameideio/ameide-sdk-ts/proto.js`, bans runtime imports from `packages/ameide_core_proto`, and greps Dockerfiles for `--no-frozen-lockfile`. It runs in the core CI workflow (`ci-core-quality`).
- [x] **Update ESLint/shared lint rules:** Both the repo-level config and the Next.js app config forbid importing `@ameideio/ameide-sdk-ts/proto.js` while allowing the workspace proto barrel for dev/test contexts (`eslint.config.mjs`, `services/www_ameide_platform/eslint.config.mjs`).
- [x] **Add dependency/lockfile validators:** `scripts/sdk_policy_check_lock_hygiene.sh` runs in CI to ensure Go modules reference the published `github.com/ameideio/ameide-sdk-go` module (no workspace replaces) and Python lockfiles pin the published `ameide-sdk-python`. (TypeScript lockfile work remains; see Implementation Gap #3.)
- [x] **Tighten Docker/Tilt guardrails:** `scripts/sdk_policy_check_docker_sources.sh` and the updated Tilt resources detect `SKIP_SDK_SYNC=1`, workspace SDK copies, or installs without `--frozen-lockfile`, blocking images that drift from stamped artifacts.
- [x] **Add published-artifact smoketests:** After `release/tilt_core_sdk_{ts,go,python}.sh`, add downstream jobs that `pnpm add`, `go get`, and `pip install` the freshly published SDK versions and run lightweight imports (`require.resolve`, `go list`, `python -c 'import ameide_sdk'`). `.github/workflows/cd-packages.yml` now installs the published npm packages (GitHub + npmjs), `go get`s the module, and `pip install`s from PyPI/GHCR to ensure artifacts match Docker inputs.
- [x] **Positive proto-usage audit:** `scripts/sdk_policy_check_proto_workspace_barrels.sh` walks every `services/*/src/proto/index.*` and fails CI if barrels stop importing `packages/ameide_core_proto/dist/src/index.js`.
- [ ] **Option A Go guardrails:** `scripts/sdk_policy_check_go_option_a.sh` blocks reintroducing the legacy workspace cache or sync script, and `scripts/sdk_policy_check_docker_sources.sh` now enforces Buf-first `GOPROXY` + `GOWORK=off` in Go Dockerfiles.

## References

- backlog/388-ameideio-sdks-north-star.md – SemVer and publish workflow details.
- backlog/373-argo-tilt-helm-north-start-v4.md – Devcontainer/Tilt/Argo alignment; this policy underpins the “prod-first, Tilt-second” contract.
- Buf blog/docs – schema-first workflow guidance for Protobuf APIs.
- googleapis/googleapis + Google Cloud client libraries – canonical contract-first + vendor SDK example.
- Thoughtworks / HappyCoders – hexagonal (ports & adapters) architecture references reinforcing the “proto = port, SDK = adapter” model.
