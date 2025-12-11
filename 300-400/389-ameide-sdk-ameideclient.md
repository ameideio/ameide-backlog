# AmeideClient Parity & Target API

Doc family: backlog/388-ameide-sdks-north-star.md (release/versioning goals), backlog/390-ameide-sdk-versioning.md (CI version computation), backlog/391-resilient-cd-packages-workflow.md (publish gates), backlog/393-ameide-sdk-import-policy.md (runtime import contract), backlog/395-sdk-build-docker-tilt-north-star.md (Docker/Tilt consumption of the SDKs).

## Problem
Each SDK currently exposes its own flavor of `AmeideClient`. Default hosts, retry policies, auth hooks, and observability features differ by language, and TypeScript carries extra helpers (e.g., `elementService`) that Go/Python never receive. Multi-language services therefore see inconsistent behavior (e.g., Python lacks telemetry entirely, Go silently dials `localhost:50051`, TypeScript refuses to retry `RESOURCE_EXHAUSTED`). We need a single target contract so every SDK implements the same functionality and we can add generated coverage tests to catch drift.

## Goals
- Document the authoritative `AmeideClient` API that TypeScript, Go, and Python must implement.
- Normalize configuration defaults (endpoints, retries, timeouts, metadata) so developers get identical behavior regardless of language.
- Require shared extensibility hooks (auth, interceptors, telemetry, transport overrides) and describe error handling semantics.
- Call out optional convenience helpers and mandate which ones belong inside the core client versus separate packages.
- Define verification steps (unit tests + release gates) to keep SDKs aligned with the Buf service roster.
- Ensure all packages and services honor the AmeideClient contract; external consumers use the published SDKs, while monorepo services may link the workspace AmeideClient implementations as long as they follow the same API (see backlog/393-ameide-sdk-import-policy.md for consumption guardrails).

## Target API Surface
### Service Coverage
All SDKs expose the same strongly-typed clients, matching `packages/ameide_core_proto/buf.gen.yaml`:
- `agents`, `agentsRuntime`
- `graph`
- `inference`
- `governance`
- `platform`: `organizations`, `organizationRoles`, `teams`, `users`, `invitations`, `tenants`
- `transformation`
- `threads`
- `workflows`

Exported property names remain camelCase per language conventions but must stay 1:1 with the proto package roster. Add a table-driven test (TS unit test, Go/Python integration test) that fails if Buf generates a service not wired into the client set.

### Construction & Options
Each SDK exposes a shared option struct/interface with the following defaults:
- `baseUrl` / `endpoint`: `https://api.ameide.io` (browser/Connect style) or `api.ameide.io:443` (gRPC). No more localhost default.
- `timeout`: 5 seconds per RPC (AbortController for TS, context deadlines for Go, interceptor timeouts for Python).
- `retry`: max 3 attempts, 250ms initial backoff, 5s cap, multiplier 2. Retry codes include `Unavailable`, `DeadlineExceeded`, `Aborted`, `ResourceExhausted`.
- `tenantId`, `userId`, `headers`: merged into every RPC metadata; `headers` keys are lower-cased.
- `requestIdGenerator`: optional function. SDKs must inject `x-request-id` when missing.
- `auth`: pluggable provider invoked per RPC. TS: `AuthProvider#getToken()` can be async. Go: `AuthTokenProvider(ctx)` returns `(string, error)`. Python: `TokenProvider(ctx?)` must be able to raise exceptions or return awaitable; expose both sync + async helpers (or document one canonical async->sync adapter).
- `interceptors`: user-defined interceptors appended after built-ins.
- Transport override knobs: `transport`/`transportFactory` (TS), `WithDialOption`/`WithTransportCredentials` (Go), `grpc.Channel` override hook (Python) so advanced users can bring their own connection stack.

### Metadata & Interceptors
- Built-in metadata interceptor handles tenant/user IDs, headers, and request ID injection.
- Auth interceptor attaches `Authorization: Bearer <token>` when the provider returns one, surfacing provider errors to callers.
- Timeout interceptor enforces the configured per-RPC budget.
- Retry interceptor implements exponential backoff with jitter (`0.5–1.5x`).
- Ordering: metadata → auth → timeout → retry → tracing → user interceptors (TS order mirrored in Go/Python by chaining interceptors in that sequence).

### Telemetry
- All SDKs expose tracing hooks. TS already takes `{ tracer, spanName?, attributes? }`. Go installs `otelgrpc.ClientHandler` today; keep it and allow custom span attributes. Python must add a tracing interceptor that accepts a tracer provider and emits `rpc.*` attributes just like TS.
- Surface hooks for metrics (future). At minimum ensure request/response metadata includes `x-request-id` so traces correlate across languages.

### Convenience Helpers
- `elementService` (Graph convenience wrapper) remains available, but move it behind an optional factory (`createElementService(client.graph)` for TS, `NewElementService(client.Graph)` for Go, etc.). Keep `AmeideClient` itself strictly focused on service stubs; export helpers from sibling modules so all languages can opt-in without bloating the constructor.

### Errors & Resource Management
- Provide explicit `close()` / `shutdown()` (Python) or `Close()` (Go) to tear down transports. TS no-op unless custom transport exposes cleanup.
- Ensure retry failures re-throw the original RPC error type (`ConnectError`, `status.Status`, `grpc.RpcError`). Include final attempt count in logs or error metadata when possible.

### Compliance Tests
- Shared fixture describing the expected service keys (JSON manifest committed in `sdk/manifest/ameide-client.json`). SDK tests read the manifest and assert each entry exists.
- Integration sanity tests hitting the Envoy endpoint once per release with identical environment variables (`INTEGRATION_GRPC_ADDRESS`, `INTEGRATION_TENANT_ID`, etc.).
- Lint: CI step compares option defaults across SDKs (script diffing manifest) so future edits fail fast.
- Enforce repo-wide lint that rejects direct proto imports/usage in services or packages (SDK-only consumption).
- Compliance checks run as part of `cd-packages` (backlog/391-resilient-cd-packages-workflow.md) so publish gates stay aligned with the manifest.

## Status / Progress
- ✅ Manifest added at `sdk/manifest/ameide-client.json` and wired into TS/Go/Python unit tests.
- ✅ TS SDK defaults aligned (hosted base URL, retry codes inc. `RESOURCE_EXHAUSTED`, interceptor order exported). `AmeideClient` now exposes a no-op `close()` and `elementService` is factory-only.
- ✅ Go SDK aligned on defaults and tracing: tracing interceptor now sits in the manifest order with optional tracer provider/span attributes, and streaming interceptors mirror the unary pipeline (metadata → auth → timeout → retry → tracing → user).
- ✅ Python SDK aligned: async auth provider, tracing interceptor emits `rpc.service`, retry ordering, manifest test, and channel override hook added.
- ✅ CI helper script `scripts/check_ameide_client_manifest.sh` now runs in the publish workflow.
- ✅ AmeideClient contract documented in `sdk/README.md` alongside the manifest.
- ✅ Convenience helpers now optional across SDKs: TS factory-only; Go/Py expose matching element helper factories without bloating constructors.
- ✅ Go SDK now has a dedicated repo (`github.com/ameideio/ameide-sdk-go`) tagged `v0.1.0` with matching GHCR SemVer aliases; services pin the published module.

## Outstanding Gaps (current code vs target)
- SDK READMEs should stay aligned with `sdk/README.md` and the manifest for quick discovery.
- Integration env names now accept `INTEGRATION_*` plus legacy fallbacks; each SDK README should document the canonical variables.
- Integration coverage still needs to assert the manifest behaviours end-to-end (headers/retry/timeout/tracing) for Go/TS/Python against real endpoints on release (Go bufconn tests cover retry/timeout/metadata/tracing for inference_gateway and agents; tracing in other services and live endpoints still pending).

## Rollout Checklist
1. ✅ Publish this spec, socialize in #sdk and architecture review.
2. ✅ Update TS SDK to export manifest-aligned defaults (optionally exposed helper factory) and provide `close()`.
3. ✅ Align Go defaults (`WithAddress` default -> hosted URL) and add tracing attributes + helper factories/interceptor ordering that includes tracing.
4. ✅ Extend Python SDK with async-capable auth provider (done), telemetry interceptor (`rpc.service` added), configurable channel factory (added), and helper factories; adopt manifest tests.
5. ✅ Wire CI guard that runs `scripts/check_ameide_client_manifest.sh` (or equivalent) before publishing packages.
6. ✅ Document the unified API in `sdk/README.md` and link from each language README (keep README links in sync).
7. ✅ Enforce SDK-only consumption in packages/services (`scripts/sdk_policy_enforce_proto_imports.sh` guard is in repo and passes).
