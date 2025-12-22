# 393 – SDK Import & Proto Consumption Policy

**Status:** Draft (enforced by policy scripts)
**Owner:** Platform DX / SDK maintainers  
**Updated:** 2025-12-18

> **Update (2025-12-22):** The TypeScript SDK now ships deterministic runtime entrypoints (`/node`, `/browser`) to remove all runtime transport auto-detection. See `backlog/589-rpc-transport-determinism.md`.

## Goal

Align the repo to a **proto-first** workflow (Buf + `.proto` as the contract source of truth) while making **SDKs the only supported import surface for services**.

The ideal shape is the layered contract stack:

1. **Protos are the contract source** (`packages/ameide_core_proto`)
2. **Generated code is an SDK implementation detail** (lives inside each SDK package)
3. **SDK is the only supported surface for services** (no direct generated-proto/grpc imports)

## Layer 0 – Proto module (canonical)

- All `.proto` live in the Buf module at `packages/ameide_core_proto/`.
- CI/guardrails rely on `buf lint` and `buf breaking` (compat enforcement belongs to the proto layer).

## Layer 1 – Generated code (internal to SDKs)

Generated artifacts must land inside the SDK packages in **generated-only** directories:

- TypeScript: `packages/ameide_sdk_ts/src/_proto/**` (generated; not exported via `package.json#exports`)
- Go: `packages/ameide_sdk_go/internal/proto/**` (generated; enforced by Go `internal/`)
- Python: `packages/ameide_sdk_python/src/ameide_sdk/_proto/**` (generated; underscore = internal)

**Rule:** services must not depend on a standalone “proto package” (either a workspace proto tree or BSR stub packages). Generated code is an SDK implementation detail.

## Layer 2 – Public SDK surface (supported imports for services)

### TypeScript

- Services import proto namespaces only via their local barrel (e.g. `services/<svc>/src/proto/index.ts`).
- The barrel must re-export from `@ameideio/ameide-sdk-ts/proto.js`.
- Services must not import `@buf/*` or `packages/ameide_core_proto/**`.
- Runtime-specific client construction is explicit:
  - Node/server code imports client factories from `@ameideio/ameide-sdk-ts/node` (defaults to gRPC).
  - Browser/client code imports client factories from `@ameideio/ameide-sdk-ts/browser` (defaults to Connect).
  - `@ameideio/ameide-sdk-ts` root export may be used for types/helpers and for `createAmeideClient({ transport })` when an explicit transport is provided.

### Go

- Services import only SDK packages (clients + exported request/response types).
- Services must not import any generated proto/grpc/connect packages directly (enforced by Go `internal/` layout).

### Python

- Services import only SDK packages (clients + exported request/response types).
- Services must not import `ameide_core_proto.*` or any SDK internal `_proto` modules directly.

## Forbidden dependency patterns (runtime services)

- Importing proto sources directly: `packages/ameide_core_proto/**`
- Direct Buf stub packages/imports: `@buf/*`, `buf.build/gen/*`
- Importing generated code directly (Go/Python/TS deep imports)

## Enforcement (repo tooling)

- `scripts/policy/enforce_proto_imports.sh` blocks direct workspace proto usage, direct Buf stub usage, and direct imports of generated code in services.
- `scripts/policy/check_proto_workspace_barrels.sh` ensures TS service proto barrels point at `@ameideio/ameide-sdk-ts/proto.js`.
