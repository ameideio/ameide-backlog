---
title: 406 - Rings refactor: workspace-first Ring 2
status: draft
owners:
  - platform
created: 2025-11-27
---

## Intent

- Keep 407's objective: single proto surface via SDK (TS/Go/Python), no services or images import workspace `ameide_core_proto` or dist bundles directly.
- Buf emits all stubs (including buf/validate) directly into the SDK trees.
- Ring 1 and Ring 2 stay workspace-first by using SDK code from the repo (not registries); Ring 2 adds stricter locking for third-party deps. Service-image workflow now selects Dockerfile.dev on dev branch and Dockerfile.release on main/tags.
- Published SDKs are only for external consumers and out-of-tree smoke/validation jobs.

---

## 1. Updated ring definitions

**Ring 0 - Schema (unchanged)**

- `packages/ameide_core_proto` remains the single schema module.
- Buf runs here and writes TS/Go/Python stubs (including buf/validate) directly into the SDK packages. Consumers import via SDK barrels; no runtime path should point at `packages/ameide_core_proto`.

**Ring 1 - Inner loop (dev / PR CI / Tilt)**

- Who: dev/Tilt containers (`Dockerfile.dev`), PR CI jobs, local dev.
- Inputs:
  - Workspace SDK packages (`packages/ameide_sdk_ts`, `packages/ameide_sdk_go`, `packages/ameide_sdk_python`) that already contain Buf outputs.
  - Looser dependency strictness (TS `pnpm install --no-frozen-lockfile`; Go `GOWORK=auto`; Python `uv sync` + editable via `tool.uv.sources`).
- No registry access required, and no runtime/Dockerfile references to `packages/ameide_core_proto`.

**Ring 2 - Workspace-first release images**

- Who: all staging/prod service images (`Dockerfile.release`) and prod-like internal images.
- Inputs:
  - Workspace SDK packages with generated stubs (no direct `ameide_core_proto` copies).
  - Stricter dependency treatment for non-Ameide deps:
    - TS: frozen `pnpm-lock.yaml` for third-party deps; Ameide packages stay `workspace:*`.
    - Go: vendored or copied workspace SDK module; `GOWORK=off` ok as long as SDK code is from the repo, not the proxy.
    - Python: `uv.lock` + `tool.uv.sources`; `uv sync --frozen` for third-party deps.
- No Ameide packages are installed from registries in Ring 2 service images; Dockerfiles copy SDK packages only (not `packages/ameide_core_proto`).

**SDK publish & smoke path (outer loop)**

- `cd-packages` builds/publishes SDK artifacts and runs out-of-tree smoke tests that install them from registries. This path is separate from Ring 2 service images.

---

## 2. Current inconsistencies this fixes

### 2.1 TypeScript

- 403/405 describe Ring 2 as installing published SDKs and allow dist imports from `packages/ameide_core_proto/dist`, conflicting with the SDK-only surface.
- Buf templates omit buf/validate output into the SDK, so the SDK surface is incomplete.
- Service-image workflow now uses `Dockerfile.dev` for dev branch; ensure dev Dockerfiles still copy SDKs and not core_proto.

### 2.2 Go

- 405 still stamps `GO_SDK_VERSION` and fetches SDK/core_proto from the proxy for Ring 2 instead of using workspace SDK sources.
- `go.work` and docs point services at `packages/ameide_core_proto` paths, not SDK-generated stubs.

### 2.3 Python

- 403-Python is close, but CI prep aliases `ameide_core_proto` into `ameide_sdk.generated`, and 405 still calls for PyPI wheels in Ring 2.

This refactor aligns all languages with 407: services only see proto through SDKs; Ring 2 uses workspace SDK code (not registries); publishes are isolated to the SDK product track.

---

## 3. Ring-aligned behaviour per language

### 3.1 TypeScript

**Ring 1 (`Dockerfile.dev`) - SDK-only surface**

- Services import types/clients from workspace `@ameideio/ameide-sdk-ts` barrels; dist artifacts must not reference `packages/ameide_core_proto/dist`.
- Buf/validate outputs are generated into `packages/ameide_sdk_ts` and checked in for dev/CI use.

**Ring 2 (`Dockerfile.release`) - workspace SDK, no core_proto copies**

- `pnpm-lock.yaml` is frozen for third-party deps and resolves `@ameideio/ameide-sdk-ts` via `workspace:*`; there should be no `@ameide/core-proto` dependency in service code.
- Dockerfile copies `packages/ameide_sdk_ts` (with generated stubs) into the context; it must not copy `packages/ameide_core_proto`.
- Policies fail Ring 2 if `pnpm why @ameideio/ameide-sdk-ts` resolves to npm instead of workspace, or if any Dockerfile/import references `packages/ameide_core_proto`.

**SDK publish track (TS)**

- `cd-packages` builds `@ameideio/ameide-sdk-ts` from the checked-in SDK tree, publishes to npm.pkg, and runs out-of-tree smoke tests installing the tagged package in a clean project.

### 3.2 Go

**Ring 1 (`Dockerfile.dev`)**

- `go.work` + `GOWORK=auto` point services at the workspace SDK module (Buf-generated stubs in `packages/ameide_sdk_go`), not `packages/ameide_core_proto`.

**Ring 2 (`Dockerfile.release`) - workspace SDK, no core_proto copies**

- Service images do not fetch `github.com/ameideio/ameide-sdk-go` or core_proto from the proxy.
- Dockerfile vendors or copies `packages/ameide_sdk_go` (with generated stubs) into the build; `GOWORK=off` points at that vendored SDK module.
- `GO_SDK_VERSION` stamping is only for SDK smoke jobs, not for service `go.mod` in Ring 2.

**SDK publish track (Go)**

- `cd-packages` tags/publishes `github.com/ameideio/ameide-sdk-go@vX.Y.Z` and runs smoke jobs that set `GOWORK=off`, `go get` the tagged SDK, and run targeted SDK tests.

### 3.3 Python

**Ring 1 (`Dockerfile.dev`)**

- `uv sync` with workspace `ameide-sdk-python` via `tool.uv.sources`; no PyPI required. Remove any `ameide_core_proto` alias so imports are `ameide_sdk.proto.ameide_core_proto.*` only.

**Ring 2 (`Dockerfile.release`) - workspace SDK, no core_proto copies**

- `Dockerfile.release` copies `packages/ameide_sdk_python` (with generated stubs) into the image.
- Uses `uv.lock` + `tool.uv.sources`; `uv sync --frozen` pins third-party deps only. Fails if `ameide-sdk-python` resolves from PyPI or if `packages/ameide_core_proto` is copied.

**SDK publish track (Python)**

- `cd-packages` builds the wheel from `packages/ameide_sdk_python`, publishes to PyPI, and runs out-of-tree smoke installs in clean venvs.

---

## 4. Changes to existing docs

- **402**: Fast map should show Ring 1/2 built from workspace SDKs (no core_proto imports) and a separate SDK product track.
- **403 - TS**:
  - Goal: prod images also use workspace `@ameideio/ameide-sdk-ts`; no `@ameide/core-proto` references in code or dist.
  - Publish/release path: stamped lockfile for third-party deps + `workspace:*` for SDK; add SDK publish + smoke subsection.
- **403 - Python**:
  - Make `Dockerfile.release` the Ring 2 Dockerfile; services import `ameide_sdk.proto.ameide_core_proto.*`; no workspace core_proto alias; no PyPI wheel in Ring 2.
- **404 - Go**:
  - Remove proxy/`GO_SDK_VERSION` guidance for service images; services import SDK-generated stubs; stamping only for SDK smoke tests.
- **405 - Dockerfiles**:
  - Impact: Ring 2 images copy workspace SDK packages (with stubs), not `packages/ameide_core_proto`, and must not install Ameide packages from registries.
  - Per-language prod/CI sections: drop version stamping/fetching; require SDK copies only.
  - `scripts/policy/check_docker_sources.sh`: expect workspace SDK copies, fail on registry installs or core_proto references.

---

## 5. Guardrails and acceptance

**Guardrails**

- TS: CI fails if `pnpm why @ameideio/ameide-sdk-ts` in Ring 2 resolves off workspace or if imports/Dockerfiles reference `packages/ameide_core_proto`.
- Go: CI lints that Ring 2 Dockerfiles omit GOPROXY/`GO_SDK_VERSION`, `go.mod` has no tagged SDK requirements, and no code imports `packages/ameide_core_proto`.
- Python: CI checks Ring 2 resolves `ameide-sdk-python` via `tool.uv.sources` and imports are `ameide_sdk.proto.ameide_core_proto.*`.
- Buf/validate: CI confirms buf/validate outputs exist inside each SDK package tree for TS/Go/Python.

**Acceptance criteria**

- Proto change -> SDK regeneration (via Buf) -> services in Ring 1 and Ring 2 fail until SDK trees are refreshed, without any SDK publish or registry access.
- No Ring 2 Dockerfile fetches Ameide SDK or core_proto from npm/PyPI/Go proxy; all copy `packages/ameide_sdk_*` only and exclude `packages/ameide_core_proto`.
- SDK publish pipelines (TS/Go/Py) run out-of-tree smoke tests that install published SDKs from registries and pass.
- 402/403/404/405 reflect the SDK-only proto surface and workspace-first Ring 2, with no guidance to use published SDKs or core_proto copies in service builds.
