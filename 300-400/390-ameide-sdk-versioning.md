# 390 - Repository Versioning Alignment

Scope/relationships: This note is the implementation detail for backlog/388-ameide-sdks-north-star.md. It defines how CI computes versions for all SDK/config artifacts and exposes them to workflows. `cd-packages` in backlog/391-resilient-cd-packages-workflow.md enforces it, and `cd-service-images`/backlog/395-sdk-build-docker-tilt-north-star.md consume the outputs to build service images. **Inner-loop/PR CI for services stays workspace-first (core_proto + AmeideClient) per backlog/402/403/404; this doc only covers the outer-loop publish math.**

**Ring map:** Ring 1 (dev/Tilt/PR CI/prod images built in-repo) uses workspace stubs + workspace SDKs; Ring 2 (publish smoke + external consumers) uses the stamped versions computed here (`NPM_VERSION`/`PYPI_VERSION`/`GO_VERSION`).

## Current state
- **Publish Packages workflow**
  - Source of truth: Git tag `vX.Y.Z` for releases; latest semver tag for dev builds with `-dev.<timestamp>+g<sha>` suffix. **Option A:** Go modules are published on tags only; dev builds reuse the latest released tag for module consumers instead of emitting pseudo-versions.
  - A single `versioning` step derives `BASE_VERSION`, `NPM_VERSION`, `PYPI_VERSION`, `GO_VERSION`, `NPM_DIST_TAG`, `GHCR_EXTRA_TAGS`.
  - TS: packs from a temp copy with `PKG_VERSION`; repo `package.json` is untouched.
  - Python: copies the SDK to a temp dir, stamps `pyproject.toml` with `PYPI_VERSION`, and builds/publishes via `uv`; repo `pyproject.toml` stays at `0.0.0`.
  - Go: tar+SBOM uses `GO_VERSION` for tagged releases only; dev channel reuses the latest tag (no new Go module publishes on dev builds).
  - GHCR artifacts: `sdks`, `ts`, `go`, `python`, `python-sdist`, `configuration` all tag off the computed versions, with extra tags derived from tag vs dev channel.
  - Npm publish skips if the exact version already exists per registry (prevents 409 on re-runs).
- **Other versioning islands** (not yet aligned to tag authority):
  - Python package setup no longer reads a committed `VERSION`; version is injected into a temp copy of `pyproject.toml`.
  - Service images and binaries derive versions via `build/scripts/version.sh` (git describe/pseudo-versions), Helm chart `AppVersion`, and `SERVICE_VERSION` envs in telemetry initializers.
  - Go/TS/Python Dockerfiles still mix workspace overlays in some cases; goal is “pure” package manager consumption (no `go.work` in CI/Docker, no workspace SDK copies in prod images, `uv sync --frozen` everywhere).

## Decision (pure package-manager approach)
- Version authority for SDK publishes is the Git tag (release) or latest semver tag + dev suffix (non-tag). No root `VERSION` file. Go dev channel no longer emits unpublished pseudo-versions; only tagged releases produce Go modules/archives (supersedes the earlier dev pseudo-version idea in backlog/388).
- Keep repo clean during packaging by mutating temp copies only.
- **TypeScript (“pure pnpm”)**: `workspace:` for inner-loop, rely on `pnpm pack/publish` to rewrite to semver in the tarball; `pnpm-lock.yaml` + `--frozen-lockfile` (or `pnpm deploy`) back container builds; no manual manifest rewrites.
- **Go (“workspace inner loop, tagged outer loop”)**: dev/PR CI and prod images built in-repo use `go.work` to consume `packages/ameide_core_proto` + `packages/ameide_sdk_go` (no Buf GOPROXY/`buf.build` imports). Outer-loop publish/tag smoke tests run with `GOWORK=off` + `go get github.com/ameideio/ameide-sdk-go@<tag>` for external consumers; no `replace` or sync scripts allowed.
- **Python (“pure uv + PyPI”)**: SDK built from a stamped copy of `pyproject.toml`; services pin `ameide-sdk-python==<semver>` in `pyproject.toml`/`uv.lock`; CI/Docker install via `uv sync --frozen` only (PEP 440 versions for release/dev).
- Services keep `workspace:*`/`workspace:^` on SDK/config deps for inner-loop; publish steps stamp temp copies with `NPM_VERSION`/`PYPI_VERSION`/`GO_VERSION` and rely on pnpm’s publish-time `workspace:`→semver rewrite. CI installs remain workspace-local; a separate smoke project installs `@ameideio/ameide-sdk-ts@$NPM_VERSION` from the registry to validate the published artifact.

## Actions proposed
1) **Document** the tag-driven source of truth for SDKs (this note) and reference in publish workflow README.
2) **Align consumers**:
   - Go: dev/PR CI + prod images built in-repo use `go.work` (workspace stubs/SDK, `GOWORK=auto`); publish smoke uses `GOWORK=off` with `go get github.com/ameideio/ameide-sdk-go@<tag>` for external consumers; no sync shim or replaces.
   - Python: stamp a temp copy of `pyproject.toml` with the workflow-injected value and build/publish via `uv`; services pin `ameide-sdk-python==<version>` and install with `uv sync --frozen`.
   - TypeScript: keep `workspace:` for inner-loop, rely on `pnpm pack/publish` to rewrite to semver in tarballs; Docker/CI use `pnpm-lock.yaml` + `--frozen-lockfile`/`pnpm deploy`.
   - Review service image versioning (`build/scripts/version.sh`, Helm chart appVersions) to optionally consume the same tag-derived base when building release images.
3) **Guardrails**:
   - Add a check in Publish Packages to fail if a tag release is attempted while a newer semver tag already exists (prevents accidental rollback).
   - Emit the resolved versions as artifacts (`ts-sdk-version.txt` already exists; consider adding go/python) for traceability.
   - Keep `scripts/sdk_policy_check_lock_hygiene.sh` wired into CI so `go.mod`/`pyproject.toml` lockfiles never drift back to workspace overrides between publishes.
   - Add an out-of-tree pnpm smoke test job that runs `pnpm init -y && pnpm add @ameideio/ameide-sdk-ts@$NPM_VERSION && node -e "require('@ameideio/ameide-sdk-ts')"` to validate the packed artifact without changing workspace manifests or locks.
- `cd-packages` in backlog/391-resilient-cd-packages-workflow.md is the concrete workflow that implements these guardrails; `cd-service-images` (see backlog/395-sdk-build-docker-tilt-north-star.md) consumes the emitted versions when building service images.

## Open questions
- Should Go/Python local dev builds also derive from `git describe --tags` instead of committed VERSION placeholders?
- Do we want Helm chart `appVersion` to mirror SDK release tags or maintain independent lifecycle?
- How to propagate the tag-derived version into service `SERVICE_VERSION` envs for telemetry/OTEL consistency?

## Reality check (Nov 2025)
- Go SDK module exists at `github.com/ameideio/ameide-sdk-go` tagged `v0.1.0`; GHCR carries `v0.1.0`, `0.1`, `latest`. Services pin `v0.1.0` without replaces; CI needs `GOPRIVATE/GONOSUMDB=github.com/ameideio` and a PAT for go get. CI guard added to fail on replaces.
- TypeScript and configuration dev builds were published to npmjs manually (`0.1.15-dev...`); GHCR still lacks SemVer/alias tags and the `cd-packages` workflow fails before publish completes.
- Python SDK is on PyPI (e.g., `2.10.1` plus dev builds); GHCR mirrors (`ameide-sdk-python`/`-sdist`) still only have `dev`/hash tags.
- GHCR SemVer tagging remains missing for TS/Python/Configuration until `cd-packages` runs clean; npm token warnings still surface in CI.
