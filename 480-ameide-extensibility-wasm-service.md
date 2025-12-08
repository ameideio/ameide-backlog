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
   - Regenerate `ameide_sdk_go` / `ameide_sdk_ts` / `ameide_sdk_py` clients with helper functions to build contexts derived from the controller’s authenticated `RequestContext`. Services consume the new RPC only via SDKs (no direct proto imports) to stay aligned with the Buf SDKs v2 policy (365). Go SDK now exposes `InvokeExtensionJSON` + `ExecutionContext` options so controllers avoid hand-written RPC glue.

2. **Service Scaffold**
   - Create `services/extensions-runtime/` with README, basic main.go, health endpoints, configuration stubs.
   - Add `Dockerfile.dev` / `Dockerfile.release` mirroring controller skeleton patterns (Go multi-stage) and copying workspace SDK packages per the Ring 1 / Ring 2 rules (408).

3. **Module Management**
   - Implement `ModuleStore` that pulls WASM blobs via `wasm_blob_ref`, verifies integrity, and caches by `(extension_id, version)` with eviction policies. Objects live in the GitOps-managed MinIO deployment (`data-minio`); respect the shared ExternalSecrets defined in backlog/362 and the storage pattern documented in 479 §3.5. Reference `380-minio-config-overview.md` for bucket tenancy so the runtime reuses the existing MinIO service rather than provisioning bespoke storage.
   - Support refreshing config from promotion events (watch storage, SSE, or queue).

4. **Sandbox Execution**
   - Integrate Wasmtime (or WasmEdge) with per-invocation CPU/memory/time enforcement.
   - Implement host functions (`host_log`, `host_call`) with deterministic behavior.

5. **Host Bridge & Policy**
   - Enforce risk-tier allowlists for host calls.
   - Emit short-lived internal tokens bound to the incoming ExecutionContext (mirroring controller auth claims), route to allowed Ameide services via existing SDKs, and block disallowed calls. Registry-driven host-call adapters (e.g., `transformation.get_transformation`) replace the interim HTTP shim and reuse Ameide SDK clients.

6. **Observability & Controls**
   - Instrument metrics per invocation: latency, resource usage, error type, tenant/extension labels.
   - Add circuit-breaker hooks for SRE to disable an extension version or risk tier.
   - Follow backlog/434/441 labeling conventions (`ameide.io/environment`, `ameide.io/tier`, `ameide.io/tenant-kind`, etc.) so dashboards, policies, and ApplicationSets can slice data consistently across environments/tenants.

7. **Deployment Assets**
   - **Service activities (this repo):** Add Helm chart scaffolding, ensure `Tiltfile` exposes a target for local development (workspace-first build), and wire GitHub workflows/cd-service-images entries so Docker images publish automatically. These assets belong in this repository because they impact developer workflows and guardrails (408).
   - **GitOps activities (platform repo):** Register ApplicationSet entries so each `ameide-{env}` namespace deploys the runtime. Follow backlog/465 for file placement and backlog/447 for rollout phases (Platform band, e.g., rollout-phase `350`), ensuring RollingSync steps pick it up correctly. Keep GitOps-specific changes in the GitOps repo to preserve separation of duties.
   - **Image promotion helper (ameide-core):** `scripts/dev/vendor_service_images.sh` re-tags any existing `ghcr.io/ameideio/<service>:<tag>` digest to aliases such as `:main`. Use it from this repo to unblock deployments when a pod references a tag that hasn’t been published yet (e.g., `extensions-runtime:main`).

---

## 4. Implementation Status — Service Repo (Dec 2025)

| Area | Status | Notes |
|------|--------|-------|
| **Proto & SDKs** | ✅ Complete | `packages/ameide_core_proto/src/ameide_core_proto/extensions/v1/runtime.proto` landed; Go/TS/Python SDKs expose `WasmExtensionRuntimeService`, and `scripts/dev/check_sdk_alignment.py` passes locally. Go SDK now includes `InvokeExtensionJSON` + ExecutionContext helpers. |
| **Service scaffold** | ✅ Complete | `services/extensions-runtime/` contains config package, Wasmtime executor, MinIO-backed ModuleStore, gRPC server wiring, README, and dual Dockerfiles (workspace-first). |
| **Testing** | ✅ Mock mode, ⚠ cluster | Mock + cluster-aware integration suite (`__tests__/integration`) follows the unified runner (430). Cluster env vars still need documentation/pipeline wiring before running against `ameide-dev`. |
| **Workspace-first build/test guardrails** | ✅ Verified | `Dockerfile.dev` / `.release` copy workspace SDK packages (408/405). Local `go test ./...` and mock-mode integration tests pass. |
| **SDK alignment (365 v2)** | ✅ | SDK regeneration landed (Go/TS/Py). CI now fails only if host-call integration/GitOps wiring is missing. |
| **Docker build** | ✅ | `docker build` succeeds for both dev/release images; GitHub workflow `extensions-runtime.yml` runs unit + integration tests and both Dockerfiles on PRs. CGO is enabled for Wasmtime. |
| **Observability/telemetry** | ✅ baseline | OTEL wiring mirrors inference-gateway pattern (contextual logging, tenant interceptor, OTLP exporter). Metrics/diagnostics still pending. |
| **Host-call bridge & Wasmtime** | ⚠ Initial adapters | Wasmtime executor runs modules with `alloc/dealloc/run` exports and `host_log/host_call` ABI. Registry-driven host bridge instantiates adapters via the Ameide SDK (first adapter: `transformation.get_transformation`). Need token minting + additional adapters. |
| **MinIO module management** | ✅ | LRU cache + singleflight loader backed by MinIO client; config struct aligned with 362. Need config reload events from Transformation. |
| **Tilt target & Helm scaffolding** | ⏳ Not started | Need Tilt resource + chart skeleton in this repo so dev workflows can build/test locally; blocks `cd-service-images` wiring. |
| **Release packaging (cd-service-images)** | ⚠ In progress | `.github/workflows/cd-service-images.yml` now includes `extensions-runtime`, but we still need to verify GitHub environments/secrets for promotion and document the workflow alongside other services. |
| **Controller integration** | ⚠ Helper implemented | A mock-mode integration test spins up Wasmtime + host-call adapter, proving controllers can invoke extensions through the helper. Need to wire a real Domain/Process/Agent controller path and add negative sandbox tests. |

### 4.1 Implementation Status — GitOps Repo

| Area | Status | Notes |
|------|--------|-------|
| **ApplicationSets & rollout phase** | ⏳ Not started | Must add Helm chart + ApplicationSet entries per environment, apply rollout phase ≈350, and ensure RollingSync picks it up alongside other platform services. |
| **Secrets & hydration** | ⏳ Not started | Need ExternalSecret references for MinIO credentials + host-call adapters, plus a config sync path for Transformation promotions. |
| **Observability/SLO wiring** | ⏳ Not started | Dashboards, runtime labels, and alert policies must be registered in GitOps alongside deployment manifests. |

Next mileposts:
1. **Service repo:** Finish HostBridge policy work (token minting, additional adapters), wire a real controller path via the new SDK helper, add Tilt target + Helm scaffolding, and extend negative sandbox tests.
2. **GitOps repo:** Create Helm/ApplicationSet assets with rollout metadata, wire ExternalSecrets + config hydration jobs, and document runtime SLO dashboards/disable switches.

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
