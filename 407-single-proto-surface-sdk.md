# 407 – Single proto surface via SDK (objective + gaps)

## Objective
Consume protos only through SDK packages (Go/Python/TS) with Buf/BSR emitting stubs directly into the SDK trees. Services/images must not rely on `packages/ameide_core_proto` copies; dist artifacts import SDK barrels; policies/CI enforce SDK-only (including buf/validate).

> **Proto naming:** When adding or refactoring proto packages in support of this objective, follow the package/topic conventions in [509-proto-naming-conventions.md](509-proto-naming-conventions.md) so schemas stay aligned across domains and SDKs.

## Current alignment (2025-11-27)
- Buf templates include protovalidate for Go/Python/TS; SDK sync scripts run in CI (`ci-proto.yml` sdk-sync-bsr) and Tilt.
- Policy scripts enforced in PR gate and service-image workflow:
  - `scripts/policy/enforce_proto_generation.sh` (single Buf module guard)
  - `scripts/policy/enforce_proto_imports.sh` (blocks @buf/*, packages/ameide_core_proto in source/dist, BSR stub wheels/modules)
  - `scripts/policy/check_proto_workspace_barrels.sh` (TS barrels import SDK proto)
  - `scripts/policy/check_lock_hygiene.sh` + `scripts/policy/check_go_option_a.sh`
  - `scripts/policy/check_docker_sources.sh` (Dockerfiles copy SDKs, no core_proto/registry SDK fetches)
- `cd-service-images` now picks `Dockerfile.dev` on dev branch and `Dockerfile.release` on main/tags, keeping Ring 1/2 workspace-first.

## Remaining gaps
1) **Buf/validate regeneration tracking**
   - Ensure a periodic or on-change regeneration of SDK stubs when Buf templates change; add a note to bump proto runbook to re-run SDK sync after template changes.

2) **Doc parity**
   - Cross-check 402/403/404/405/408 to ensure they no longer suggest pulling published SDKs for service images or copying core_proto; align wording with workspace SDK-only for Rings 1/2.

3) **Artifact scans**
   - Keep TS dist scans for `@buf/` and `packages/ameide_core_proto` (already in policy); consider similar checks for Go/Py built artifacts referencing BSR/gen paths.

4) **Channel clarity**
   - Service-image workflow now respects branch channel for Dockerfile selection; ensure docs mention dev → Dockerfile.dev and main/tags → Dockerfile.release.
