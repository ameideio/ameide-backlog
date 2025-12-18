# 403 – Python Buf-first migration plan

**Status:** Draft  
**Intent:** Apply the buf-first/client-gen model from 402 to every Python service so proto edits break builds immediately (workspace stubs) across dev/CI/prod; SDKs still publish for external consumers but Ameide images build from repo protos/SDK code. One `uv.lock` stays canonical; CI/dev/prod choose workspace editables vs frozen wheel via install mode, not by carrying two locks. Version pins track the tag authority from backlog/390.

## Context
- 402 requires proto-first for all rings (dev/CI/prod). Prod images must build from the repo proto outputs + SDK code, not from published wheels.
- Python services pin `ameide-sdk-python==2.10.1` in `pyproject.toml`/`uv.lock`; guardrails enforce PyPI-only. That delays “proto change → service break” until a publish.
- Scope: `services/inference`, `services/agents_runtime`, `services/workflows_runtime`, `services/workflow_worker`.
- Gaps: inference still builds stubs/transports manually; dev/Tilt/CI/prod install the published wheel instead of the workspace SDK; no guard that buf regen ran before Python tests.
- **Same imports, different source**: Python code should be identical across rings; Ring 1 resolves editable workspace proto/SDK, Ring 2 uses the pinned wheel version in `uv.lock`.

**Naming:** when this doc says “dev/Tilt Dockerfile”, it means `Dockerfile.dev` (Ring 1, workspace stubs/SDK). “Prod/CI Dockerfile” refers to the plain `Dockerfile` (Ring 2, stamped lockfile + published wheel).

## Plan

**Phase 0 – Prereqs / plumbing**
- Add workspace sources for Python in `tool.uv.sources` (or env toggle) so `uv sync` installs `packages/ameide_sdk_python` (with generated stubs) editable for all modes (dev/CI/prod); services do not import `packages/ameide_core_proto` directly.
- Update `scripts/ci/prepare_python_env.sh` and Tilt Python resources to run the editable installs after `pnpm --filter @ameide/core-proto build` so regenerated stubs are what tests see.
- Adjust `scripts/policy/check_lock_hygiene.sh` to allow/expect workspace sources (drop PyPI-only enforcement); tighten drift checks instead.
- Ensure `packages/ameide_core_proto/buf.gen.sdk-python.yaml` regeneration is part of the standard proto build (emits to `packages/ameide_sdk_python/src/ameide_sdk/_proto`, followed by import rewrites in `packages/ameide_sdk_python/scripts/postprocess_generated.py`); add a CI check that fails if manifests drift.

**Phase 1 – CI/build flow**
- `ci-core-quality` (dev + PR CI): run `pnpm --filter @ameide/core-proto build`, install editable `ameide_sdk_python`, and execute a smoke pytest over SDK interceptors to prove the workspace build is used (no PyPI required).
- Add a guard that a proto diff without regenerated Python outputs fails CI (reuse `scripts/dev/check_sdk_alignment.py` or a Python-specific variant).
- For release workflows (`cd-packages`, `cd-service-images`), build images against the workspace proto/SDK (copied into the build context). Still publish and smoke-install the wheel for external consumers, but Ameide images do not depend on it.

**Phase 2 – Service migrations (single path: proto → workspace SDK → image)**
- `services/agents_runtime`  
  - Switch `pyproject.toml` to workspace sources for `ameide-sdk-python` only.  
  - Update Dockerfile.dev/prod/Tilt to copy `packages/ameide_sdk_python` and install editable after sync (no PyPI stamping).  
  - Add an SDK-backed metadata/auth/retry/timeout integration test that asserts headers and deadlines survive through the AmeideClient wrappers.
- `services/workflows_runtime` and `services/workflow_worker`  
  - Same dependency swap + Dockerfile changes; keep Temporal callbacks on the SDK clients.  
  - Add coverage that callback clients still inject tenant/source metadata via AmeideClient interceptors.
- `services/inference`  
  - Replace direct `grpc_aio.server` + raw stubs usage with AmeideClient-based clients (reuse retry/metadata/tracing helpers from `ameide_sdk`).  
  - Wire SDK config (addr/auth/tenant/request-id/timeouts) from env; add integration tests for headers/retries/timeouts/tracing via the SDK.  
  - Ensure proto imports come from `ameide_sdk.proto.ameide_core_proto.*` so proto edits break tests without a publish; SDK stubs are synced from the BSR module (not generated in services).

**Phase 3 – Locks & artifacts**
- Single path: `uv.lock` records workspace sources for Ameide SDK; no separate release lock. Ameide images build from repo sources.
- Update `scripts/policy/check_docker_sources.sh` to permit/expect workspace SDK copies in Dockerfiles and fail on core_proto copies.
- Keep release provenance for external consumers: `cd-packages` still stamps/labels `ameide-sdk-python`, builds the wheel, and smoke-installs it out-of-tree, but Ameide services/images do not depend on the published wheel.

**Phase 4 – Acceptance gates**
- Proto change breaks Python service tests without publishing an SDK.
- Devcontainer/Tilt/prod images all use workspace proto/SDK sources.
- CI enforces regenerated Python outputs and rejects stale manifests.
- SDK parity tests (metadata/auth/retry/timeout/tracing/request-id) run in CI for all Python services and the SDK itself.

## Progress (2025-11-24)
- [x] CI scripts install workspace proto/SDK (editable) after `pnpm --filter @ameide/core-proto build` (`scripts/ci/install_python_workspace_sdks.sh`, `scripts/ci/prepare_python_env.sh`, wired into `ci-core-quality`).
- [x] Guardrails updated to allow/expect workspace proto/SDK in Docker/locks (`scripts/policy/check_lock_hygiene.sh`, `scripts/policy/check_docker_sources.sh`).
- [x] Agents runtime switched to proto-first across dev/prod: `pyproject.toml` now uses `tool.uv.sources` for `ameide-sdk-python`; Dockerfiles copy `packages/ameide_sdk_python` and drop SDK stamping; `uv.lock` refreshed to workspace SDK.
- [x] Workflows_runtime and workflow_worker now install workspace proto/SDK (uv sources + editable) in pyproject/Dockerfiles; Temporal callbacks still run through AmeideClient.
- [x] Inference pyproject/Dockerfile now consume workspace proto/SDK (uv sources + editable install) so proto changes break builds; service still uses raw stubs and needs AmeideClient + header/retry/tracing coverage.
- [ ] Add proto/SDK drift check before Python tests (fail CI on unstamped regen).
- [ ] Complete inference refactor onto AmeideClient and add SDK header/retry/timeout/tracing tests.

## Rollout order
1) Land plumbing: uv source overrides + CI job to regen/install workspace SDK.  
2) Update guardrail scripts (lock hygiene, docker source) for the single-path model.  
3) Migrate agents_runtime + workflows_runtime/workflow_worker to workspace dependencies and add their SDK header/retry tests.  
4) Refactor inference onto AmeideClient + add SDK integration tests.  
5) Add proto/manifest drift enforcement to `cd-packages`/`cd-service-images` and flip CI to fail on proto drift.
