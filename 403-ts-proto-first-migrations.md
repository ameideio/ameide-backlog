---
title: 403 – TypeScript proto-first service migrations
status: draft
owners:
  - platform
created: 2025-11-24
---

## Problem
- 402/407/408 require services to consume protos only through the SDK surface, with Buf outputs emitted into the SDK packages (synced from the BSR module); guardrails still force published `@ameideio/ameide-sdk-ts`/npm.pkg in Ring 2.
- Proto edits do not reliably break TS services until the SDK publish path runs; PR CI and Tilt carry npm/pkg auth and Buf registry coupling that slows iteration.
- Policy runner (`scripts/policy/run_all.sh`, including Docker checks) enforces the SDK-only stance; keep it aligned to the proto-first rings.

## Goal
- Inner loop + PR CI (dev and CI): TS services import a single surface (`@ameideio/ameide-sdk-ts/proto.js` + SDK entrypoints) resolved to the **workspace** SDK that already contains Buf outputs (including buf/validate) synced from BSR; proto changes break typecheck/tests immediately without any registry access.
- Ring 2 prod/staging images: built from the same workspace SDK (copied into Dockerfile.release) with strict third-party locks; no `packages/ameide_core_proto` copies and no npm/pkg fetch of Ameide packages.
- Outer loop: `cd-packages` still publishes `@ameideio/ameide-sdk-ts` for external consumers and out-of-tree smoke tests; release images in-repo do not depend on the publish.
- One AmeideClient contract (389) remains the runtime surface; only the dependency resolution changes.
- **Same imports, same source** for services: both rings resolve to workspace SDK packages; the published artifact is only used in the SDK product track.
- Supersedes backlog/393 for Rings 1/2 (dev/Tilt/PR CI/prod images built in-repo). 393 remains relevant for the SDK publish track (external consumers + smoke).

**File mapping reminder:** `Dockerfile.dev` == Ring 1 (workspace SDK, Tilt/dev containers). `Dockerfile.release` == Ring 2 (workspace SDK copied in, strict third-party locks). SDK publish/smoke uses separate jobs.

## Scope
- Services: platform, threads, transformation, workflows, chat, graph, repository, www_ameide, www_ameide_platform.
- Packages: `@ameide/core-proto`, `@ameideio/ameide-sdk-ts`, shared TS config/telemetry helpers.
- CI/Tilt/Docker flows used by the TS services above.

> **Proto naming:** All proto packages referenced here must follow the conventions in [509-proto-naming-conventions.md](509-proto-naming-conventions.md) (root `ameide_core_proto.<context>[.<sub>].v<major>` packages and the intent/fact suffix rules).

## Non-goals
- Go/Python migration (covered elsewhere).
- Changing AmeideClient behaviour; this plan only rewires how TS services consume it.

## Plan

### 0) Foundations (workspace-first SDK)
- Single import surface: services import protos/clients from `@ameideio/ameide-sdk-ts/proto.js` (and SDK entrypoints) in both rings; dist artifacts must not reference `packages/ameide_core_proto`.
- Make the build chain deterministic: `pnpm --filter @ameide/core-proto build` → `pnpm --filter @ameideio/ameide-sdk-ts build` using workspace links only; document this as the pre-step for TS service builds.
- Ensure SDK unit/integration tests run against workspace stubs (mock + cluster) and fail fast if the resolved proto package is not the workspace one.

### 1) Guardrails & CI realignment
- Update `.github/workflows/ci-core-quality.yml` to remove “published-only” enforcement for TS and replace with:
  - a check that TS services resolve `@ameideio/ameide-sdk-ts` from the workspace (no npm.pkg hits) in PR jobs,
  - a lint that forbids `@buf/ameideio_*` imports outside `packages/ameide_core_proto`,
  - a required `pnpm --filter @ameide/core-proto build` before typecheck/tests.
- Remove npm.pkg auth from PR/Tilt install paths; keep it only in publish/release workflows.
- Add a policy script that fails if `pnpm why @ameideio/ameide-sdk-ts` shows a non-workspace version in any TS service’s `node_modules`.
- Document Docker split: `.dev` uses `pnpm install --no-frozen-lockfile` + workspace deps and no registry auth; `Dockerfile.release` uses stamped lockfile for third-party deps + `workspace:*` for SDK (copied into the image) with `--frozen-lockfile`.

### 2) Service migrations (single import surface)
For each service: import protos/clients solely from `@ameideio/ameide-sdk-ts/proto.js` (and SDK entrypoints); ensure Rings 1/2 resolve to the workspace SDK build. Remove registry auth from Docker/Tilt.
- **Platform/Threads/Transformation/Workflows/Chat/Graph/Repository/www_ameide/www_ameide_platform**: update proto barrels/clients to the single import surface; ensure Tilt/dev images run `pnpm --filter @ameide/core-proto build` then `pnpm --filter @ameideio/ameide-sdk-ts build` before service builds; confirm bundlers/Jest resolve the workspace SDK path and dist outputs have no `packages/ameide_core_proto` imports.

### 3) Dev ergonomics & Tilt
- Add a single `pnpm proto:ts` (core-proto build) + `pnpm sdk:ts` (SDK build) path to Tilt and devcontainer init so local runs never require registry access.
- Document inner-loop workflow: edit proto → `pnpm proto:ts` → service typecheck/test; Tilt watch should rebuild protos/SDK on proto changes.
- Add a fast “proto drift” check that compares working tree `packages/ameide_core_proto/dist` to regenerated output and fails dev/CI if stale.

### 4) Publish/release path
- Keep `cd-packages` responsible for tagging/publishing `@ameideio/ameide-sdk-ts`; run `npm pack` + out-of-tree install smoke tests on the built artifact.
- **Prod images (cd-service-images)**: use stamped `pnpm-lock.yaml` for third-party deps + `workspace:*` resolution for SDK; `pnpm install --frozen-lockfile` should still resolve to the copied workspace SDK (no npm/pkg access). Dev/Tilt images stay workspace-first.
- Emit SDK version + proto git SHA as artifacts so external consumers can match published SDKs to schema revisions.

### 5) Exit criteria / signals
- Proto changes that remove/rename fields break TS service typecheck/tests locally and in PR CI without hitting npm.pkg.
- `pnpm why @ameideio/ameide-sdk-ts` in each TS service points to the workspace; no `packages/ameide_core_proto` imports in runtime/dist.
- Docker/Tilt/CI paths for TS services run without registry credentials; publish workflow remains green using tagged builds and out-of-tree smoke.
- Service checklist below is all ✅.

## Status (2025-11-24)
- Workspace-first path landed: SDK builds from `packages/ameide_core_proto/dist` (no `@buf/...` in pnpm-lock), and TS services import proto barrels from that package with `workspace:*` SDK deps.
- Service Dockerfiles now build against workspace protos/SDK (`pnpm --filter @ameide/core-proto run build` + SDK build) with no registry auth.
- Remaining: add an explicit policy check that services resolve SDK/proto from the workspace and a proto-drift CI gate; re-run SDK + representative service builds in CI to lock this in.
- Verification pending: `pnpm -C packages/ameide_sdk_ts test` and a representative service build (e.g., `pnpm -C services/platform build`).

## Service checklist (TS)
- Platform [x]
- Threads [x]
- Transformation [x]
- Workflows [x]
- Chat [x]
- Graph [x]
- Repository [x]
- www_ameide [x]
- www_ameide_platform [x]
