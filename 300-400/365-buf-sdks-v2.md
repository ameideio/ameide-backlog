# Buf SDKs Architecture (v2)

**Status**: Draft  
**Intent**: Buf is the single source of truth. All languages ship parity SDKs published to Ameide GitHub registries, and every service consumes those SDKs at runtime (no intra-repo stubs or replaces). Dev/test can read the committed proto bundle from the shared package; runtime never does.

## Principles
- **Buf-centric**: One managed Buf module defines APIs; no service owns bespoke protos.
- **Language parity**: TS, Go, and Python expose the same service roster and behaviours (auth/tenant headers, retries, timeouts, tracing, request IDs).
- **Published SDKs**: SDKs are versioned and released to Ameide GitHub registries (npm, Go module proxy, PyPI, GHCR). Services consume released artifacts only.
- **Policy in SDKs**: Auth, tenancy, retries, deadlines, tracing, metadata, and error shaping live in the SDKs, not per-service glue.
- **CI guardrails**: CI blocks runtime `buf.build/gen` imports in services, enforces SDK test suites, and publishes SDKs on git `vX.Y.Z` tags (dev snapshots from `main` only).
- **Single source of truth**: All protos live in one Buf-managed module; generated outputs are committed once in the shared proto package for dev/test. No generated code is committed in services; SDKs are the runtime distribution vehicle for stubs.
- **Thin wrappers**: SDKs are minimal layers atop BSR-generated code, adding only cross-cutting concerns (auth, tenancy, retries, timeouts, tracing, errors). The exported API surface must stay aligned with the Buf manifest.
- **Wrapper validation only**: We trust BSR for generated stubs; CI validates only the custom thin wrappers’ surface and behaviour.

## Deliverables
- **TypeScript**: `@ameideio/ameide-sdk-ts` (Connect transport factory, interceptors, error helpers, proto re-exports).
- **Go**: `github.com/ameideio/ameide-sdk-go` (managed dialer, interceptors, typed client set, structured errors).
- **Python**: `ameide-sdk-python` (lazy channel, metadata/auth/retry/timeout wrappers, unified client).
- **Release automation**: Git `vX.Y.Z` tags publish all SDKs plus GHCR images (`ghcr.io/ameideio/ameide-sdk-{ts,go,python}:<sha|dev|X.Y.Z>`), with Dependabot PRs in downstream services.

## Gaps to Close (target state only)
- **Publish pipeline not aligned**: TS/Python/Configuration still lack SemVer tags + GHCR aliases in CI; latest dev builds for TS/config were published manually to npmjs but the `cd-packages` workflow fails earlier (pnpm install) so GHCR aliases and SemVer releases remain missing; no “fail on older tag” guard yet. (Align with 388/390/391.)
- **Parity gates missing**: AmeideClient manifest and wrapper-behaviour parity (auth/tenant/retry/timeout/tracing, headers/request-id) are not enforced as publish blockers across TS/Go/Python (389).
- **Policy enforcement**: Proto-import bans, go `replace` bans, lock hygiene, and Buf bundle freshness checks exist but are not wired as merge blockers in this spec (393/391).
- **Service adoption gaps**: Remaining services still touch raw stubs or lack SDK config/tracing/tests (Graph, Repository, www apps; Go agents/inference_gateway; Python inference + runtime tests).
- **Integration coverage**: SDK-backed integration suites for headers/auth/retry/timeout/tracing are incomplete and not dual-mode (376).

## Implementation Plan (no shims, target-state only)
1) **Align publish/CD with north star**  
   - Adopt version computation from 388/390: Git tag drives SemVer; main produces dev/SHA builds; always publish GHCR `:<sha>` + semantic aliases; block releases older than the latest tag.  
   - Run SDK tests before packing; smoke-install from GHCR tarballs (no registry fetch) per 391; emit versions as artifacts.
2) **Enforce AmeideClient parity as a gate**  
   - Wire manifest parity + wrapper-behaviour tests from 389 into the publish workflow for TS/Go/Python; fail publishes on drift (service roster + auth/tenant/retry/timeout/tracing/headers).  
   - Keep stubs trusted; only wrappers are validated.
3) **Turn on CI guardrails (merges fail fast)**  
   - Proto-import lint: forbid runtime `buf.build/gen` imports outside SDK packages; tests use `packages/ameide_core_proto` only.  
   - Go `replace` ban and module path guard; ensure `GOPRIVATE/GONOSUMDB` set in CI.  
   - Lock hygiene for SDK deps (npm/go/uv) and Buf bundle freshness check.  
   - Reject service-local `buf.gen.yaml` and SDK proto access in runtime.
4) **Service migrations (no compatibility layers)**  
   - **Go**: Agents + Inference Gateway use `ameide-sdk-go` clients exclusively, wire config (addr/auth/tenant/timeouts), enable tracing interceptors, and add bufconn tests covering headers/auth/retry/timeout/tracing.  
   - **Python**: Inference swaps bespoke transports for `ameide-sdk-python`; all Python services add SDK-backed integration tests for headers/auth/retry/timeout/tracing.  
   - **TypeScript**: Graph, Repository, www_ameide, www_ameide_platform enforce SDK runtime clients (no raw `@buf`/proto.js), wire config/auth/tracing, and add SDK-backed integration/contract tests.  
   - Remove any proto-direct client usage; no shims or dual-path adapters.
5) **Integration mode + coverage**  
   - Apply integration-mode runner (376): `INTEGRATION_MODE=repo|local|cluster` enforced; non-cluster uses deterministic stubs, cluster asserts live deps.  
   - Add SDK-path integration suites per service covering metadata/auth/retry/timeout/tracing/request-id; wire into CI matrices (mock for PR, cluster for main/nightly).
6) **Release hygiene & dependabot**  
   - Ensure services pin released SDK versions; Dependabot bumps auto-merged; CI blocks if red checklist items remain.

## Service Checklists (must all be green)
Legend: [x] done, [~] partial, [ ] todo.

### Go
- **Agents**: [x] Depend on `ameide-sdk-go` (no replace); [~] server remains gRPC-based (`agentsv1grpc` allowed for serving, outbound must use SDK); [ ] wire config (addr/tenant/auth/timeout); [ ] tracing interceptors; [~] bufconn integration test for headers/auth/retry/timeout/tracing (metadata/retry/timeout/tracing covered; config/tracing wiring in runtime still TODO).
- **Inference Gateway**: [x] Depend on `ameide-sdk-go`; [~] server remains gRPC-based (`inferencev1grpc` allowed for serving, outbound uses SDK); [ ] config wiring (tenant/auth/timeouts); [~] tracing (bufconn test now asserts spans); [~] bufconn test (metadata/retry/timeout/tracing covered).

### Python
- **Agents-runtime / agents_runtime**: [x] Outbound SDK clients; [~] servicer imports only; [ ] integration test for headers/auth/retry/timeout via SDK.  
- **Inference**: [ ] Swap bespoke transports to `ameide-sdk-python`; [ ] wire config/auth/timeouts; [ ] tracing interceptors; [ ] SDK-backed integration test.  
- **Workflows runtime / worker**: [x] SDK callback clients; [~] auth provider optional; [ ] SDK integration tests for headers/retry/timeout/tracing.

### TypeScript
- **Platform, Threads, Transformation, Workflows, Chat**: [x] Runtime uses SDK; [x] config wiring; [ ] integration/contract test via SDK.  
- **Graph**: [ ] Enforce SDK runtime clients (no raw `@buf`); [ ] config/auth/tracing wiring; [ ] integration test via SDK.  
- **Repository**: [ ] SDK runtime clients; [ ] config/auth/tracing; [ ] integration test via SDK.  
- **www_ameide / www_ameide_platform**: [ ] SDK runtime clients only; [ ] config/auth/tracing; [ ] integration test via SDK.

## CI / enforcement tasks
- Wire proto-import lint, go replace guard, lock hygiene, and Buf bundle freshness into merge blockers.  
- Publish SDKs on git `vX.Y.Z` tags; main produces dev/SHA builds; fail releases older than latest tag; publish GHCR `<sha>` + semver aliases.  
- Run SDK unit/integration/parity suites in CI with Buf registry credentials before packing.  
- Block merges if any checklist item is red; reject service-local `buf.gen.yaml` and non-SDK runtime proto usage.  
- Guard module resolution with `GOPRIVATE/GONOSUMDB=github.com/ameideio`; ensure Dependabot PRs keep SDK pins current.

## Publishing tasks (SDKs to GitHub registries)
- Configure CI secrets for GitHub Packages (npm token, Go proxy credentials, PyPI token for GitHub Packages).  
- Add release workflows to publish `@ameideio/ameide-sdk-ts` to npm.pkg.github.com/npmjs with provenance on git tags.  
- Add release workflows to publish `github.com/ameideio/ameide-sdk-go` tags to GitHub Packages/Go proxy.  
- Add release workflows to build and publish `ameide-sdk-python` wheel to GitHub Packages (PyPI-compatible).  
- Publish GHCR images for each SDK/config artifact with tags `<sha>`, `dev`, `X.Y.Z`, `X.Y`, `latest`.  
- Generate SBOM/provenance for each SDK artifact; store alongside releases.  
- Create Dependabot configs to track SDK versions in all services (npm, Go modules, Python).  
- Document how to consume the published SDKs (auth to registries, example import/version pinning).

## API alignment tasks
- Emit a Buf-derived manifest of services/methods on each label; publish it as an artifact (source of truth for wrapper exports).  
- In each SDK, add tests that compare exported clients/services against the manifest (no missing/extra APIs); do not re-validate generated stubs.  
- Add contract tests (shared fixtures) to assert wrapper-only behaviour: auth/tenant headers, request IDs, timeouts, retry semantics, error mapping, and tracing are consistent across languages.  
- Gate SDK publish on passing wrapper parity + contract tests for all languages.
