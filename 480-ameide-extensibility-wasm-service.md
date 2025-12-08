# 480 – Ameide Extensibility WASM Service

**Status:** Draft  
**Owner:** Architecture / Platform  
**Depends on:** 479 (Extensibility model), 478 (Service catalog), 476 (Security), 472 (Controller contracts), 362 (Unified Secret Guardrails v2), 365 (Buf SDKs v2), 408 (Workspace-first Ring 2), 430 (Unified Test Infrastructure), 434 (Unified Environment Naming), 441 (Networking), 447 (ArgoCD rollout phases), 465 (ApplicationSet architecture)

---

## 1. Purpose

Establish the shared `extensions-runtime` service that runs Tier 1 WASM extensions inside `ameide-{env}`. The service must:

1. Expose the `WasmExtensionRuntime.InvokeExtension` RPC for Process/Domain/Agent controllers.
2. Resolve extension artifacts, execute them in a sandbox, and enforce risk-tier controls.
3. Ship as a standard platform workload (Helm + ArgoCD) with dev/release Dockerfiles similar to other services.

---

## 2. Scope

- **In scope**
  - New service under `services/extensions-runtime` (Go preferred) with README, Tilt target, and pairing Dockerfiles.
  - Runtime proto (`packages/ameide_core_proto/ameide/extensions/v1/runtime.proto`) plus SDK bindings.
  - Module storage/caching, Wasmtime-based sandbox executor, and host-call bridge leveraging shared MinIO buckets per 479/362 guardrails.
  - Config hydration from Transformation promotion events, observability, and guardrails (limits, SLO telemetry).
  - Helm chart + GitOps wiring for each platform environment (per the dual ApplicationSet model). Although tenant-authored WASM runs in `ameide-{env}`, modules are treated purely as data executed by a sandboxed, Ameide-owned runtime; all host calls remain scoped by the caller’s tenant/org/user and reuse existing controller APIs, keeping the 478 namespace invariant intact.

- **Out of scope**
  - Transformation UI/UX for authoring ExtensionDefinitions (handled by EXT-WASM-03).
  - Backstage templates for tenant-tier services (covered by 478).

---

## 3. High-Level Plan

1. **Proto & SDK Foundations**
   - Author `runtime.proto` per §3.2 of 479 (ExecutionContext, InvokeExtension, responses) and keep it inside the shared Buf module (365).
   - Regenerate `ameide_sdk_go` / `ameide_sdk_ts` clients with helper functions to build contexts derived from the controller’s authenticated `RequestContext`. Services consume the new RPC only via SDKs (no direct proto imports) to stay aligned with the Buf SDKs v2 policy (365).

2. **Service Scaffold**
   - Create `services/extensions-runtime/` with README, basic main.go, health endpoints, configuration stubs.
   - Add `Dockerfile.dev` / `Dockerfile.release` mirroring controller skeleton patterns (Go multi-stage) and copying workspace SDK packages per the Ring 1 / Ring 2 rules (408).

3. **Module Management**
   - Implement `ModuleStore` that pulls WASM blobs via `wasm_blob_ref`, verifies integrity, and caches by `(extension_id, version)` with eviction policies. Objects live in the GitOps-managed MinIO deployment (`data-minio`); respect the shared ExternalSecrets defined in backlog/362 and the storage pattern documented in 479 §3.5.
   - Support refreshing config from promotion events (watch storage, SSE, or queue).

4. **Sandbox Execution**
   - Integrate Wasmtime (or WasmEdge) with per-invocation CPU/memory/time enforcement.
   - Implement host functions (`host_log`, `host_call`) with deterministic behavior.

5. **Host Bridge & Policy**
   - Enforce risk-tier allowlists for host calls.
   - Emit short-lived internal tokens bound to the incoming ExecutionContext (mirroring controller auth claims), route to allowed Ameide services via existing SDKs, and block disallowed calls.

6. **Observability & Controls**
   - Instrument metrics per invocation: latency, resource usage, error type, tenant/extension labels.
   - Add circuit-breaker hooks for SRE to disable an extension version or risk tier.
   - Follow backlog/434/441 labeling conventions (`ameide.io/environment`, `ameide.io/tier`, `ameide.io/tenant-kind`, etc.) so dashboards, policies, and ApplicationSets can slice data consistently across environments/tenants.

7. **Deployment Assets**
   - Add Helm chart (`charts/extensions-runtime`) with Deployment, Service, config (bucket refs, host-call policies), and ExternalSecret references that align with backlog/362 guardrails (no inline secrets, use `existingSecret`/ClusterSecretStore). Build artifacts respect the Ring 2 contract (workspace SDKs, no registry fetches) when published to GitOps (408).
   - Register ApplicationSet entries so each `ameide-{env}` namespace deploys the runtime. Follow backlog/465 for file placement and backlog/447 for rollout phases (Platform band, e.g., rollout-phase `350`), ensuring RollingSync steps pick it up correctly.
   - Wire CI/CD (GitHub workflow + cd-service-images entry) so every PR builds/tests the service and both Dockerfiles, and main branch builds artifacts for promotion.

---

## 4. Implementation Status (Dec 2025)

| Area | Status | Notes |
|------|--------|-------|
| **Proto & SDKs** | ✅ Complete | `packages/ameide_core_proto/src/ameide_core_proto/extensions/v1/runtime.proto` landed; Go SDK stubs regenerated and vendored via workspace copies per 365/408. TS/Python SDK sync pending the next Buf push. |
| **Service scaffold** | ✅ Complete | `services/extensions-runtime/` contains config package, sandbox executor (placeholder), MinIO-backed ModuleStore, gRPC server wiring, README, and dual Dockerfiles (workspace-first). |
| **Testing** | ✅ Mock mode, ⚠ cluster | Mock + cluster-aware integration suite (`__tests__/integration`) follows the unified runner (430). Cluster env vars still need documentation/pipeline wiring before running against `ameide-dev`. |
| **Docker build** | ✅ | `docker build` succeeds for both dev/release images; GitHub workflow `extensions-runtime.yml` exercises `go test`, integration tests (mock mode), and both Dockerfiles on PRs. |
| **Observability/telemetry** | ✅ baseline | OTEL wiring mirrors inference-gateway pattern (contextual logging, tenant interceptor, OTLP exporter). Need to add metrics + diagnostics once host-call bridge is implemented. |
| **Host-call bridge & Wasmtime** | 🚧 In progress | Current executor is a sandbox stub that echoes payloads; Wasmtime embedding, fuel/time limits, and host-call policy enforcement are slated for the next iteration. |
| **MinIO module management** | ✅ | LRU cache + singleflight loader backed by MinIO client; config struct aligned with 362 (endpoint, secure flag, bucket/prefix). Need to layer in config reload events from Transformation. |
| **Helm/GitOps deployment** | ⏳ Not started | Chart + ApplicationSet entries still to be created (blocked on Wasmtime/host-call completion). |
| **Controller integration** | ⏳ Not started | Controllers currently have no helper to invoke the runtime; SDK convenience methods + wiring into Platform/Domain/Process controllers to follow once runtime API stabilizes. |

Next mileposts:
1. Swap the placeholder executor for Wasmtime + host-call enforcement (per §3.4/3.5).
2. Emit controller-side SDK helpers and RPC client wrappers.
3. Author Helm chart + GitOps Application (platform band, rollout-phase 350).
4. Extend CI to publish images through `cd-service-images` once Helm manifests exist.

---

## 5. Acceptance Criteria

- Controllers can call `InvokeExtension` via generated SDKs and receive sandboxed results.
- The runtime enforces limits declared in `ExtensionDefinition.limits` and risk-tier host-call policies.
- Dev/release images build reproducibly in CI (Tilt + production).
- Helm chart deploys successfully to a test `ameide-dev` namespace with ArgoCD.
- Basic observability dashboards/slo alerts exist for invocation latency/error rates.
- Integration tests follow the dual-mode (mock/cluster) contract in backlog 430 (runner script, fixtures, fail-fast env validation).
- Runtime manifests include the required namespace/pod labels so networking guardrails from backlog 441 apply automatically.
- Benchmark shows `InvokeExtension` adds <10 ms p95 latency for a pure function with small payloads (baseline on `ameide-dev`), and negative sandbox tests prove denied operations (network/FS/CPU abuse) are blocked.
- Operators can disable or degrade individual extensions via config/feature flags when risk thresholds are exceeded (ties into 476 incident response expectations).

---

## 6. Open Questions / Follow-ups

1. Preferred storage backend for WASM blobs (S3-compatible bucket vs database) and caching invalidation mechanism.
2. How Transformation publishes promotion events (Webhook? Kafka?) for the runtime to consume.
3. Whether we need a lightweight CLI for simulating extensions locally for agent developers.
