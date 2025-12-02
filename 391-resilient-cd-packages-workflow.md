# 391 - Resilient CD / Packages Workflow

Purpose/relationships: This workflow operationalizes backlog/388-ameide-sdks-north-star.md and backlog/390-ameide-sdk-versioning.md. It is the source of truth for publishing SDK/config artifacts, and its outputs are consumed by `cd-service-images` (see backlog/395-sdk-build-docker-tilt-north-star.md). Guardrails rely on the `scripts/sdk_policy_*` checks; older script names (e.g., `enforce_no_proto_imports.sh`) are deprecated.

## Context
- CD / Packages had recurring flakes:
  - `pnpm init -y` option dropped in pnpm >=10 caused smoke tests to fail.
  - Transient 5xx during Syft/Cosign downloads aborted runs.
  - Smoke install fetched Buf artifacts from npmjs.org leading to 404s.
  - Release safety net missing (could publish older tag while a newer one existed).

## Decisions
- Harden the reusable workflow instead of adding per-run overrides.
- Use Git tag authority for versioning (from backlog/390) and enforce guardrails inside workflow.
- Prefer local artifacts/OCI pulls instead of registry lookups during smoke tests.

## Changes Implemented
1. **Versioning Guardrails**
   - `Compute SDK versions` now records latest semver tag, fails if trying to publish a release older than that, and emits npm/PyPI/Go versions as artifacts for traceability.
   - Service images reuse these outputs to tag GHCR images (`cd-service-images.yml`).
2. **Download Resilience**
   - Shared `download_with_retry` helper wraps ORAS, Syft, and Cosign downloads, avoiding transient HTTP 503s.
   - CI workflows improved Helm repo add/update to retry rather than aborting.
3. **Smoke Test Hardening**
   - TypeScript smoke install pulls the GHCR tarball, runs `pnpm init` non-interactively with stdout suppressed (no Broken pipe), and installs the local `.tgz` via `pnpm add file:${TS_TGZ}` to avoid hitting public registries.
   - Ensures Buf-generated deps don't require npmjs.com access.
4. **Workflow Consumers Alignment**
   - `cd-service-images` waits for `publish_sdks`, enforces release builds run from tags, and tags images by channel: dev → `dev` + SHA, main → `latest` + SHA, and SemVer only on tag refs (see backlog/395-sdk-build-docker-tilt-north-star.md for the image tiering model).
   - `cd-promote-release` uses a helper to poll ACR for digests + official Trivy action.
   - `ci-core-quality` and `ci-helm` adopt retry-friendly tooling (uvx Ruff, Helm repo retry).
5. **Reusable outputs & private registry auth**
   - `cd-packages` exposes the computed channel/base-version outputs to workflow callers so `cd-service-images` can enforce release/tag invariants without re-running version guards.
   - All pnpm-based CI entrypoints (`ci-integration-packs`, `ci-proto`) export `PKG_GITHUB_COM_TOKEN`, falling back to `github.token` when repo secrets are unavailable so PR installs of `@ameideio/*` dependencies no longer rely on manual env injection.

## Follow-ups
- Document retry helpers in `.github/workflows/README.md` for future reuse.
- Extend Buf registry overrides to other pnpm commands if we start installing Buf artifacts elsewhere.
- Monitor the next `CD / Packages` run to confirm the smoke test remains green and update backlog/390 once consumers finish aligning.

## Local validation status (Nov 2025)
- `pnpm lint` / `pnpm typecheck` currently fail inside `services/repository` because the workspace installs `@ameideio/ameide-sdk-ts` from npm.pkg.github.com without a compiled `dist/` directory. Consumers running with `moduleResolution: NodeNext` cannot resolve `@ameideio/ameide-sdk-ts/proto.js`, so every suite that touches `lib/sdk/*` explodes (`services/www_ameide_platform/...`). We either need to link the workspace package (override `link-workspace-packages`) or publish artifacts that include the compiled output.
- Python quality gates fail locally: `uvx ruff check ...` flags unsorted imports + legacy typing (`packages/ameide_core_platform_common/...`, `services/inference/...`), `mypy --config-file mypy.ini packages` requires `types-PyYAML` and stricter typing in `packages/ameide_core_platform_common/testing.py`, and `pytest -q packages ...` cannot import `ameide_core_proto.common` because the proto package is not installed in the synced venv.
- Until the SDK/package linkage and Python typing issues are resolved, we cannot produce `.coverage/www_ameide_platform` or `.coverage/ameide_sdk_ts` artifacts locally, so CI will continue to report “No coverage files found” for Codecov even though the underlying workflows are wired for Buf auth.
