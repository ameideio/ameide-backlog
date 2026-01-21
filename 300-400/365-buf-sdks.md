# Buf SDKs & Thin Wrapper Rollout

## Status: DONE
**Created**: 2025-02-14  
**Priority**: High  
**Complexity**: Large  
**Related**: [336-sdk-proto-dependency-hardening.md](./336-sdk-proto-dependency-hardening.md), [357-protovalidate-managed-mode.md](./357-protovalidate-managed-mode.md)

## Problem Statement

Even after adopting Buf Managed Mode, most services still import generated stubs directly:

1. **Inconsistent policy** – auth headers, retries, request IDs, and tracing are re‑implemented per service and drift quickly.
2. **Redundant transports** – each TypeScript service builds its own Connect transport; Go services hand-roll gRPC clients; Python lacks any reusable layer.
3. **Difficult releases** – without official SDK packages, Dependabot cannot track Buf label bumps and partner teams cannot consume prereleases.
4. **Testing gaps** – no unit coverage for interceptors/options, so regressions in headers/timeouts go unnoticed.

## Target State

* Buf builds the raw artifacts; we ship **thin, opinionated SDKs** per language.
* TypeScript (`packages/ameide_sdk_ts`): single entry point exporting clients, transports, interceptors, error helpers, and proto re-exports.
* Go (`packages/ameide_sdk_go`): `sdk.Client` controls metadata, auth, retries, OTel interceptors, and exposes structured `RPCError`.
* Python (`packages/ameide_sdk_python`): installable wheel that lazy-loads Buf stubs, applies metadata/auth/retry interceptors, and provides a parity API surface.
* CI runs unit suites for every SDK and releases only require bumping dependency labels.

## Implementation Summary

### TypeScript
- `packages/ameide_sdk_ts` now exposes a single `AmeideClient`/`ClientSet` entry point plus transports, interceptors, context builders, and RPC error helpers. The tracing interceptor had an ordering bug that prevented spans from starting; calling `createTracingInterceptor` now correctly invokes `tracer.startActiveSpan(name, options, context, fn)` so OTEL spans wrap every RPC and span attributes/statuses are emitted as intended (`src/interceptors.ts`).
- Transport factories autodetect Node vs browser, attach shared interceptors, and all interceptors remain pure functions (`createMetadataInterceptor`, `createAuthInterceptor`, `createRetryInterceptor`, `createTimeoutInterceptor`, `createTracingInterceptor`) so services can mix/match behaviour.
- Error normalization (`src/errors.ts`) maps status codes into categories and retryable flags; request-context helpers and proto re-exports keep downstream code away from raw Buf imports.
- Dec 2026 update: `proto.ts` now re-exports `AgentsService` and `ClientSet`/`AmeideClient` expose matching clients, so the TS SDK stays 1‑to‑1 with every Buf service. A new unit test asserts the client keys line up with the proto roster, catching future drift.
- Tests: unit suites cover metadata/auth/retry/timeout interceptors and client wiring; the integration spec (`__tests__/integration/ameide-client.integration.test.ts`) now targets the live Envoy endpoint using the Vault-sourced `INTEGRATION_*` secrets enforced by backlog/362.

### Go
- `packages/ameide_sdk_go` options expose auth providers, retry config, headers, request IDs, timeouts, and custom interceptors. Dialing (`conn.go`) wires keepalives, otelgrpc stats handlers, metadata/auth/retry/timeouts, and any user-provided interceptors so policies stay consistent.
- Metadata/auth/retry interceptors live in `interceptors.go`, while `errors.go` emits structured `RPCError` values for upstream handling. `client.go` wraps every Buf service client into a single struct for parity with the TS SDK.
- Tests: unit tests cover metadata headers and retry behaviour; the integration test spins up a bufconn server to verify metadata injection, retries, auth, and deadline enforcement. CI now runs these `go test` suites with Buf registry credentials while excluding generated packages.

### Python
- The `ameide-sdk` package mirrors TS features: `SDKOptions` (now with `request_id_provider`), interceptors for metadata/auth/timeout, lazy channel creation, and request context helpers. A new retry wrapper (`ameide_sdk/retry.py`) wraps unary stubs so retry logic actually executes; the old retry interceptor has been removed to avoid double-interceptor recursion.
- Metadata interceptors respect pre-existing `x-request-id` headers, timeout interceptors clamp deadlines, and the client composes metadata, auth, timeout, and retry layers before exposing service stubs.
- Protobuf shims (`ameide_core_proto/__init__.py`, `buf/validate/validate_pb2.py`) allow local tests without installing Buf wheels.
- Dec 2026 update: `AmeideClient` exposes the `AgentsService` stub alongside every other Buf service, and a unit test asserts all accessors exist. Integration tests now run exclusively against the live Envoy endpoint and refuse to start unless the secrets mandated by backlog/362 (`INTEGRATION_*`, `INTEGRATION_GRPC_ADDRESS`/`NEXT_CORE_GRPC_URL`) are present, eliminating the old mock-server path.
- Tests: unit tests cover metadata/auth/timeout interceptors plus the retry wrapper; integration t
[... output truncated to fit 10240 bytes ...]


## Definition of Done

- All services depend solely on the SDK packages.
- Buf label promotion automatically produces SDK releases (npm/Go/PyPI) with provenance.
- CI enforces SDK tests and forbids direct proto imports.
- Documentation and examples published for internal and partner teams.
