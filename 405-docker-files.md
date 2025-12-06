# 405 – Dockerfiles (rings-aligned, SDK-only surface)

This aligns Dockerfiles with 402/407/408: services consume protos only through SDK packages; Buf outputs (including buf/validate) live in the SDK trees. Rings 1 and 2 both use workspace SDKs; the SDK publish/smoke track is separate.

## Rings recap

- **Ring 0 (Buf + core_proto + BSR)** – single schema module; Buf writes stubs into SDK packages by targeting the BSR module.
- **Ring 1 (`Dockerfile.dev`)** – dev/Tilt/PR CI images copy workspace SDK packages (not `packages/ameide_core_proto`), use relaxed installs, no registry access.
- **Ring 2 (`Dockerfile.release`)** – staging/prod images also copy workspace SDK packages (no Ameide registry downloads), use strict frozen installs for third-party deps; Ameide packages resolve via `workspace:*`/vendored SDK. SDK publish/smoke is an outer loop.

## Dev / Tilt Dockerfiles (`Dockerfile.dev`)

General:
- Copy workspace SDK packages only; do not copy `packages/ameide_core_proto`.
- No npm/PyPI/Go registry access for Ameide packages; relaxed installs for third-party deps.
- Service-image workflow on `dev` selects `Dockerfile.dev`; keep SDK-only rules identical to Ring 2.

TypeScript:
- Copy `packages/ameide_sdk_ts` + service sources; install with `pnpm install --no-frozen-lockfile`.
- Imports must go through `@ameideio/ameide-sdk-ts/proto.js`; dist must not reference `packages/ameide_core_proto`.

Go:
- Copy `packages/ameide_sdk_go`; set `GOPROXY=off`; use a shared helper (e.g., `scripts/ci/go_use_workspace_sdk.sh`) to prime the SDK module, apply `go mod edit -replace` to the workspace SDK, and `go mod download` before build/test.
- Services import `ameide-sdk-go/gen/...` only; no `buf.build/gen/...` modules.

Python:
- Copy `packages/ameide_sdk_python`; use `tool.uv.sources` + editable installs; `uv sync` (non-frozen) ok.
- Imports must use `ameide_core_proto.*` from the SDK package; no workspace `ameide_core_proto` alias.

## CI / Prod Dockerfiles (`Dockerfile.release`)

General:
- Copy workspace SDK packages; do not copy `packages/ameide_core_proto`.
- Ameide packages must resolve from the copied SDKs (no registry fetch). Frozen installs apply to third-party deps.

TypeScript:
- `pnpm-lock.yaml` stamped for third-party deps; `@ameideio/ameide-sdk-ts` stays `workspace:*`.
- `pnpm install --frozen-lockfile` should resolve to the copied workspace SDK; fail builds that pull from npm.

Next.js Standalone (www-ameide, www-ameide-platform):
- `outputFileTracingRoot: repoRoot` in next.config.ts to include shared packages and resolve pnpm symlinks.
- Standalone output preserves monorepo directory structure; server.js is at `services/<name>/server.js`.
- Dockerfile.release COPY and CMD must use nested paths:
  ```dockerfile
  COPY --from=builder /app/services/<name>/.next/standalone ./
  CMD ["node", "services/<name>/server.js"]
  ```
- Reference: https://nextjs.org/docs/pages/api-reference/config/next-config-js/output

Go:
- Vendor/copy `packages/ameide_sdk_go`; set `GOPROXY=off`; use the shared helper to prime the SDK module, apply the replace to the vendored path, and `go mod download`/`go mod tidy` with tags as needed.
- `go.mod` should use the SDK module path with a replace to the vendored path; no `buf.build/gen/...` modules in service images; no `GO_SDK_VERSION` stamping.

Python:
- Copy `packages/ameide_sdk_python`; `uv sync --frozen` with `tool.uv.sources` so Ameide packages resolve to workspace; forbid PyPI for Ameide deps.
- Forbid direct BSR stub package installs in service images across languages (`@buf/...`, `buf.build/gen/go/...`, BSR Python wheels); only SDK packages should appear.

## Guardrails

- `scripts/policy/check_docker_sources.sh` expects SDK copies and fails on registry fetches, `packages/ameide_core_proto` copies, or GO_SDK_VERSION/PYTHON_SDK_VERSION/NPM-SDK version stamping in service Dockerfiles.
- `scripts/policy/check_dockerfile_parity.sh` enforces consistency between `Dockerfile.dev` and `Dockerfile.release`:
  - Neither should copy `packages/ameide_core_proto` (SDK-only surface)
  - Python services must have consistent `PYTHONPATH` including SDK src in both variants
  - TypeScript services must copy `packages/ameide_sdk_ts` in both variants
  - Go services must copy `packages/ameide_sdk_go` and use `go_use_workspace_sdk.sh` in both variants
  - No SDK version stamping in either variant
- `scripts/policy/enforce_proto_imports.sh` and `scripts/policy/check_lock_hygiene.sh` enforce SDK-only imports (no `packages/ameide_core_proto` in runtime code) and buf/validate presence in SDK trees.
- `scripts/ci/go_use_workspace_sdk.sh` provides a shared Go workspace-SDK hydration step (prime SDK module, disable GOPROXY, apply replace); dev and release Dockerfiles should use it for Go services.
- CI should fail if `pnpm why @ameideio/ameide-sdk-ts`, Go `go.mod` resolution, or Python imports resolve anywhere other than the workspace SDK packages for Rings 1/2.
