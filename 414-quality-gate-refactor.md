# 414 – Unified Quality Gate & Policy Layout

## Objective
Document the consolidated structure for policy guardrails, local/CI runners, and the end-to-end quality checks that gate merges, images, and packages.

## Current Structure
- **Entrypoint runner:** `scripts/policy/run_all.sh`
  - Default channel: `workspace` (set `SDK_POLICY_CHANNEL=release` for release semantics).
  - Runs all policy guardrails and reports aggregated status (no early exit).
- **Policy scripts (under `scripts/policy/`):**
  - `enforce_proto_generation.sh` — only `packages/ameide_core_proto` may contain `buf.gen*.yaml`.
  - `check_proto_source.sh` — `proto-source.json` matches sync script defaults (TS/Go/Py).
  - `enforce_proto_imports.sh` — enforces SDK-only proto usage; blocks direct workspace/BSR stubs; scans built artifacts and Dockerfiles for frozen-lock installs.
  - `check_proto_workspace_barrels.sh` — service proto barrels must import `@ameideio/ameide-sdk-ts/proto.js`.
  - `check_lock_hygiene.sh` — Go/Py/TS lock/import hygiene (no BSR stubs, required replaces, no PyPI pins).
  - `check_docker_sources.sh` — Dockerfiles must copy workspace SDKs; no registry fetches/core_proto copies; no SKIP_SDK_SYNC.
  - `check_go_option_a.sh` — blocks legacy sdk/go cache or sync scripts.
  - `check_go_replaces.sh` — in `release` channel forbids ameide-sdk-go replaces; `workspace` channel logs expected replaces.
  - `enforce_build_secrets.sh` — BuildKit secrets required (no ARG/ENV tokens).
  - TS helpers in `scripts/policy/ts/`: `rewrite_lock.sh`, `rewrite_manifests.py` for publish-time rewrites.
- **Local/CI test gate:** `ameide ci test` runs the 430v2 contract (Phase 0/1/2) and emits JUnit evidence under `artifacts/agent-ci/`.
- **Docs:** `docs/quality-checks.md` summarizes the structure and entrypoints.

## Workflows using the new runner
- `.github/workflows/ci-core-quality.yml` — runs `scripts/policy/run_all.sh` plus lint/typecheck/tests (JS/TS, Ruff, mypy (soft), Jest suites, Go/Py tests).
- `.github/workflows/cd-packages.yml` — calls `scripts/policy/run_all.sh` before SDK build/test/publish.
- `.github/workflows/cd-service-images.yml` — calls `scripts/policy/run_all.sh` before image builds.

## What we test end to end (Core Quality Gate)
1) **Policy guardrails:** proto generation, proto source alignment, SDK-only imports, barrel checks, lock hygiene, Docker SDK source enforcement, Go Option A, build secrets.
2) **Linting:** ESLint (workspace fan-out), Ruff.
3) **Type checks:** `pnpm typecheck`; `uvx mypy --config-file mypy.ini packages` (continues-on-error as in CI).
4) **JS tests:** `services/www_ameide_platform` unit + integration; `packages/ameide_sdk_ts` unit + integration with coverage.
5) **Python tests:** inference SDK integration (`services/inference/__tests__/integration/test_sdk_metadata_retry.py`); package-wide pytest with coverage.
6) **Go tests:** `packages/ameide_sdk_go` (excluding generated `/gen/`).
7) **Helm lint:** `scripts/infra/lint-image-values.py` (skips known third_party chart defaults).

## Channel behavior
- `SDK_POLICY_CHANNEL=workspace` (default): allows expected workspace replaces; skips some release-only checks (e.g., ameide_sdk.generated in Python).
- `SDK_POLICY_CHANNEL=release`: stricter (no Go replaces; full proto import enforcement).

## Next Steps / Follow-ups
- Decide if release channel should be run in CI pre-merge or only on publish.
- Optionally add a short CLI wrapper (`bin/core-quality`) that runs the local gate with common flags (e.g., `--skip-helm-lint`).
