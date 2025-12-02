# 311: AI Platform Implementation Plan

**Status**: Draft  
**Priority**: High  
**Complexity**: X-Large

---

## 1. Goal

Stand up a production-grade AI platform composed of:

1. **`service/agents`** – new control-plane service that owns catalog metadata, versioned agent/tool definitions, routing rules, and CRUD/publish workflows (spec locked in [310-agents-v2](310-agents-v2.md)).
2. **`service/agents_runtime`** – successor to the existing Python inference runtime, refactored around registry-driven LangGraph compilation and gRPC parity (scope defined in [309-inference](309-inference.md)).

This plan sequences the work across environments (local → staging → production) and highlights dependencies, migrations, and validation checkpoints.

---

## 2. Workstreams Overview

| Workstream | Summary | Key Outputs |
|------------|---------|-------------|
| WS-A: Proto & SDKs | Define `agents.proto`, generate Go/Python/TS clients, set up Buf pipeline | Proto repo update, generated packages (`ameide_core_proto`, `ameide_sdk_ts`, `ameide_core_proto_python`) |
| WS-B: Agents Service | Bootstrap `service/agents` (Go), implement catalog/routing APIs, persistence, and eventing | Connect service, Postgres migrations, SSE/gRPC streaming, admin UI endpoints |
| WS-C: Inference Runtime | Create `service/agents_runtime` repo target, migrate LangGraph runtime to registry-driven design, expose gRPC APIs, register Temporal activities | New FastAPI app, registry cache, routing engine, tool DI, gRPC `Invoke`/`Stream`, Temporal worker |
| WS-D: Gateways & Clients | Update inference gateway to call new gRPC APIs, upgrade web UI + workflows clients to generated SDKs, add routing sandbox, route between direct and Temporal execution paths | Gateway release, Next.js agent designer integration, Temporal job adaptations |
| WS-E: Migration & Ops | Data migration from legacy threads configs, rollout strategy, observability, SLOs, runbooks | Migration scripts, feature flags, dashboards, incident playbooks |

---

## Progress Summary (2025-02)

- **Phase 0 (Proto & SDKs)** – `agents.proto` ships in `packages/ameide_core_proto`, and generated Go/Python/TS SDKs are in the repo; end-user documentation for consuming the SDKs is still missing.
- **Phase 1 (Agents service)** – CRUD, routing, and publish flows are live; artifacts upload to MinIO. Compiler records a `plan.v1` snapshot (no LangGraph DAG yet), catalog streams node+agent snapshots, and the container image bundles integration tests consumable via Tilt/Helm. Tool grants lack enforcement, and domain events are absent.
- **Phase 2 (Inference runtime)** – gRPC `Invoke`/`Stream` endpoints and routing fallback are implemented; runtime now hydrates the precompiled LangGraph DAG emitted by the control plane (including middleware stacks and tool wiring) and emits `content` events. Shared cache eviction remains open.
- **Phase 3 (Gateway & clients)** – Gateway proxies the new gRPC APIs; Next.js agent APIs now flow the active tenant ID end-to-end, the designer/test console/routing sandbox call the Agents & Inference gRPC services directly via the generated SDK, integration packs expose optional publish/invoke smoke tests, and logging in the inference integration suite now surfaces connection/provisioning details. The designer now emits save/publish toasts, the test console supports cancellation, the property inspector renders collection/callout schemas with dedicated components, and workflows/automation hooks consume the generated SDK with tenant-aware requests. Remaining gaps: runtime streams still need middleware-derived events, and component/automation parity requires dedicated coverage.

> **Action:** land shared executor cache eviction, stream agent-instance deltas, and emit publish/routing events before attempting migration.

---

## 3. Detailed Phases

### Phase 0 – Foundations

- Finalize `agents.proto` (nodes, versions, routing, tool grants).
- Verify proto covers n8n property flags (callouts/notices, `noDataExpression`), node wiring (`inputs`, `outputs`, `hints`), and tool metadata (`sourceNodeName`, `isFromToolkit`).
- Configure Buf generation targets for:
  - Go (Connect server/client stubs for Agents service + gateway).
  - Python (Pydantic/Grpc tools for inference).
  - TypeScript (Connect-Web + Zod schema wrappers for UI).
- Add CI jobs enforcing breaking-change checks and codegen freshness.

**Exit Criteria**
- [x] Proto approved & merged. _(met – see `packages/ameide_core_proto/src/ameide_core_proto/agents/v1/agents.proto`.)_
- [x] Generated packages published (npm, PyPI internal, Go module). _(met – codegen under `packages/ameide_sdk_ts`, `packages/ameide_sdk_python`, `packages/ameide_sdk_go`.)_
- [ ] Documentation for consuming generated SDKs, including mapping from n8n node definitions (e.g., [`AgentV3`](../reference-code/n8n/packages/@n8n/nodes-langchain/nodes/agents/Agent/V3/AgentV3.node.ts#L21), [`ToolCalculator`](../reference-code/n8n/packages/@n8n/nodes-langchain/nodes/tools/ToolCalculator/ToolCalculator.node.ts#L21)).

### Phase 1 – Agents Service (Control Plane)

1. Scaffold `service/agents` (Go, Connect, Postgres).
2. Implement modules:
   - Node catalog CRUD (`agents.nodes`, `agents.node_versions`).
   - Agent instance lifecycle (`agents.instances`).
   - Routing rules storage + evaluation (`agents.routing_rules`).
   - Tool bindings (`agents.agent_tools`) that persist parameters + metadata needed to rebuild n8n toolkits.
   - Tool grants management (`agents.tool_grants`).
   - Publish pipeline that compiles agent instances into LangGraph DAG artifacts via `langchain.agents.create_agent`, captures middleware + `content_blocks` schemas, serializes the DAG into the artifact payload, uploads the bundle to MinIO (S3-compatible), and persists the content hash/URI timestamps on the agent record.
3. Expose APIs:
   - `ListNodes`, `StreamCatalog`, `CreateAgent`, `UpdateAgent`, `PublishAgent`, `ListAgents`.
   - Routing simulation endpoint (`SimulateRoute` – optional Connect method).
4. Admin CLI or seed scripts to import initial agent/tool definitions.
5. Emit domain events (`agent.instance.published`, `agent.routing.changed`).

**Exit Criteria**
- [x] Service running locally with Tilt. _(met – service deployed and publish exercised end-to-end.)_
- [ ] Integration tests covering catalog streaming & CRUD. _(basic CRUD tests exist; add LangGraph compilation, artifact assertions, and agent-delta streaming coverage.)_
- [ ] Observability (metrics + logs) wired to platform stack. _(pending – no structured metrics/events/domain events emitted.)_

### Phase 2 – Inference Service V2

1. Create `service/agents_runtime` package (Python, FastAPI + gRPC).
2. Implement runtime pieces (per [309-inference](309-inference.md)):
   - Catalog client + registry cache.
   - Routing engine (priority conditions).
   - LangGraph builder pipeline (prompts, tools, memories).
   - Middleware + context schema wiring that mirrors `create_agent` behaviour (PII redaction, summarization, human-in-the-loop) and forwards standard `content_blocks` through gRPC streaming.
   - Artifact hydrator that fetches compiled DAGs from MinIO via `compiled_artifact_uri` / `compiled_artifact_hash`, warms a local cache, and recompiles only on cache miss or hash mismatch.
   - Tool runtime with DI that emits LangChain `Tool` objects carrying `metadata.sourceNodeName` / `metadata.isFromToolkit` (see [`getConnectedTools`](../reference-code/n8n/packages/@n8n/nodes-langchain/utils/helpers.ts#L201)).
   - Invocation envelope mapping `InvocationContext` (`tenant_id`, `ui_page`, `user_role`, graph/selection flags, custom metadata) into the routing engine and LangGraph execution context to stay in lock-step with n8n’s `IExecuteFunctions`.
   - Streaming soak tests that validate back-pressure and event ordering with slow consumers.
3. Provide gRPC APIs:
   - `Invoke` (unary) for sync calls.
   - `Stream` (server streaming) for SSE/WebSockets.
4. Maintain REST/SSE compatibility via adaptors.
5. Register a Temporal activity worker that wraps the gRPC execution path (heartbeats, retries, incremental progress) so workflows can execute agents without duplicating runtime logic.
6. Define and enforce the `AgentStepOutput` schema returned by activities (status, plan, diagnostics, final payload) using LangChain structured output.
7. Add instrumentation (Langfuse, Prometheus) tagging direct vs. Temporal-triggered runs and recording chunked streaming metrics.

**Exit Criteria**
- [ ] Legacy hardcoded configs removed. _(registry client added but legacy helpers still in tree.)_
- [ ] Published agents hydrate compiled artifacts from MinIO (hash verified, cache fallback exercised). _(runtime downloads artifact per request; no cache or LangGraph hydration yet.)_
- [x] gRPC smoke tests pass (gateway still on REST temporarily). _(Invoke/Stream exercised via integration tests; functionality limited to echo runtime.)_
- [ ] Feature flags allow toggling between legacy and new registries. _(flag wiring pending.)_
- [ ] Streaming emits tokens, `content_blocks`, and tool events aligned with LangChain v1 semantics. _(blocked on runtime echo implementation.)_
- [ ] Temporal activity path delivers identical outputs/telemetry as direct gRPC calls, including chunked signal/update streaming and `AgentStepOutput` schema validation (verified in integration tests).

### Phase 3 – Gateway & Client Integration

- Update inference gateway:
  - Use Connect-Go client to call Agents service for routing metadata.
  - Proxy requests to new gRPC endpoints; maintain REST fallback (`/v1/stream` kept for compatibility).
  - Choose between direct gRPC execution and Temporal workflows orchestration per request (sync threads vs. intelligent process) and surface status APIs for both paths (see [305-workflows](305-workflows.md) for orchestration guardrails).
- Update UI:
  - Switch to generated TS SDK for agent catalogs. _(Palette, inspector, and catalog tables now source definitions via Connect-Web.)_
  - Integrate agent designer with Agents service. _(Session/save/publish/test console use Agents & Inference gRPC clients.)_
  - Ensure tenant context flows through designer/editor APIs (Next.js routes now derive tenant ID from org context and forward it to `publish`/`invoke`; integration runner exposes optional smoke tests gated by `AGENTS_TEST_*` env vars). _(done)_
  - Add routing sandbox using `SimulateRoute`. _(Designer toolbar exposes sandbox dialog backed by AgentsService.)_
  - Implement PropertyType → component registry with JSON-editor fallback for unknown/custom types. _(Inspector handles collection/notice/credential/custom types with audited fallbacks.)_
  - Surface save/publish feedback in the designer and support cancelling inflight test runs. _(Sonner toasts wired for save/publish; test console aborts via Connect cancellation.)_
  - Surface workflows-backed executions (status, approvals) alongside direct runs in the console.
- Update Temporal/workflows jobs to use new SDKs and the shared agent activity.

**Exit Criteria**
- [x] Gateway streaming via gRPC enabled in staging. _(gateway forwards Invoke/Stream to inference service)_
- [ ] UI & automation clients using generated SDKs with callout/notice rendering parity. _(Agent designer/console/sandbox and workflows hooks now use the generated SDK with tenant context; collection/callout renderers ship in the inspector. Pending Storybook coverage and any remaining automation surfaces.)_
- [ ] Routing sandbox shows correct agent selection in staging. _(requires UI wiring to consume new runtime event shape)_
- [ ] Component registry covers all PropertyTypes in catalog (Storybook snapshot + automated tests). _(component audit outstanding)_
- [ ] Temporal workflows successfully invoke agents end-to-end with observable progress + correlation IDs, respecting chunked token updates and idempotent connector activities.

### Phase 4 – Migration & Rollout

1. Snapshot legacy threads configs; map to agent instances + routing rules; import via Agents service.
2. Enable inference service registry mode in staging; verify agents/tools operate.
3. Run compatibility suite (existing tests + new Playwright flows).
4. Production rollout:
   - Enable Agents service + gRPC streaming behind feature flag per tenant.
   - Monitor metrics (`agent_latency_seconds`, error rates).
5. Decommission legacy REST pathways once stable.
6. Publish operator/developer runbooks (agent creation guide, routing sandbox tutorial).

**Exit Criteria**
- [ ] All tenants on new registry; legacy configs deleted.
- [ ] Runbooks & dashboards validated.
- [ ] Post-rollout review complete.

---

## 4. Dependencies & Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Proto churn during build-out | Gateway/UI integration blocked | Lock proto before Phase 1; use feature branches + Buf breaking checks. |
| Migration gaps (missing agent metadata) | Agents fail post-cutover | Build migration tooling early; dry-run in staging; maintain legacy fallback until completed. |
| gRPC performance issues | Streaming regressions | Load test new inference gRPC endpoints; tune gateway pools; add back-pressure metrics. |
| Tool DI misconfiguration | Runtime errors | Provide default dependency pool, exhaustive tests per tool, staged rollout. |
| Observability gaps | Difficult to debug issues | Integrate Langfuse/Prometheus from Phase 2, require dashboards before prod. |
| Streaming compatibility | Legacy clients break on new event schema | Version stream payloads (`/v1/stream` vs `/v2/stream`), provide adapter in gateway. |
| Routing rule conflicts | Ambiguous agent selection | Enforce unique priority per tenant, add validation + dry-run tooling. |
| Property schema expansion | UI unable to render new property types | Implement fallback renderer (JSON editor), ship component tests for each `PropertyType`. |
| Artifact storage misconfiguration | Publish succeeds but inference cannot load LangGraph graphs | Automate MinIO bucket creation, add health check that verifies read/write during startup, block publish if artifact upload fails. |

---

## 5. Timeline (Indicative)

| Milestone | Target |
|-----------|--------|
| Proto & SDKs complete | Week 1-2 |
| Agents service MVP (catalog streaming) | Week 4 |
| Inference service V2 functional (registry + gRPC) | Week 6 |
| Gateway/UI integration in staging | Week 8 |
| Production rollout (phased) | Week 10-12 |
| Contingency buffer (UI/streaming hardening) | Week 12-14 |

*Actual schedule depends on team allocation and parallel workstreams; the buffer is reserved for property-rendering polish, load tests, or migration surprises.*

---

## 6. Success Metrics

- 100% of agent/tool definitions served from Agents service (no static configs).
- Inference gateway uses gRPC by default; REST only for legacy clients.
- Mean streaming latency ≤ current baseline; error rate < 0.5%.
- Agent designer -> publish -> inference round-trip in < 5 minutes without deployments.
- Comprehensive logs/traces per execution with agent instance IDs.

---

## 7. References

- [309-inference.md](309-inference.md) – detailed inference runtime redesign.
- [310-agents-v2.md](310-agents-v2.md) – agents service architecture.
- n8n LangChain code (`reference-code/n8n/packages/@n8n/nodes-langchain/`).
- Current inference gateway graph (`services/inference_gateway`).

---

**Author**: AI Platform Working Group  
**Last Updated**: 2025-02-10
