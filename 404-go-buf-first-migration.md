---
title: 404 – Go Services Buf-First Migration
status: draft
owners:
  - platform
  - go
created: 2025-11-24
---

## Goal

Move **all Go services (agents, inference_gateway, future templates)** to the 402/407/408 model: proto-first, SDK-only surface with Buf outputs generated into `packages/ameide_sdk_go`, immediate break on proto drift, and outer-loop publishes that do not block internal builds. Services consume stubs from the SDK (not `packages/ameide_core_proto` or `buf.build/gen/...`). Ring 1 and Ring 2 resolve to the workspace SDK (copied/vendor in Dockerfiles); the SDK publish track is separate for external consumers. Versioning/tag authority follows backlog/390 for the publish track.
**Same imports, same source for services**: Go code stays identical across rings; both resolve modules via workspace SDK code, not registry downloads.

## Context

- Current Go services still reference Buf registry stubs (`buf.build/gen/...`) or pull tagged SDKs in go.mod/Dockerfiles. Proto changes do not surface until a publish.
- Buf generation needs to emit stubs (including buf/validate) directly into `packages/ameide_sdk_go` synced from the BSR module; services should import only from that SDK surface.
- Rings 1/2 should both use the workspace SDK (copied/vendor) with no GOPROXY fetch of Ameide modules; the publish track is for external consumers and smoke only.

**Naming:** `Dockerfile.dev` implements Ring 1 (workspace stubs/SDK in Tilt/dev containers). `Dockerfile.release` implements Ring 2 (release/CI builds that pin tagged SDK modules). References below follow that convention.

## Target State (Go)

- `packages/ameide_core_proto` pushes schema to BSR; stubs (including buf/validate) are synced into `packages/ameide_sdk_go/gen/...` from the BSR module; SDK sources are the only import surface for services.
- `packages/ameide_sdk_go` is consumed via `go.work` in Rings 1/2; services do not import `packages/ameide_core_proto/gen/...` or `buf.build/gen/...`.
- Services build/tests run with `GOWORK=auto` and fail immediately when protos change and stubs are regenerated into the SDK.
- Ring 2 service images vendor/copy the workspace SDK and build with `GOWORK=off` pointing to that vendored module; no GOPROXY fetch or GO_SDK_VERSION stamping.
- SDK publish track uses `GOWORK=off` + tagged `ameide-sdk-go@vX.Y.Z` in smoke to guard external consumers.
- Guardrails prevent `buf.build/gen/...` imports, `packages/ameide_core_proto` imports, and Buf GOPROXY usage in service/Dockerfile paths.

## Plan

1) **Stub source pivot (packages/ameide_core_proto → BSR)**
   - Add a Go codegen template that writes stubs into `packages/ameide_sdk_go/internal/proto` (private) with module path `github.com/ameideio/ameide-sdk-go/internal/proto/...`, then regenerate the service-facing SDK proto surface under `packages/ameide_sdk_go/proto` (imports for services: `github.com/ameideio/ameide-sdk-go/proto/...`).
   - Wire `pnpm --filter @ameide/core-proto build` (and CI proto job) to run Go generation against the BSR ref and fail if `git status` shows dirty SDK `gen/go`.
   - Export Go package docs showing the import path pattern for services and SDK (connect + grpc packages) from the SDK module.

2) **SDK alignment (packages/ameide_sdk_go)**
   - Point SDK imports to the in-tree generated stubs under `packages/ameide_sdk_go/gen/...`; trim Buf proxy/env docs.
   - Keep SDK tests green under `GOWORK=auto`; add a quick check that no dependency in go.mod/go.sum references `buf.build`.
   - Document outer-loop publish path: tags build with `GOWORK=off`, `go get github.com/ameideio/ameide-sdk-go@<tag>`, and smoke against a clean module cache (external consumers only; prod images remain workspace SDK-first).

3) **Workspace wiring**
   - Expand `go.work` to include `services/agents` and `services/inference_gateway` so `go list`/`go test ./...` resolve workspace SDK consistently.
   - Add a `tools/go-env.sh` (or README snippet) setting `GOWORK=auto` for dev/CI inner loop; make Tilt/PR jobs invoke Go with `GOWORK=auto`.
   - Add a repo-level lint that rejects `GOPROXY=https://buf.build/...` in Dockerfiles and `buf.build/gen` or `packages/ameide_core_proto` module paths in go.mod/go.sum outside Ring 0 tooling.

4) **Service migrations**
   - **agents**
     - Update imports from `buf.build/gen/...` (or core_proto paths) to `github.com/ameideio/ameide-sdk-go/proto/...`.
     - Drop Buf modules from go.mod; rely on workspace SDK stubs. Ensure integration tests (bufconn) still run under `GOWORK=auto`.
     - Simplify Dockerfile/Dockerfile.dev to remove Buf GOPROXY/GONOSUMDB; dev/Tilt images copy `go.work` + workspace SDK, prod images keep `GOWORK=off` pointed at the copied SDK (no GO_SDK_VERSION/proxy).
   - **inference_gateway**
     - Same import swap to workspace SDK stubs; purge Buf GOPROXY settings.
     - Keep dual-mode integration test using SDK clients; verify spans/metadata assertions still pass with local stubs.
   - **templates/new Go services**
     - Update `services/_templates/integration/go` to use workspace SDK stubs and inherit the new GOPROXY/GOWORK defaults.

5) **CI + release path**
   - **PR/branch**: run `pnpm --filter @ameide/core-proto build && go test ./...` from repo root (go.work active); fail on dirty tree or `buf.build`/core_proto imports in services.
   - **Main/tag publish**: run `GOWORK=off go test ./...` in SDK + services using the tagged SDK version for smoke; prod images stay on workspace SDK even after tag.
   - Add an automated check that regenerating protos (`pnpm --filter @ameide/core-proto build`) followed by `go test` produces no diff; block merges otherwise.

6) **Clean-up and comms**
   - Update READMEs (core-proto, SDK, services) to reflect “workspace stubs for internal, tagged SDK for external”.
   - Remove Buf registry notes from onboarding docs; keep a short “outer loop publish” appendix that explains when Buf/registry is still used.
   - Announce the migration steps and the temporary dual path (Buf proxy support stays only until both services flip).

## Status (2025-02-22)

- Workspace Go stubs are committed under `packages/ameide_sdk_go/internal/proto/...`; `pnpm --filter @ameide/core-proto build` emits Go outputs into the SDK and regenerates the public proto surface under `packages/ameide_sdk_go/proto/...`.
- Go SDK imports the workspace stubs (no `buf.build` modules).
- `go.work` includes `packages/ameide_sdk_go`, `services/agents`, and `services/inference_gateway`; service Dockerfiles/dev images now set `GOWORK=auto` and build from workspace sources (no Buf GOPROXY).
- Services (`agents`, `inference_gateway`) import stubs from `ameide-sdk-go/proto/...`; integration/test binaries are built from local stubs.
- Remaining gaps: CI guard for `buf.build` imports/stub drift is still needed; root `go.mod` still carries buf.build module deps for tooling and should be folded into the workspace story or isolated. Outer-loop publish smoke with `GOWORK=off` is still required.

## Acceptance Checklist

- [x] `packages/ameide_core_proto` generates and commits Go stubs into the SDK; `pnpm --filter @ameide/core-proto build` is green and leaves a clean tree.
- [x] `packages/ameide_sdk_go` imports only workspace stubs; go.mod/go.sum contain no `buf.build` entries.
- [x] `go.work` includes agents + inference_gateway; `go test ./...` at repo root works without GOPROXY overrides.
- [x] `services/agents` and `services/inference_gateway` import stubs from `ameide-sdk-go/proto/...` and build without Buf registry access.
- [x] Dockerfiles drop Buf GOPROXY and rely on workspace SDK for dev/prod in-repo; tag-smoke runs use `GOWORK=off` + tagged SDK for external consumers.
- [ ] CI enforces no `buf.build` imports in services, validates stub freshness, and runs inner-loop tests with workspace deps; publish jobs smoke `GOWORK=off` with tagged SDK.

## Implementation Status (2025-11-24)

- Ring 0: `packages/ameide_core_proto` generates committed Go stubs into the SDK; `go.mod` clean, `go test` passes.
- Ring 1 SDK: `packages/ameide_sdk_go` imports workspace stubs only; no `buf.build` deps; tests green.
- Services: `agents` and `inference_gateway` import workspace SDK stubs via replaces; `go test` green; Dockerfiles vendor workspace SDK and run with `GOWORK=off`.
- go.work scoped to `sdk_go`, `agents`, `inference_gateway` to avoid container-context path leaks.
- Remaining gap: add CI guards (no `buf.build` imports/GOPROXY/core_proto imports, stub freshness check) and run publish smoke with `GOWORK=off` + tagged SDK.
