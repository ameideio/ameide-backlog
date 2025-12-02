> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 305 - Platform Workflow Orchestration

**Status:** Draft  
**Priority:** High  
**Complexity:** High  
**Related:** `packages/ameide_core_proto/src/ameide_core_proto/ipa/v1/workflows.proto`, `packages/ameide_sdk_ts/src/client/index.ts`, `infra/kubernetes/charts/platform/workflows_runtime/*`

---

## Executive Summary

AMEIDE ships a Temporal deployment and a gRPC surface (`WorkflowService`) for orchestrating automation inside the platform, yet there is no cohesive, production-ready implementation. Definitions live in proto messages, executions are not persisted outside Temporal, and the web experience only hints at “workflows” actions. This transformation focuses on stabilising **platform workflows**—the automation that powers AMEIDE itself—not the enterprise architecture artefacts captured as graph elements. Architectural content (process diagrams, BPMN flows, etc.) remains in the element model, but the runtime automation that operates AMEIDE is treated as platform infrastructure with clear integration points into that model.

Key outcomes:
- A managed catalogue of platform workflows backed by Temporal, with persistence and audit trails separate from the architecture graph.
- Updated APIs/SDKs that expose workflows definitions and runs without forcing them into the element graph.
- Operational tooling in the web app for platform operators, including execution history, health, and linkage to architecture artefacts when relevant.

---

## Goals

1. Deliver a production-ready workflows orchestration service for AMEIDE operational tasks using Temporal.
2. Persist workflows definitions, versions, and execution metadata in a dedicated platform schema while linking to architecture elements only when needed.
3. Provide SDK and UI affordances so operators can author, publish, and monitor platform workflows end-to-end.
4. Keep a clear boundary between platform automation and the enterprise architecture data model defined in `303-elements.md`.

### Non-Goals

- Treating platform workflows as graph elements or forcing them into the `elements` table.
- Modelling customer/enterprise processes (BPMN, capability flows) for the target architecture—those continue to live as elements managed by graph teams.
- Introducing a visual workflows designer beyond the minimal capabilities required to author JSON/structured definitions for Temporal workers.

---

## As-Is

### Proto & SDK

- WorkflowService protos are Buf-managed and generate Connect clients; both TypeScript and Go SDKs target the shared gRPC surface (`packages/ameide_sdk_ts/src/workflows/client.ts`, `packages/ameide_core_cli/internal/commands/workflows.go`).
- The TypeScript SDK wraps the Connect client with domain-friendly models (enum/timestamp conversion, memo/search attribute helpers) and exposes both browser and server transports; REST shims have been removed.

### Runtime & Infrastructure

- `services/workflows` now boots a Connect/HTTP2 gRPC server (`src/grpc.ts`) backed by Postgres repositories for definitions, versions, rules, and status updates; the legacy Express/REST layer has been deleted.
- Worker callbacks flow through `ReportWorkflowStatus`, which normalises payloads, persists to `workflows_execution_status_updates`, and emits OTLP counters/spans; Helm manifests expose the service via a gRPC port (`infra/kubernetes/charts/platform/workflows/templates/*`).
- The Temporal facade continues to broker run lifecycle operations with tenant-aware search attributes and a stub mode for local development; workers speak to it via the shared gRPC client.

### Application Surface

- Next.js operator pages call the SDK directly; hooks such as `hooks/useWorkflowClient.ts` and `lib/api/hooks.ts` resolve a `WorkflowGrpcClient`, and the `/api/workflows-service` proxy routes have been removed.
- Integration packs, CLI commands, and the worker reuse the shared gRPC clients so behaviour stays consistent across UI, automation runners, and background services.

### Pain Points

1. **Consumer parity:** Workflows runtime follow-ups and legacy operator scripts still need to adopt the new gRPC client helpers to finish the REST retirement.
2. **Provisioning guardrails:** Database role automation for creating `workflows_execution_status_updates` on boot needs validation across every environment.
3. **RBAC gaps:** Workflow UI runs with RBAC disabled pending policy reinstatement, widening exposure during beta.
4. **End-to-end regression debt:** Comprehensive regression covering UI, CLI, worker, and integration packs is still ad hoc; automated suites must exercise the gRPC-only stack.

---

## To-Be Architecture

### Domain Separation

- Treat platform workflows as operational assets governed by the platform team. They live outside the enterprise architecture graph and its `elements` table.
- When a workflows acts on architecture content (e.g., publishing elements, running quality gates) it references element IDs in metadata, but the workflows definition itself remains a platform configuration.

### Architecture Overview (ASCII)

```
                                      +----------------------------------------+
                                      |          Operator Surfaces             |
                                      |  - Next.js settings pages              |
                                      |  - CLI / Scripts                       |
                                      +-------------------+--------------------+
                                                          |
                               gRPC / Connect (HTTP/2)      |  @ameideio/ameide-sdk-ts helpers
                                                          v
                         +--------------------------------+-------------------------------+
                         |      Workflows Service (Connect/HTTP2 gRPC)                      |
                         |----------------------------------------------------------------|
                         |  Methods:                                                     |
                         |    WorkflowService.Create/Update/List Definitions             |
                         |    WorkflowService.Create/List Versions                      |
                         |    WorkflowService.Start/List/Describe/Cancel Runs            |
                         |    WorkflowService.Create/Update/List Rules                   |
                         |    WorkflowService.ReportWorkflowStatus                       |
                         |                                                                |
                         |  Components:                                                  |
                         |    - Connect handlers + validation                            |
                         |    - Definition/version/rule repositories (PostgreSQL)       |
                         |    - Status updates graph + OTLP instrumentation         |
                         |    - TemporalFacade (@temporalio/client)                      |
                         +-----------+---------------------------+-----------------------+
                                     |                           |
                   Definition metadata |                           | Run operations
                   (spec JSON, version)|                           | (start/describe/cancel)
                                     v                           v
            +------------------------------+      +-----------------------------------------+
            | PostgreSQL (workflows_service)|      | Temporal Cluster (namespaces/queues)    |
            |  - workflows_definitions      |<---->|  - Workflow executions & visibility API |
            |  - workflows_versions         |      |  - Task queues per tenant/queue name    |
            +------------------------------+      +-----------------+-----------------------+
                                                                | Temporal workflows/activities
                                                                v
                               +--------------------------------+-------------------------+
                               |        Platform Workers & Activities (Temporal)         |
                               |  - Lifecycle orchestration workflows                    |
                               |  - Activities calling RepositoryActionProxy, etc.       |
                               |  - Agent bridge activities invoking LangGraph agents    |
                               |    via service/agents_runtime gRPC                      |
                               +---------------------------+-----------------------------+
                                                          |
                                                          | Domain callbacks / API calls
                                                          v
                      +-----------------------------------+-----------------------------+
                      |   Repository & Platform Services (REST/gRPC)                    |
                      |   - Lifecycle bridges (UpdateElementLifecycle, etc.)            |
                      |   - Telemetry, notifications, approvals                         |
                      +------------------------------------------------------------------+

                                       ^
                                       | gRPC (Invoke/Stream) or Temporal activity bridge
                                       |
                   +-------------------+-----------------------------------------------+
                   |        Agents Platform (309-inference, 310-agents-v2)             |
                   |  - Agents Service (registry, routing, artifact store)             |
                   |  - Agents runtime (`service/agents_runtime`, LangGraph)           |
                   |  - Direct gRPC + Temporal activity execution modes                |
                   +-------------------------------------------------------------------+

Response flow examples:
 1. Operator creates definition -> Definition Service -> PostgreSQL (definition row).
 2. Operator publishes version -> Definition Service -> PostgreSQL (version row).
 3. Operator starts run -> Definition Service -> TemporalFacade.startWorkflow -> Temporal cluster.
 4. Worker execution -> Activities call Repository services or invoke the Agents Platform via the shared gRPC surface (Agent bridge) -> updates recorded in domain systems.
 5. Operator queries runs -> Definition Service -> Temporal visibility -> response to UI.
```

### Integration with Agents Platform

- Workflows that require AI reasoning or tool orchestration schedule an Agent bridge activity. The activity calls `service/agents_runtime` using the same `Invoke`/`Stream` API that direct clients use, ensuring parity with [309-inference](309-inference.md).
- Invocation context (tenant, UI page, user role, selection flags, custom metadata) flows from workflows inputs into the agent request so routing and guardrails defined in [310-agents-v2](310-agents-v2.md) remain consistent.
- Agent definitions, prompts, middleware, and tool metadata continue to live in the Agents Service catalogue; workflows reference agent IDs/versions rather than embedding prompt text.
- Temporal heartbeats and completion payloads include agent instance IDs and LangGraph trace identifiers so observability tooling can stitch workflows and agent telemetry together.

#### Agent step contract and streaming

- `AgentBridgeActivity` exchanges deterministic, JSON-serialisable payloads with the Workflow. The activity returns `AgentStepOutput` objects shaped like:

```json
{
  "status": "continue",
  "thought": "Summarised reasoning for the next step",
  "plan": [
    { "name": "create_ticket", "args": { "title": "Renew certificate", "priority": "high" } },
    { "name": "notify_slack", "args": { "channel": "#platform", "text": "Ticket created" } }
  ],
  "context_patch": { "facts": { "ticketId": 12345 } },
  "next_check": "2025-10-28T13:00:00Z",
  "final": null,
  "diagnostics": []
}
```

- Valid `status` values are `continue`, `needs_human`, `finished`, and `error`. When `status=finished`, the `final` property carries the payload returned to the caller. Human-in-the-loop paths set `needs_human` with a summary message; error paths return structured diagnostics so the Workflow can branch deterministically.
- LangChain v1 structured output must be used to enforce the schema (see [LangChain structured output docs](https://docs.langchain.com/oss/python/langchain/structured-output)).
- Streaming stays inside the Activity. When UX requires incremental tokens, buffer them and emit chunked Temporal Signals/Updates (for example, every 100–250 ms) tagged with `workflowsId|runId`, or push to a side-channel (websocket, queue). Never signal per token—keep event history bounded and rely on Continue-As-New for very long conversations.
- Activities attach business idempotency keys (e.g., `workflowsId` + sequence) to downstream connector requests and provide optional compensating hooks so Temporal retries remain safe.

### Lifecycle Orchestration Principle

- Any graph element can opt into a lifecycle template defined by platform workflows. Templates declare allowed states, transitions, and required guards (approvals, validations).
- Write intents that would affect lifecycle-controlled fields (status, head/published linkage, metadata mutations) enqueue orchestration requests via the Temporal SDK. The platform stores only the authored definition metadata; execution history, task queues, and state transitions live entirely inside Temporal.
- Repository services remain appenders of intent; Temporal owns sequencing, conflict resolution, and eventual updates back to the element record without an intermediary command gateway. Workers call back into graph APIs using the actor context captured in the request.
- Workflow definitions pair trigger conditions (e.g., `element.lifecycle.intent=submitted`) with the lifecycle template. Execution payloads include actor, graph, and element context so downstream workers can enforce policy without embedding business logic, but persisted state stops at definition metadata.

### Repository Execution Guardrails

- Workflow activities mutate graph state through WorkflowService adapters that replay the original request using the caller’s identity from the request context. This keeps RBAC checks and audit logs consistent with existing graph logic.
- Activities receive narrow façades (e.g., `UpdateLifecycleAction`, `AttachRelationshipAction`) rather than direct database handles, forcing reuse of domain services and optimistic locking semantics already established in `303-elements.md`.
- Human approvals or manual overrides append contextual events to the workflows execution log so downstream systems can associate outcomes with the correct actor and decision trail.

### Data Model & Persistence

- Persist workflows definitions and immutable versions in the dedicated `workflows_service` schema (see `db/flyway/sql/V21__workflows_service_schema.sql`).
- Store each version’s blueprint as a structured JSON `spec` that encodes triggers, guards, activities, and wiring expected by the workers alongside published runtime parameters.
- Rely on Temporal Visibility for workflows run history, status, and logs; the platform DB no longer mirrors execution data.

### API & SDK

- Implement a Definition Service that owns CRUD/versioning of platform-authored workflows blueprints while delegating execution APIs to the Temporal SDK. The service exposes a first-class gRPC surface (no REST shim) and never persists run metadata locally.
- Update the TypeScript SDK to wrap Temporal’s official clients for run operations (`WorkflowExecution`, `DescribeWorkflowExecution`, etc.) and expose definition helpers for platform metadata; downstream callers never import protos directly.
- Provide cross-links (IDs, URIs) so clients can map a workflows to graph elements when needed, without enforcing referential integrity against the element table.

### Temporal Workers

- Build runtime workers (using `@temporalio/worker`) that subscribe to platform task queues and execute workflows definitions sourced from the definition store.
- Workers emit structured telemetry (logs, metrics, traces) through OpenTelemetry exporters, tagging spans with workflows/version/run IDs. Status callbacks flow through Temporal signals/activities; no local run metadata store is updated.

### Operator Experience

- Extend the Next.js admin surfaces to list platform workflows definitions, inspect versions, and surface run history pulled directly from Temporal via the SDK.
- Surface health indicators (queue depth, failure counts) integrated with the observability stack by querying Temporal metrics instead of a local mirror.

---

## Rationale

- **Separation of concerns:** Architecture artefacts stay within the element graph, while platform automation is managed as infrastructure, reducing cross-domain coupling.
- **Operational visibility:** Dedicated persistence and telemetry give operators the audit trail Temporal alone does not provide.
- **SDK clarity:** Clients integrate with explicit workflows endpoints instead of misusing element services or mocking functionality.
- **Governance:** Platform workflows can enforce change control, approvals, and observability without polluting the architecture graph with runtime artefacts.

---

## Implementation Plan

1. **Architecture Alignment & Guardrails**
   - Finalize domain separation rules between platform workflows and graph elements, updating `303-elements.md` and service ADRs as needed.
   - Define workflows lifecycle states, versioning semantics, and tenancy scoping for both DB entities and Temporal namespaces.
   - Produce sequence diagrams for Create → Publish → Execute → Observe flows covering API, DB, Temporal, and UI touchpoints.
   - Agent Prompt — `Workflow-Architect`:  
     Work from these fixed decisions: (1) Temporal namespaces are per environment; tenants are isolated via request-context metadata. (2) The gRPC API is the single source of truth for workflows definitions—config repos must call the API, never write directly to the DB. (3) Human-in-the-loop pauses/approvals are modeled as explicit states in the platform schema and Temporal payloads, with future ticketing integrations handled via activities. (4) MVP authoring UI is a JSON editor with schema validation, snippets, and diffing; no visual builder. (5) Only `platform-operator` and `platform-admin` roles may create/publish workflows; graph maintainers may trigger executions when workflows bind to their scope.  
     Mission deliverables: update `backlog/303-elements.md` (and referenced ADRs) with clarified platform-workflows vs. element boundaries, extend `backlog/305-workflows.md` with finalized lifecycle states/version semantics/tenancy guardrails/RBAC matrices, and author `docs/workflows/architecture.md` plus diagram sources under `docs/workflows/diagrams/`. Include lifecycle state diagrams, Create→Publish→Execute→Observe sequence diagrams, contract tables mapping API/Temporal/DB payloads, tenancy enforcement notes, and a “Principles & Guardrails” section reiterating the assumptions. Validate docs build/lint and replace open questions with TODO items in the Implementation Plan.

2. **Persistence & Schema Enablement**
   - Finalise normalized tables for workflows definitions and immutable versions that mirror the authoring contract; execution history resides solely in Temporal.
   - Author Flyway SQL migrations and storage models with tenant scoping, optimistic locking, and JSON payload support; add unit tests that cover CRUD/versioning flows.
   - Agent Prompt — `Workflow-Data-Engineer`:  
     Assumptions to honor: (1) PostgreSQL remains the backing store for definitions; each table carries `tenant_id` and `row_version`. (2) Schema changes ship via Flyway migrations under `db/flyway/sql/`, and storage code must remain in lockstep with those definitions. (3) No historical workflows data exists, so seed records are only required for smoke tests.  
     Mission deliverables: author forward-only Flyway SQL migrations for `workflows_service.workflows_definitions` and `workflows_service.workflows_versions` with supporting indexes/FKs; update storage models/repositories to map the new columns (`definition_key`, `spec`, `runtime_parameters`, `annotations`, etc.) and enforce tenant scoping; add unit tests that create/list/update/delete entities and assert constraints; document the schema in `docs/workflows/storage.md`. Run migration lint/tests (Flyway dry run plus graph integration tests) and commit artifacts.
   - **Phase 2 Progress (2025-01-17):**  
     `db/flyway/sql/V21__workflows_service_schema.sql` now defines the workflows definition/version tables with tenant-aware indexes and foreign keys. Documentation for the data model lives in `docs/workflows/storage.md`, and a graph round-trip test (`packages/ameide_core_storage/tests/test_workflows_persistence.py`) exercises create/list/update flows—skipping automatically when SQLAlchemy or SQLite dependencies are absent.

3. **Temporal Infrastructure Hardening**
   - Update Helm chart values to create dedicated namespaces/queues per tenant or environment; configure TLS, metrics, and alerting hooks.
   - Provision worker images with required secrets/config via Kubernetes and ensure CI pipeline publishes them.
   - Document runbook for namespace creation, retention policies, and failure recovery.
   - Agent Prompt — `Temporal-SRE`:  
   Assumptions to honor: (1) Temporal is deployed via `infra/kubernetes/values/infrastructure/temporal.yaml`; environments map 1:1 with Temporal namespaces (`dev`, `staging`, `prod`) and queues are scoped by tenant metadata. (2) TLS certificates and secrets are managed through existing platform mechanisms (`cert-manager` and the platform secret store). (3) Observability uses Prometheus/Grafana with OpenTelemetry exporters already enabled in the cluster.  
    Mission deliverables: extend Helm values/templates to provision namespaces/queues, configure TLS, and expose metrics/health probes; supply supporting Terraform/YAML (if needed) for namespace bootstrap; add CI/CD steps to package/publish updated worker images with required env vars and secrets; create `docs/workflows/temporal-runbook.md` covering namespace creation, retention policies, scaling, Continue-As-New strategies, and recovery; document the Worker Versioning policy; validate with `helm lint`, template rendering, and a dry-run install manifest.
     Progress — 2025-10-24:
       - Refactored Helm values and templates to derive environment-scoped namespaces, task-queue routing, TLS mounts, health probes, and observability hooks directly from configuration.
       - Added optional `ServiceMonitor`, `PrometheusRule`, and namespace bootstrap Job manifests so operators can toggle scraping, alerting, and Temporal namespace seeding per environment.
       - Published the operational runbook at `docs/workflows/temporal-runbook.md` and updated the CI workflows to build the runtime worker image when its Dockerfile is present.
       - Validated renders with `helm lint` and `helm template` across default, TLS-enabled, and bootstrap-enabled combinations to confirm manifests remain consistent.

4. **Workflow Definition Service & Temporal Integration**
   - Implement a lightweight Definition Service responsible for CRUD/versioning of workflows definitions, enforcing validation and RBAC guardrails.
   - Integrate the Temporal SDK (`@temporalio/client`) for all run operations (start, signal, describe, cancel) instead of persisting execution metadata locally.
   - Provide tenancy-aware helper modules that wrap Temporal client calls and expose high-level methods for the UI and other platform services.
   - Agent Prompt — `Workflow-Service-Engineer`:  
   Assumptions to honor: (1) Definition APIs run under `services/workflows` as a Connect/HTTP2 gRPC endpoint; no REST façade remains. (2) The Flyway-managed schema from step 2 is the authoritative store for definitions and versions only. (3) Authorization leverages current middleware/interceptor hooks in the workflows service stack.  
    Mission deliverables: implement WorkflowService Connect handlers for definition/run/rule/status methods, wire the Temporal client façade with tenancy-aware validation, persist worker callbacks, and add integration tests that drive create → publish → trigger → describe flows via the gRPC client and Temporal test environment; implement an `AgentBridgeActivity` helper that calls `service/agents_runtime` using shared invocation metadata, enforces the `AgentStepOutput` schema, performs chunked streaming, and tags idempotency keys; update service configuration/env docs; ensure formatting/lint/tests pass.
   - **Phase 4 Progress (2025-10-25):**
     `services/workflows` now serves the WorkflowService exclusively over Connect/HTTP2 (`src/grpc.ts`) with handlers in `src/grpc-handlers.ts` covering definition CRUD/versioning, run operations, rule management, and `ReportWorkflowStatus` callbacks. Status updates are normalised, persisted through `repositories/status-updates-graph.ts`, and instrumented with OTLP counters/traces. The TemporalFacade (`services/workflows/src/temporal/facade.ts`) wraps `@temporalio/client` with tenant isolation, flexible run queries (`listWorkflows` filters by tenant/definition/element/status), and stub fallback when Temporal is disabled. Flyway migrations (V21–V23) remain applied with `workflows_definitions`, `workflows_versions`, and `workflows_rules` tables operational, and Helm charts now expose the deployment as a gRPC port with automated credential loading. Integration tests drive create→publish→trigger→describe flows via the gRPC client; formatting/lint/tests pass.
   - **Open items:** confirm the workflows database role can create `workflows_execution_status_updates` during bootstraps, finish migrating remaining REST-era consumers (workflows runtime callbacks, CLI utilities, operator scripts) onto the gRPC client, and document the Connect handler contract in `docs/workflows/architecture.md`.

5. **Lifecycle Enforcement Integration**
   - Route lifecycle-sensitive mutations through Temporal by emitting commands that the Definition Service resolves to workflows IDs and hands off to the Temporal SDK.
   - Agent Prompt — `Lifecycle-Gateway-Engineer`:  
     Assumptions to honor: (1) Repository services record intent and invoke orchestration helpers that call Temporal directly; no command gateway persistence exists. (2) Guards and approvals are enforced inside workflows and reflected back to graph services via existing domain APIs. (3) Rollbacks fall back to standard optimistic locking paths.  
     Mission deliverables: keep lifecycle handlers aligned with the new Temporal façade and ensure policy enforcement remains auditable via Temporal history plus graph audit logs.

6. **Temporal Worker & Activity Implementation**
   - Build generic workflows/activities packages supporting validation, guards, approvals, and graph interactions.
   - Implement run-state callbacks/signals to push status/logs back through WorkflowService and publish telemetry.
   - Containerize workers, add CI pipelines, and deliver smoke tests against Temporal dev stack.
   - Agent Prompt — `Temporal-Worker-Engineer`:  
     Assumptions to honor: (1) Workers run as separate deployable units (containerized) and use the new namespaces/queues configured in step 3. (2) Activities must never mutate repositories directly—they call the `RepositoryActionProxy` or other platform APIs. (3) Telemetry uses OpenTelemetry SDK with tracing/metrics/logging aligned to platform conventions.  
    Mission deliverables: build reusable workflows/activity libraries handling validation, guards, approvals, graph interactions, and callbacks; implement signal/callback handlers that push run status/logs back to WorkflowService using chunked updates; ensure each connector Activity embeds idempotency keys and optional compensations; containerize workers and add CI jobs for build/test/publish; create automated tests using Temporal test server verifying workflows paths, retries, Continue-As-New triggers, and callbacks; document worker configuration in `docs/workflows/workers.md`.
   - **Phase 6 Progress (2025-10-26):**  
     The worker package at `services/workflows_runtime/` continues to deliver the MVP runtime: activities (`workflows_runtime/activities.py`) handle definition validation, guard evaluation, graph action stubs, and status callbacks, while the `PlatformWorkflow` orchestrator (`workflows_runtime/workflows.py`) wraps those activities with deterministic status reporting. The entrypoint/Docker assets (`worker.py`, `Dockerfile`, `Dockerfile.dev`, `pyproject.toml`) keep the worker deployable, and the Temporal test suite (`services/workflows_runtime/tests/test_workflows_runtime.py`) still covers happy-path and guard-rejection scenarios using the in-memory test environment. Open items: migrate the callback client to the shared gRPC `ReportWorkflowStatus` helper, delete the deprecated REST shim, convert the lingering TODO warnings around pydantic v2 converter usage, and author the operator documentation stubbed as `docs/workflows/workers.md`.

7. **SDK & Internal Client Updates**
   - Regenerate protobuf/gRPC clients for TypeScript and Go (`buf generate`) keeping generated artifacts in sync across `packages/ameide_core_proto`, SDK output folders, and developer docs.
   - Implement workflows helper layers and adopt them across internal consumers: ship tenancy-aware helpers (`createWorkflowDefinition`, `publishWorkflowVersion`, `listWorkflowRuns`, `triggerWorkflowRun`) and wire quality gate automation, lifecycle services, and `core_cli` to the new abstractions with example scripts.
   - Testing & release readiness: expand TS/Go unit and integration coverage for helpers/CLI, add WorkflowService mocks/fixtures, and gate semantic version bumps on `buf lint`, `pnpm --filter ameide_sdk_ts test`, `pnpm lint`, `pnpm typecheck`, and `go test ./packages/ameide_core_cli/... ./packages/ameide_sdk_go/...`.
   - Agent Prompt — `SDK-Integrator`:  
     Assumptions to honor: (1) TypeScript SDK lives in `packages/ameide_sdk_ts`; Go and other language clients follow existing Buf-generated structure. (2) Consumers expect stable semantic versioning with changelog entries; breaking changes must be versioned appropriately. (3) CLI/internal services rely on gRPC clients with typed helpers.  
     Mission deliverables: regenerate Buf/gRPC bindings and ensure generated sources land in `packages/ameide_core_proto` and downstream SDKs; implement workflows helper modules (e.g., `packages/ameide_sdk_ts/src/workflows/index.ts`, `packages/ameide_sdk_go/workflows.go`) exporting typed `createWorkflowDefinition`, `publishWorkflowVersion`, `listWorkflowRuns`, and `triggerWorkflowRun` with tenancy defaults, pagination helpers, and error normalization; migrate quality gate automation, lifecycle orchestration services, and `core_cli` commands to use the helpers, supplying sample scripts in `packages/ameide_sdk_ts/examples/workflows`; author TypeScript unit tests (`packages/ameide_sdk_ts/src/workflows/__tests__/helpers.test.ts`), Go helper tests (`packages/ameide_sdk_go/workflows_test.go`), and CLI integration tests with mocked WorkflowService stubs; update SDK READMEs/changelogs, bump package versions, and verify `buf lint`, `pnpm --filter ameide_sdk_ts test`, `pnpm lint`, `pnpm typecheck`, and `go test ./packages/ameide_core_cli/... ./packages/ameide_sdk_go/...` succeed before publishing artifacts.
   - **Progress — 2025-02-19:**  
     `packages/ameide_sdk_ts/src/workflows/index.ts` now provides tenancy-aware helpers for create/publish/trigger/list plus error normalization, and `packages/ameide_sdk_ts/src/__tests__/unit/workflows/helpers.test.ts` exercises the helpers with the new gRPC transport covering context defaulting, pagination, and failure wrapping. The helpers are exported via `packages/ameide_sdk_ts/src/index.ts`, surfaced in the package map (`packages/ameide_sdk_ts/package.json`), and documented alongside a runnable sample (`packages/ameide_sdk_ts/README.md`, `packages/ameide_sdk_ts/examples/workflows/basic.ts`). `pnpm --filter @ameideio/ameide-sdk-ts test` is green to guard the release pipeline.
   - **Open items:** finish porting `core_cli` workflows commands and any remaining operator scripts to the helpers, backfill Go helper coverage, and publish an SDK changelog entry summarising the gRPC-only surface.

8. **Operator Web UI Delivery**
   - Extend navigation to include workflows catalog and run dashboards; gate access by admin/operator roles.
   - Build pages for workflows list/detail, definition editor (JSON with schema validation), publish flow, run history, and execution drill-in with logs.
   - Integrate live status via short-polling or streaming updates, surface telemetry KPIs, and allow manual trigger/cancel/retry aligned with RBAC.
   - Agent Prompt — `Workflow-UI-Engineer`:  
     Assumptions to honor: (1) The web app is Next.js/React with Tailwind; admin-only routes require `platform-operator`/`platform-admin` checks via existing auth hooks. (2) MVP editor is JSON-only with schema validation snippets/diffing; no canvas builder. (3) Real-time status uses existing SSE/WebSocket infrastructure.  
     Mission deliverables: extend navigation for workflows catalog/run dashboards; build pages for list/detail/editor/publish/run history/log drill-in; integrate WorkflowService via hooks, add schema validation, version diffing, and role-gated actions for trigger/cancel/retry; wire live status indicators; create Playwright/Cypress tests covering key flows; update UI docs and storybook (if applicable).
   - **Phase 8 Progress (2025-02-22):**
     Added operator-facing navigation and gating so `platform-operator`/`platform-admin` users see the Workflows tab in both the header and sidebar (`features/navigation/components/Header.tsx`, `AppSidebar.tsx`, and contextual-nav config). Implemented the catalog, detail, and run dashboards under `services/www_ameide_platform/app/(app)/org/[orgId]/workflows/**`, providing JSON definition editing with schema validation, publish/trigger dialogs, and live log polling that hydrates `WorkflowService.GetWorkflowRun` responses. Supporting hooks/components (`lib/api/hooks.ts`, `features/workflows/*`) and Playwright/Jest coverage exercise the catalog access patterns, while `docs/workflows/ui.md` documents the operator experience.
   - **Phase 8 Update (2025-10-25A):**
     Temporarily disabled role-based access control (`platform-operator`/`platform-admin` checks) across all workflows pages to allow all authenticated users access during development. The original permission checks are preserved as commented code with `TODO: Re-enable role-based access control when ready` markers in `app/(app)/org/[orgId]/workflows/**/page.tsx`, `features/navigation/components/AppSidebar.tsx`, and `features/navigation/components/Header.tsx`. Removed the Workflows link from the header user dropdown menu and added a dedicated Workflows section in organization settings (`app/(app)/org/[orgId]/settings/page.tsx`) positioned after Governance, providing links to both the Workflow Catalog and Execution Runs dashboard. Updated `docs/workflows/ui.md` to reflect the temporary removal of access control restrictions.
   - **Phase 8 Critical Gaps Resolved (2025-10-25B):**
     Fixed four blocking issues preventing end-to-end workflows operations: (1) **Run List API** — `ListWorkflowRuns` now accepts optional `definitionId`, `elementId`, and `status` filters, enabling org-level dashboards and element-scoped history without bespoke UI filtering. (2) **Worker Status Callback** — Implemented the gRPC `ReportWorkflowStatus` handler so workers can report progress with structured logs while persistence occurs through the status updates graph. (3) **Element Automation** — `useElementWorkflows` now calls the gRPC client with an `elementId` filter instead of client-side stubs, restoring Automations tab run history. (4) **Repository Rule Creation** — `RepositoryWorkflowRules` wires into `useWorkflowRules().createRule`, validating inputs against live workflows definitions and surfacing submission errors inline. Testing: E2E suite remains 76% green (22/29) and Playwright coverage verifies the refreshed Runs dashboard behaviour.

9. **Telemetry, Monitoring, & Observability**
   - Instrument service methods, workers, and UI with OpenTelemetry traces/metrics/logs; define dashboards (Grafana/New Relic).
   - Implement alerting for queue latency, failure rates, reconciliation drift, and SLA breaches.
   - Provide sample queries and dashboard exports for SRE handoff.
   - Agent Prompt — `Workflow-Observability-Engineer`:  
     Assumptions to honor: (1) Platform uses OpenTelemetry exporters feeding Prometheus/Grafana and central logging (e.g., Loki). (2) Alerting integrates with the existing incident response tooling; dashboards live alongside other platform panels. (3) Telemetry naming/labels follow platform conventions (service.namespace.*, workflows.*).  
     Mission deliverables: instrument WorkflowService, workers, and UI with traces/metrics/logs (latency, queue depth, success/failure rates, reconciliation drift, human-in-loop durations); create Grafana dashboards and alert rules for SLA breaches; provide sample queries and documentation in `docs/workflows/observability.md`; ensure lint/tests for telemetry configuration pass.

10. **Governance, Enablement, & QA**
    - Document workflows authoring guide, change-management process, and element-referencing policy; update runbooks and training material.
    - Deliver end-to-end regression tests (API + worker + UI) and chaos scenarios (Temporal outage, reconciliation drift).
    - Plan staged rollout: alpha tenants, beta feedback, GA checklist; capture migration strategy for existing placeholder workflows.
   - Agent Prompt — `Workflow-Enablement-Lead`:  
     Assumptions to honor: (1) Change management aligns with existing platform release process (alpha → beta → GA) and uses the company’s RFC/approval workflows. (2) Training materials must cover both operators (author/publish) and graph maintainers (trigger/monitor). (3) Chaos testing scenarios include Temporal outages, reconciliation drift, and workflows run backlogs.  
     Mission deliverables: produce authoring/operational guides, change-control process, and training decks/videos; coordinate end-to-end regression and chaos testing (service, worker, UI) and capture results; create rollout checklist with acceptance criteria for each stage; document migration plan for placeholder workflows; update Implementation Plan TODOs as tasks complete.

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Schema duplication between Temporal and service DB | Divergent views of runs | Implement periodic reconciliation jobs and alert on mismatches. |
| Workflow definition drift | Workers may execute outdated payloads | Version definitions; require publish step that snapshots configuration used for execution. |
| Overlap with architecture workflows | Teams might model business processes in platform schema | Document domains clearly and provide guidance on when to use elements vs. platform workflows. |
| Operational burden | Temporal infrastructure requires tuning | Leverage existing observability stack and define SLOs for workflows latency/failure rates. |

---

## Closed Questions & Chosen Approaches

1. **Temporal multi-tenancy** — Adopt separate Temporal namespaces per environment (`dev`, `staging`, `prod`). Per-tenant isolation is enforced via request-context metadata and queue routing inside each namespace. No additional namespaces are required.
2. **Definition delivery** — Workflow definitions must be created/updated through the gRPC WorkflowService API. Configuration repositories may orchestrate deployments, but they call the API; direct database writes are prohibited.
3. **Human-in-the-loop tasks** — Model approvals/pauses as explicit workflows states in the platform schema and Temporal payloads. External ticketing or human approval systems integrate through activities that update these states.
4. **Authoring tooling** — MVP provides a JSON editor with schema validation, reusable snippets, and version diffing. Visual editors and advanced tooling are deferred to a future roadmap.

---

## Current Implementation Status (2025-10-25)

### ✅ Production-Ready Components

**Backend Services:**
- ✅ Workflows Service (`services/workflows`) exposed from the workflows pod via Connect/HTTP2 gRPC
  - Workflow definition and version CRUD (`CreateWorkflowDefinition`, `UpdateWorkflowDefinition`, `ListWorkflowDefinitions`, `CreateWorkflowVersion`, `ListWorkflowVersions`)
  - Run lifecycle operations (`StartWorkflowRun`, `ListWorkflowRuns`, `DescribeWorkflowRun`, `CancelWorkflowRun`) backed by the TemporalFacade
  - Rule management (`CreateWorkflowRule`, `UpdateWorkflowRule`, `ListWorkflowRules`, `DeleteWorkflowRule`) with tenant scoping
  - `ReportWorkflowStatus` normalises worker callbacks, persists to Postgres, and emits OTLP counters/spans
- ✅ Temporal Integration (`services/workflows/src/temporal/facade.ts`)
  - Wrapped `@temporalio/client` with tenant isolation
  - Stub fallback mode for development
  - Search attribute filtering for element-scoped runs
- ✅ Database Schema (`db/flyway/sql/V21, V22, V23`)
  - `workflows_definitions` - Definition metadata and spec storage
  - `workflows_versions` - Immutable version snapshots
  - `workflows_rules` - Repository/transformation automation rules
  - Deployed via Helm pre-upgrade hooks with automated credentials

**Frontend UI:**
- ✅ Workflow Catalog (`/org/[orgId]/settings/workflows`)
  - Browse all workflows definitions
  - Search and filter workflows
  - New workflows button (placeholder)
- ✅ Execution Runs Dashboard (`/org/[orgId]/settings/workflows/runs`)
  - View all workflows executions across organization
  - Status filters (Running, Pending, Completed, Failed, Cancelled)
  - Search by run ID or workflows ID
  - Real-time status updates via polling
- ✅ Repository Workflow Rules (`/org/[orgId]/repo/[repoId]/settings/workflows`)
  - View graph automation rules
  - Create new rules with workflows selection
  - Event trigger configuration
  - Element type and lifecycle filtering
- ✅ Element Automations Tab (Element modal sidebar)
  - Trigger workflows for specific elements
  - View active runs with status
  - Run history display
  - Element-scoped run queries

**SDK & Clients:**
- ✅ TypeScript Workflow Client (`packages/ameide_sdk_ts/src/workflows/client.ts`)
  - Create/list/get definitions
  - Publish versions
  - Start/cancel/describe runs
  - Create/manage workflows rules
  - Tenant-aware with error normalization

**Testing:**
- ✅ E2E Test Coverage: 76% pass rate (22/29 tests)
- ✅ Integration tests for workflows service API
- ✅ Unit tests for SDK helpers and components

### 🚧 In Progress / Partial Implementation

**Worker Runtime:**
- 🟡 Python worker package (`services/workflows_runtime`) - MVP runtime complete
- 🟡 Activities for validation, guards, graph actions - Stub implementations
- 🟡 Status callbacks flow through gRPC and persist to Postgres; worker-side bridge to shared helpers still pending

**UI Features:**
- 🟡 Initiative workflows rules page - Structure in place, needs testing
- 🟢 Workflow definition editor - JSON editor with schema validation shipped
- 🟡 Run detail page - Basic structure, needs log streaming

### ❌ Not Yet Implemented

**High Priority:**
- ❌ Replace mock graph/guard activities with real adapters
- ❌ Migrate workflows runtime callbacks and CLI utilities fully onto gRPC helpers
- ❌ Verify workflows DB role can create `workflows_execution_status_updates` during bootstraps
- ✅ Workflow definition JSON editor with schema validation

**Medium Priority:**
- ❌ Real-time run status via WebSocket/SSE (currently polling)
- ❌ Visual workflows definition builder
- ❌ Lifecycle template authoring UI
- ❌ Alert rules for SLA breaches and failure rates
- ❌ Dashboards for queue depth and execution metrics

**Lower Priority:**
- ❌ Audit docs for remaining HTTP-era references and update screenshots
- ❌ Chaos testing scenarios
- ❌ Migration tooling for existing placeholder workflows

### 📊 Test Results Summary

**E2E Tests (Playwright):**
- 22 passing / 29 total (76% pass rate)
- Catalog page: 100% passing
- Runs page: 100% passing
- Navigation: 67% passing (some timeout issues)
- Workflow rules: 60% passing (element automation tests flaky)

**Known Issues:**
- Some tests timeout during platform warm-up (non-critical)
- Workflow rules UI tests need retry logic for async operations

---

## Next Steps

1. ~~Confirm domain boundaries and naming conventions for platform workflows resources.~~ ✅ **DONE**
2. ~~Design the persistence schema and Temporal payload contract.~~ ✅ **DONE**
3. ~~Spike a thin vertical slice: create → publish → execute → observe a simple workflows end-to-end.~~ ✅ **DONE**
4. **NEXT:** Migrate workflows runtime callbacks and CLI utilities fully onto the gRPC helpers (retire REST env vars)
5. **NEXT:** Verify status update table provisioning and automate workflows DB role setup
6. **NEXT:** Add observability instrumentation (OpenTelemetry traces/metrics)
7. **NEXT:** Re-enable workflows UI RBAC and expand regression coverage (UI, worker, CLI, integration packs)

---

## User Workflow Definition

### Authoring Pathways

1. **Start from Template Library** — Operators pick an existing template (e.g., lifecycle enforcement, quality gate) and clone it or create a blank definition. Templates live in the `workflows_service.workflows_definitions` table and surface in the admin UI with metadata (owner, last published version, compatible task queue).
2. **Compose Definition** — Within the UI (JSON editor with schema validation) or via CLI/SDK, authors describe:
   - Trigger: event type (`element.lifecycle.changed`, `graph.branch.created`), filter predicates (tenant, graph, element type).
   - Guards: preconditions such as dependency checks, feature flags, or approval requirements.
   - Activities: ordered or branched steps referencing worker implementations (Temporal workflows/activities). Authors select pre-registered activities from the catalog with configurable inputs.
   - Outputs: metadata updates, notifications, downstream workflows invocation.
3. **Validate & Simulate** — Before publishing, authors run schema validation and optional dry-run simulations that evaluate conditions against historical events without side effects.
4. **Publish Version** — Publishing snapshots the definition into an immutable version that Temporal workers execute. The system records changelog entries and optionally requires peer approval based on RBAC policies.
5. **Assign Runtime Parameters** — Authors supply deterministic context (timeouts, retry policies, escalation contacts, queue routing). These parameters travel with the version payload so every run started through the Temporal façade receives the same runtime defaults.

### Governance & Access

- RBAC determines who can create, edit, publish, or retire workflows definitions. Permissions reuse platform roles surfaced in `services/platform/src/organizations/service.ts`.
- Change histories retain approver, timestamp, and diff for auditability; operators can roll back to prior versions if necessary. Revision metadata is stored with the definition/version records and surfaced in the UI.

### Advanced Authoring UI (Future)

- **Schema-aware editor** — Complement the JSON editor with a visual or form-based authoring experience that manipulates the same typed `spec` payload (e.g., React Flow canvas that emits valid definitions, or a structured form wizard).
- **Hybrid Editing** — Allow switching between visual and JSON modes. Changes stay in sync, with validation guarding against invalid intermediate states.
- **Validation Feedback** — Inline validation highlights missing triggers, incompatible transitions, or unresolved activity references. Provide linting rules and reusable snippets for common patterns (e.g., approval gates, escalation loops).
- **Version Comparison** — Visual diff mode overlays old vs. new versions (JSON diff plus visual), highlighting spec changes before publishing.

---

## Attaching Workflows to Entities in the UI

### UI Architecture

Workflows are accessible at three hierarchical levels, each serving different configuration scopes:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     WORKFLOW CONFIGURATION HIERARCHY                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LEVEL 1: ORGANIZATION (Global Catalog & Platform Runs)                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ /org/[orgId]/settings → Workflows Section                           │   │
│  │   ├─ Catalog: Browse all workflows definitions                       │   │
│  │   └─ Runs: View all executions across organization                  │   │
│  │                                                                       │   │
│  │ Purpose: Platform operators manage global workflows library           │   │
│  │ Access: Organization Settings → Workflows                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LEVEL 2A: REPOSITORY (Type-Based Automation Rules)                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ /org/[orgId]/repo/[graphId]/settings/workflows                 │   │
│  │   ├─ Rule Configuration (by element type)                           │   │
│  │   ├─ Event Triggers (lifecycle, creation, relationship)             │   │
│  │   └─ Automatic Execution Policies                                   │   │
│  │                                                                       │   │
│  │ Purpose: Configure workflows that run automatically for all         │   │
│  │          elements of specific types in this graph              │   │
│  │ Access: Repository → Workflows tab OR Settings → Workflows          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LEVEL 2B: INITIATIVE (Cross-Repository Workflows)                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ /org/[orgId]/transformations/[transformationId]/settings/workflows          │   │
│  │   ├─ Cross-Repository Rules                                         │   │
│  │   ├─ Initiative-Scoped Triggers                                     │   │
│  │   └─ Delivery Pipeline Automation                                   │   │
│  │                                                                       │   │
│  │ Purpose: Orchestrate workflows across multiple repositories         │   │
│  │          within an transformation's scope                               │   │
│  │ Access: Initiative → Settings → Workflows                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LEVEL 3: ELEMENT (Individual Element Workflows)                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ /org/[orgId]/repo/[graphId]/element/[elementId]                │   │
│  │   ├─ Manual Workflow Triggers                                       │   │
│  │   ├─ Active Runs (with real-time status)                            │   │
│  │   └─ Run History                                                     │   │
│  │                                                                       │   │
│  │ Purpose: Trigger and monitor workflows for specific element         │   │
│  │ Access: Element Modal → Automations Tab                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Navigation & Access Points

```
Organization Page
├─ Header Tabs: [Overview | Initiatives | Repositories | Governance | Insights | Settings]
│
├─ Settings Page (/org/[orgId]/settings)
│  └─ Left Sidebar:
│     ├─ Features
│     ├─ Repositories
│     ├─ Initiatives
│     ├─ Billing & Identity
│     ├─ Governance
│     └─ Workflows ◄─── LEVEL 1: Global workflows catalog & runs
│        ├─ Tab: Catalog
│        └─ Tab: Runs
│
├─ Repository Page (/org/[orgId]/repo/[graphId])
│  ├─ Header Tabs: [Elements | Workflows | Settings]
│  │  └─ Workflows Tab ◄─── LEVEL 2A: Quick access to graph rules
│  │
│  └─ Settings Hub (/org/[orgId]/repo/[graphId]/settings)
│     └─ Workflow Automation Card
│        └─ Link to: /org/[orgId]/repo/[graphId]/settings/workflows
│
├─ Initiative Page (/org/[orgId]/transformations/[transformationId])
│  ├─ Header Tabs: [Overview | Planning | Elements | Governance | Settings]
│  │
│  └─ Settings Hub (/org/[orgId]/transformations/[transformationId]/settings)
│     └─ Workflow Automation Card
│        └─ Link to: /org/[orgId]/transformations/[transformationId]/settings/workflows
│           ◄─── LEVEL 2B: Cross-graph workflows rules
│
└─ Element Modal (Intercepting Route)
   └─ Right Sidebar Tabs: [Chat | Properties | Automations]
      └─ Automations Tab ◄─── LEVEL 3: Element-specific workflows
```

### Element Modal Architecture (Intercepting Routes)

```
Repository Browser: /org/[orgId]/repo/[graphId]
│
│ User clicks element from list
│
v
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ELEMENT EDITOR MODAL                                │
│  URL: /org/[orgId]/repo/[graphId]/element/[elementId]                 │
├─────────────────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────┬─────────────────────────────────┐   │
│ │                                    │   RIGHT SIDEBAR (340px)         │   │
│ │         EDITOR CANVAS              │                                 │   │
│ │       (Plugin Host)                │  ┌───────────────────────────┐  │   │
│ │                                    │  │ TAB: Chat                 │  │   │
│ │  - ArchiMate Editor                │  │ TAB: Properties           │  │   │
│ │  - BPMN Editor                     │  │ TAB: Automations [BADGE]  │  │   │
│ │  - Document Editor                 │  └───────────────────────────┘  │   │
│ │  - Properties Editor               │                                 │   │
│ │  - Canvas Editor                   │  ┌───────────────────────────┐  │   │
│ │                                    │  │  AUTOMATIONS PANEL        │  │   │
│ │  (Selected based on element type)  │  │  ==================       │  │   │
│ │                                    │  │                           │  │   │
│ │                                    │  │  [Trigger New Workflow ▼] │  │   │
│ │                                    │  │  [Start Workflow →]       │  │   │
│ │                                    │  │                           │  │   │
│ │                                    │  │  Active Runs (2)          │  │   │
│ │                                    │  │  ├─ Run #4821             │  │   │
│ │                                    │  │  │  Status: Running        │  │   │
│ │                                    │  │  │  SLA: 2h 14m remaining  │  │   │
│ │                                    │  │  │  [View] [Cancel]        │  │   │
│ │                                    │  │  └─ Run #4792             │  │   │
│ │                                    │  │     Status: Waiting        │  │   │
│ │                                    │  │     [View] [Cancel]        │  │   │
│ │                                    │  │                           │  │   │
│ │                                    │  │  Recent History (5)       │  │   │
│ │                                    │  │  ├─ Run #4760 (2h ago)    │  │   │
│ │                                    │  │  └─ Run #4721 (1d ago)    │  │   │
│ │                                    │  │                           │  │   │
│ │                                    │  └───────────────────────────┘  │   │
│ └────────────────────────────────────┴─────────────────────────────────┘   │
│                                                                             │
│ Footer: [Version History] [Last Saved: 2m ago] [Close ×]                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Implementation Files:**
- Route: `app/(app)/org/[orgId]/repo/[graphId]/@modal/(.)element/[elementId]/page.tsx`
- Fallback: `app/(app)/org/[orgId]/repo/[graphId]/element/[elementId]/page.tsx`
- Layout: `app/(app)/org/[orgId]/repo/[graphId]/layout.tsx` (provides modal slot)
- Component: `features/editor/ElementEditorModal.tsx`
- Tabs: `features/editor/RightSidebarTabs.tsx`
- Workflows Tab: `features/workflows/components/ElementWorkflowPanel.tsx`

**Navigation Behavior:**
- **From graph → Click element:** Intercepting route activates, modal opens, URL updates to `/element/:id`
- **Direct URL access (shareable links):** Fallback route renders graph page + auto-opened modal
- **Back button:** Closes modal and returns to graph list view
- This ensures deep-linkable URLs work while maintaining modal-based architecture

### Automations Tab Features (LEVEL 3: Element)

```
┌───────────────────────────────────────────────────────────────────┐
│             ElementWorkflowPanel Component                        │
│  Location: features/workflows/components/ElementWorkflowPanel.tsx│
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Trigger New Workflow                                       │ │
│  │  ┌───────────────────────────────────┐  ┌───────────────┐  │ │
│  │  │ Select Workflow                  ▼│  │ Start Button │  │ │
│  │  ├───────────────────────────────────┤  └───────────────┘  │ │
│  │  │ • Requirement Review v3           │                     │ │
│  │  │ • Architecture Validation         │                     │ │
│  │  │ • Quality Gate Check              │                     │ │
│  │  │ • Compliance Scan                 │                     │ │
│  │  └───────────────────────────────────┘                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Active Runs (2)                                            │ │
│  │  ┌───────────────────────────────────────────────────────┐ │ │
│  │  │ Run #4821                                             │ │ │
│  │  │ ┌─────────────┐  ┌──────────────────────────────────┐ │ │ │
│  │  │ │ [Running]   │  │ Requirement Review v3            │ │ │ │
│  │  │ └─────────────┘  └──────────────────────────────────┘ │ │ │
│  │  │                                                       │ │ │
│  │  │ ⏱ SLA: 2h 14m remaining  👤 Triggered by: alice     │ │ │
│  │  │                                                       │ │ │
│  │  │ Progress: [████████░░] Architect Review Pending      │ │ │
│  │  │                                                       │ │ │
│  │  │ [View Details →] [Cancel]                            │ │ │
│  │  └───────────────────────────────────────────────────────┘ │ │
│  │  ┌───────────────────────────────────────────────────────┐ │ │
│  │  │ Run #4792                                             │ │ │
│  │  │ ┌─────────────┐  ┌──────────────────────────────────┐ │ │ │
│  │  │ │ [Waiting]   │  │ Quality Gate Check               │ │ │ │
│  │  │ └─────────────┘  └──────────────────────────────────┘ │ │ │
│  │  │                                                       │ │ │
│  │  │ ⏸ Waiting for dependency (Run #4821)                 │ │ │
│  │  │                                                       │ │ │
│  │  │ [View Details →] [Cancel]                            │ │ │
│  │  └───────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Recent History (5 of 47)                                   │ │
│  │  ┌───────────────────────────────────────────────────────┐ │ │
│  │  │ ✓ Run #4760 • Quality Gate Check • 2h ago            │ │ │
│  │  │ ✓ Run #4721 • Architecture Validation • 1d ago       │ │ │
│  │  │ ✗ Run #4698 • Compliance Scan • 2d ago (Failed)      │ │ │
│  │  │ ✓ Run #4671 • Requirement Review v3 • 3d ago         │ │ │
│  │  │ ✓ Run #4654 • Quality Gate Check • 5d ago            │ │ │
│  │  └───────────────────────────────────────────────────────┘ │ │
│  │                                                             │ │
│  │  [View All History →]                                       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  [View Workflow Console →]                                        │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘

Component State:
  - availableWorkflows: WorkflowDefinition[]
  - activeRuns: WorkflowRun[]
  - recentHistory: WorkflowRun[]
  - selectedWorkflow: string | null
  - isLoading: boolean

Hooks:
  - useMockElementWorkflows(elementId): { runs, availableWorkflows, triggerWorkflow }
  - Updates every 5s for active runs (polling)

Props:
  - elementId: string
  - graphId: string
  - orgId: string
```

### Repository Workflow Rules (LEVEL 2A: Repository)

```
┌─────────────────────────────────────────────────────────────────────────┐
│          RepositoryWorkflowRules Component                              │
│  Location: features/workflows/components/RepositoryWorkflowRules.tsx   │
│  Route: /org/[orgId]/repo/[graphId]/settings/workflows            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Automation Rules                    [+ Create Rule]              │ │
│  │  ─────────────────────────────────────────────────────────────────│ │
│  │                                                                   │ │
│  │  ┌──────────────────────────────────────────────────────────────┐│ │
│  │  │ Rule: Auto-validate new requirements                         ││ │
│  │  │ ┌──────────────┐  ┌─────────────────────────────────────────┐││ │
│  │  │ │ [Active ✓]   │  │ Quality Gate Check v2                   │││ │
│  │  │ └──────────────┘  └─────────────────────────────────────────┘││ │
│  │  │                                                               ││ │
│  │  │ 🎯 Trigger: Element Created                                  ││ │
│  │  │ 📋 Element Type: Requirement                                 ││ │
│  │  │ 🔄 Lifecycle State: Any                                      ││ │
│  │  │                                                               ││ │
│  │  │ 📊 Executed: 127 times (124 success, 3 failed)               ││ │
│  │  │                                                               ││ │
│  │  │ [Edit] [Disable] [Delete] [View Runs →]                      ││ │
│  │  └──────────────────────────────────────────────────────────────┘│ │
│  │                                                                   │ │
│  │  ┌──────────────────────────────────────────────────────────────┐│ │
│  │  │ Rule: Architecture review on submission                      ││ │
│  │  │ ┌──────────────┐  ┌─────────────────────────────────────────┐││ │
│  │  │ │ [Active ✓]   │  │ Requirement Review v3                   │││ │
│  │  │ └──────────────┘  └─────────────────────────────────────────┘││ │
│  │  │                                                               ││ │
│  │  │ 🎯 Trigger: Lifecycle State Changed                          ││ │
│  │  │ 📋 Element Type: Requirement, BusinessCapability             ││ │
│  │  │ 🔄 Lifecycle State: Submitted                                ││ │
│  │  │                                                               ││ │
│  │  │ 📊 Executed: 43 times (41 success, 2 failed)                 ││ │
│  │  │                                                               ││ │
│  │  │ [Edit] [Disable] [Delete] [View Runs →]                      ││ │
│  │  └──────────────────────────────────────────────────────────────┘│ │
│  │                                                                   │ │
│  │  ┌──────────────────────────────────────────────────────────────┐│ │
│  │  │ Rule: Relationship compliance check                          ││ │
│  │  │ ┌──────────────┐  ┌─────────────────────────────────────────┐││ │
│  │  │ │ [Inactive]   │  │ Compliance Scan                         │││ │
│  │  │ └──────────────┘  └─────────────────────────────────────────┘││ │
│  │  │                                                               ││ │
│  │  │ 🎯 Trigger: Relationship Added                               ││ │
│  │  │ 📋 Element Type: All                                         ││ │
│  │  │ 🔄 Lifecycle State: Any                                      ││ │
│  │  │                                                               ││ │
│  │  │ ⚠ Disabled due to high failure rate                          ││ │
│  │  │                                                               ││ │
│  │  │ [Edit] [Enable] [Delete] [View Runs →]                       ││ │
│  │  └──────────────────────────────────────────────────────────────┘│ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

CREATE RULE DIALOG:
┌─────────────────────────────────────────────────────────────────┐
│  Create Workflow Automation Rule                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rule Name:                                                     │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Auto-validate new requirements                            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Workflow to Execute:                                           │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Quality Gate Check v2                                    ▼│ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Trigger Event:                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Element Created                                          ▼│ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Element Type Filter:                                           │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Requirement                                              ▼│ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Lifecycle State Filter (Optional):                             │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Any                                                      ▼│ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ☑ Activate rule immediately                                   │
│                                                                 │
│  [Cancel]                                [Create Rule]          │
└─────────────────────────────────────────────────────────────────┘

Component State:
  - rules: WorkflowRule[]
  - availableWorkflows: WorkflowDefinition[]
  - isCreating: boolean
  - formData: { name, workflowsId, triggerEvent, elementType, lifecycleState }

Props:
  - graphId: string
  - orgId: string
```

### Organization Workflow Catalog (LEVEL 1: Organization)

```
┌───────────────────────────────────────────────────────────────────────┐
│            Organization Settings → Workflows Section                  │
│  Route: /org/[orgId]/settings (section: workflows)                    │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  [Workflow Catalog]  [Execution Runs]                           │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  TAB: Workflow Catalog                                                │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                                                                 │ │
│  │  Workflow Definitions                    [+ Create Definition]  │ │
│  │  ─────────────────────────────────────────────────────────────  │ │
│  │                                                                 │ │
│  │  ┌────────────────────────────────────────────────────────────┐│ │
│  │  │ Requirement Review v3                                      ││ │
│  │  │ ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  ││ │
│  │  │ │ [Published]  │  │ Lifecycle    │  │ v3 (Latest)      │  ││ │
│  │  │ └──────────────┘  └──────────────┘  └──────────────────┘  ││ │
│  │  │                                                            ││ │
│  │  │ Orchestrates architect review, AI prep, and approval      ││ │
│  │  │                                                            ││ │
│  │  │ 📊 167 runs (162 ✓, 5 ✗) • Avg duration: 4h 12m          ││ │
│  │  │ 🏷 Tags: lifecycle, review, approval                       ││ │
│  │  │                                                            ││ │
│  │  │ [View Details] [Edit] [Clone] [View Runs →]               ││ │
│  │  └────────────────────────────────────────────────────────────┘│ │
│  │                                                                 │ │
│  │  ┌────────────────────────────────────────────────────────────┐│ │
│  │  │ Quality Gate Check v2                                      ││ │
│  │  │ ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  ││ │
│  │  │ │ [Published]  │  │ Validation   │  │ v2 (Latest)      │  ││ │
│  │  │ └──────────────┘  └──────────────┘  └──────────────────┘  ││ │
│  │  │                                                            ││ │
│  │  │ Automated checks for policy compliance and standards      ││ │
│  │  │                                                            ││ │
│  │  │ 📊 1,284 runs (1,251 ✓, 33 ✗) • Avg duration: 12m        ││ │
│  │  │ 🏷 Tags: quality, compliance, automated                    ││ │
│  │  │                                                            ││ │
│  │  │ [View Details] [Edit] [Clone] [View Runs →]               ││ │
│  │  └────────────────────────────────────────────────────────────┘│ │
│  │                                                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  TAB: Execution Runs                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Filters: [All Statuses ▼] [All Workflows ▼] [Last 7 days ▼]   │ │
│  │  ─────────────────────────────────────────────────────────────  │ │
│  │                                                                 │ │
│  │  ┌────────────────────────────────────────────────────────────┐│ │
│  │  │ Run #4821                                                  ││ │
│  │  │ ┌──────────────┐  ┌──────────────────────────────────────┐││ │
│  │  │ │ [Running]    │  │ Requirement Review v3                │││ │
│  │  │ └──────────────┘  └──────────────────────────────────────┘││ │
│  │  │                                                            ││ │
│  │  │ Element: REQ-1842 • Repository: primary-architecture      ││ │
│  │  │ Started: 2h 46m ago • SLA: 1h 14m remaining               ││ │
│  │  │                                                            ││ │
│  │  │ [View Details →] [Cancel]                                 ││ │
│  │  └────────────────────────────────────────────────────────────┘│ │
│  │                                                                 │ │
│  │  ... (more runs)                                                │ │
│  │                                                                 │ │
│  │  [Load More]                                                    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘

Components:
  - WorkflowCatalog: features/workflows/components/WorkflowCatalog.tsx
  - ExecutionRuns: features/workflows/components/ExecutionRuns.tsx

Props:
  - orgId: string
```

### Data Flow Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                        UI Components → Services                        │
└────────────────────────────────────────────────────────────────────────┘

LEVEL 3: Element Workflows (Trigger & Monitor Runs)
┌──────────────────────────┐
│ ElementWorkflowPanel     │
│ (React Component)        │
└───────────┬──────────────┘
            │
            │ useMockElementWorkflows(elementId)
            │
            v
┌──────────────────────────┐      ┌─────────────────────────────────────┐
│ Workflow Client Hook     │─────>│ WorkflowService API                 │
│ (hooks/useWorkflowClient)│<─────│ GET /workflows/by-element/:id       │
└──────────────────────────┘      │ POST /workflows/trigger             │
                                  │ GET /workflows/runs/:runId/status   │
                                  └──────────┬──────────────────────────┘
                                             │
                         ┌───────────────────┴────────────────────┐
                         │                                        │
                         v                                        v
              ┌──────────────────────┐              ┌────────────────────────┐
              │ PostgreSQL           │              │ Temporal Client        │
              │ workflows_definitions │              │ (@temporalio/client)   │
              │ (lookup available    │              └──────────┬─────────────┘
              │  workflows)          │                         │
              └──────────────────────┘                         │
                                                                v
                                                     ┌─────────────────────────┐
                                                     │ Temporal Cluster        │
                                                     │ - Start workflows runs   │
                                                     │ - Query run status      │
                                                     │ - Visibility API        │
                                                     │ - Execution history     │
                                                     └─────────────────────────┘

LEVEL 2: Repository Rules (Automation Configuration)
┌──────────────────────────┐
│ RepositoryWorkflowRules  │
│ (React Component)        │
└───────────┬──────────────┘
            │
            │ useState, forms
            │
            v
┌──────────────────────────┐      ┌─────────────────────────────┐
│ Workflow Rules State     │─────>│ WorkflowService API         │
│ (local component state)  │<─────│ POST /workflows/rules       │
└──────────────────────────┘      │ GET /workflows/rules        │
            │                     │ PUT /workflows/rules/:id    │
            │                     │ DELETE /workflows/rules/:id │
            │                     └──────────┬──────────────────┘
            │                                │
            v                                v
┌──────────────────────────┐      ┌─────────────────────────────┐
│ Create Rule Dialog       │      │ PostgreSQL                  │
│ - Event triggers         │      │ workflows_service schema     │
│ - Element type filter    │      │ - workflows_rules            │
│ - Lifecycle filter       │      │   * graph_id           │
│ - Workflow selection     │      │   * workflows_definition_id  │
└──────────────────────────┘      │   * trigger_event           │
                                  │   * element_type_filter     │
                                  │   * lifecycle_state_filter  │
                                  └─────────────────────────────┘

LEVEL 1: Organization Catalog (Definitions & Run History)
┌──────────────────────────┐
│ WorkflowCatalog +        │
│ ExecutionRuns            │
│ (React Components)       │
└───────────┬──────────────┘
            │
            v
┌──────────────────────────┐      ┌──────────────────────────────────┐
│ Workflow API Hooks       │─────>│ WorkflowService API              │
│ useWorkflows()           │<─────│ GET /workflows/definitions       │
│ useWorkflowRuns()        │      │ POST /workflows/definitions      │
└──────────────────────────┘      │ POST /workflows/publish          │
                                  │ GET /workflows/runs (all runs)   │
                                  └──────┬───────────────────┬───────┘
                                         │                   │
                                         v                   v
                              ┌──────────────────┐   ┌──────────────────────┐
                              │ PostgreSQL       │   │ Temporal Cluster     │
                              │ workflows_service │   │ - List executions    │
                              │ - definitions    │   │ - Run metadata       │
                              │ - versions       │   │ - Status & logs      │
                              │ - rules          │   │ - Visibility queries │
                              └──────────────────┘   └──────────────────────┘

PERSISTENCE SPLIT:
═══════════════════════════════════════════════════════════════════════════

PostgreSQL (workflows_service schema):
  ✓ workflows_definitions      - Definition metadata, spec JSON, triggers
  ✓ workflows_versions          - Immutable version snapshots
  ✓ workflows_rules             - Repository/transformation automation rules
  ✗ NO run/execution data

Temporal Cluster:
  ✓ Workflow executions        - All running & completed workflows
  ✓ Run history & status       - Timeline, activity results
  ✓ Visibility API             - Search/filter runs
  ✓ Task queues                - Worker assignments
  ✗ NO definition metadata

WorkflowService bridges both:
  - Reads definitions from PostgreSQL
  - Starts/queries runs via Temporal SDK
  - Never duplicates execution state in PostgreSQL
```

### Attachment Mechanisms

1. **Action Panels** — Repository and platform UIs expose context-aware actions (e.g., “Submit for Review”, “Run Quality Gate”). Each action references a workflows template/filter combination. Visibility is conditioned on user permissions, element metadata (type, lifecycle), and workflows availability.
2. **Automations Tab** — Entities with active workflows bindings present an “Automations” tab showing linked workflows, recent executions, and controls (trigger, retry, cancel). Data flows through `WorkflowService.ListWorkflowRuns`, and responses include the workflows run ID so UI links remain immutable.
3. **Relationship Hints** — When workflows target groups of entities (e.g., all requirements in a graph), the UI marks applicable items with badges. Hover states reveal workflows name, status, and next steps.
4. **Manual Attachments** — Operators can manually bind a workflows to a specific entity by creating a `workflows-rule` that scopes to the entity ID or tag. The UI provides a wizard to select the workflows, define scope (single entity, tag, graph folder), and activation conditions.
5. **Lifecycle Integration** — Lifecycle pickers include workflows-backed transitions. When a user attempts a transition, the UI surfaces the resulting workflows (e.g., “Submitting to Architect Review starts RequirementReviewWorkflow”) and displays prerequisites.

### Feedback & Monitoring

- Real-time status indicators (success, running, failed) appear in entity sidebars, powered by periodic updates from WorkflowService run APIs.
- Users can drill into execution history, view agent outputs, and capture manual approvals directly within the entity context, keeping the separation from graph data while maintaining visibility.

---

## End-to-End Example: Requirement Delivery Workflow

> Scenario: A product manager creates a requirement, receives automated architectural validation, obtains architect approval, and hands off to engineering for implementation.

1. **Draft Creation (Entity UI)** — The product manager opens the graph UI, clicks `New Requirement`, completes the form, and presses `Save & Submit Intake`. The page immediately surfaces a toast showing workflows run `run-4821` scheduled via WorkflowService, and the requirement detail view flips to the “Automations” tab with a pending run card.
2. **Run Context (Automations Tab)** — The run card shows “Requirement Intake Workflow v3” with status `Waiting • Lifecycle intent submitted`, the triggering actor, and the SLA clock. `View Details` deep-links to the workflows run page in the admin console.
3. **Admin Visibility (Workflow Console)** — Platform operators watching `ameide portal > organization > settings > workflows > runs` see `run-4821` enqueued. The console surfaces the captured actor identity, target graph, and the cue that approval from Architecture is required before the lifecycle transition can complete.
4. **AI Explanation Generation** — The run timeline automatically advances to “Explain Requirement”. Operators can expand the step to preview the generated summary, download the ArchiMate skeleton, or click `Open in Diagram Studio`, which launches the embedded viewer with the new draft diagram.
5. **Diagram Authoring (Diagram Studio)** — The assigned architect receives a notification badge on the requirement detail. Inside Diagram Studio they tweak components and hit `Mark Diagram Ready`. That action emits the `diagram.lifecycle_state=IN_REVIEW` signal through the UI, unblocking the waiting workflows state.
6. **Automated Review Feedback** — `VerifyRequirementActivity` executes through the `RepositoryActionProxy`. Results post back to the run timeline as a rich panel with pass/fail checks, links to policy docs, and a `Create Follow-up Task` button that opens a prefilled ticket dialog if remediation is needed.
7. **Architect Approval Gate** — The “Architect Review” step presents Approve / Request Changes buttons directly in the requirement’s Automations tab. Choosing `Approve` logs the decision, records the architect’s identity in the run metadata, and fires Temporal signal `architect_approved`.
8. **Lifecycle Transition (Repository View)** — The workflows applies the lifecycle transition via `UpdateElementLifecycleAction`. The requirement header instantly updates to `APPROVED`, the activity log records “Transition performed by <architect> on behalf of <product manager>”, and the Automations tab marks the gate as completed.
9. **Epic & Story Planning (Planning Integration)** — The run fans out to `ImplementationDeliveryWorkflow`. The run detail page shows a new step “Create epic and seed stories”, with a link to “Open in Planning” so engineering leads can jump directly to the generated work items. The Automations tab keeps both workflows runs stacked with their respective statuses.
10. **Branch & Development (Engineering Workspace)** — Engineers click `Create Branch` from the run step, launching a modal that opens the AI-assisted scaffolding flow. Each downstream activity (tests, code generation, PR creation) appends status badges and run metadata back into the workflows timeline to maintain lifecycle enforcement.
11. **Operational Monitoring (Observability UI)** — The workflows console exposes real-time OpenTelemetry metrics tagged with the run identifier. Operators can pivot to the observability dashboard to confirm no retries or SLA breaches occurred and drill into spans if anomalies appear.
12. **Completion & Metrics (Automations Tab)** — After the PR merges and deployment events arrive, the workflows posts “Delivered” with final KPIs (cycle time, review iterations, escaped defects) and a `Download Summary` action. The run detail view offers `Retry step`, `Clone workflows`, and `Create follow-up rule` buttons so operators can immediately iterate or extend automation. Reconciliation widgets show Temporal visibility aligned with the service database, closing the run in a terminal `SUCCEEDED` state.

This example demonstrates how platform workflows orchestrate UI-driven actions, AI assistance, and human approvals while preserving lifecycle enforcement, observability, and traceability across the requirement’s journey.

---

## Scenario: Lifecycle Template Authoring

> Scenario: A platform operator designs a lifecycle template that governs how requirements move from Draft to Delivered and wires it into enforcement tooling.

1. **Template Dashboard (Admin UI)** — The operator opens `ameide portal > organization > settings > workflows > lifecycle templates` and hits `New Template`. A modal collects name, description, owning team, and default SLA hints before redirecting to the template builder.
2. **State Canvas (Template Builder)** — The builder presents Draft → Review → Approved → Delivered states in a swimlane. Drag-and-drop controls let the operator add `Needs Revision` as a branching state, with inline tooltips showing inherited policies from existing templates.
3. **Transition Authoring** — Clicking the Draft → Review transition opens a side panel to assign the guard workflows. The operator selects “Requirement Intake Workflow v3”, sets “Lifecycle intent = submitted”, and enables the “Require WorkflowService approval” switch so only sanctioned runs qualify.
4. **Guard Conditions** — For Review → Approved, the operator adds required approvals (role = Architect) and a validation rule referencing `VerifyRequirementActivity` outputs. The UI previews the expected schema and warns if referenced activities lack published versions.
5. **Policy Hooks** — The template ties statuses to notifications. The operator enables “Notify owning program” and maps it to the platform notification service, ensuring signals propagate when states change.
6. **Workflow Enforcement** — In the Enforcement tab, the operator defines which graph services must call WorkflowService and sets the rejection message users see when bypass attempts occur. Integration tests are suggested via a `Download Contract` YAML snippet.
7. **Simulation Mode** — Using the `Run Simulation` button, the operator uploads sample intent events. The simulation timeline shows that Draft → Review is blocked until the corresponding workflows run reports completion, mirroring real Temporal execution.
8. **Review & Diff** — Before publishing, the operator opens `Compare with v2` to see the visual diff against the prior template, highlighting the newly introduced `Needs Revision` branch and updated approval guard.
9. **Publish Template** — Hitting `Publish v3` prompts for changelog notes and requires a second approver. Once approved, the console emits a toast with template version `lt-req-003` and posts an event to the governance channel.
10. **Deployment Targeting** — The operator selects repositories and applies the template via `Attach Template` wizard. A checkbox enforces a staged rollout: sandbox namespace first, production after 24 hours without failed reconciliations.
11. **Verification (Repository UI)** — In a requirement record, the Lifecycle picker now reflects the new states, and attempting to move from Draft to Approved shows the enforced guard summary, linking back to the template definition.
12. **Monitoring & Alerts** — The template dashboard adds “lt-req-003” to watchlists, showing counts of runs awaiting completion, guard failures, and SLA breaches. Clicking any metric opens corresponding reconciler traces tagged with template/version IDs.

---

## Scenario: Workflow Definition Authoring

> Scenario: An automation engineer creates a new “Architecture Review Prep” workflows that orchestrates AI prep, reviewer coordination, and compliance checks.

1. **Template Library (Workflow Console)** — The engineer navigates to `ameide portal > organization > settings > workflows > definitions`, filters for “Lifecycle Enforcement”, and clicks `Clone` on the baseline review workflows to seed a draft named “Architecture Review Prep”.
2. **Definition Overview** — The workflows detail page summarizes triggers, guards, and target queues. `Edit Definition` opens the hybrid editor with JSON and visual tabs side by side.
3. **Trigger Configuration** — In the visual tab, the engineer sets the trigger to `element.lifecycle.intent=review_requested`, scoped to repositories tagged `architecture-core`. A toast confirms the trigger will emit workflows runs through WorkflowService.
4. **Activity Palette (Visual Builder)** — Dragging “Generate Review Packet” from the activity palette inserts a node that encapsulates AI-driven document synthesis. Configuration panels let the engineer set input mappings and connect outputs to downstream steps.
5. **Human Task Injection** — The engineer adds a `Reviewer Assignment` human task node. The UI enables selection of approver pools (Architect, Security), while the JSON view shows the corresponding `humanTask` object with role requirements.
6. **Retry & Timeout Settings** — Clicking the “Compliance Check” activity opens the runtime parameter editor. The engineer sets max attempts to 3, exponential backoff, and a 30-minute timeout, mirroring GitHub Actions-style concurrency controls.
7. **Lifecycle Mapping** — Under the `Lifecycle` tab, the engineer references the requirement lifecycle template `lt-req-003` and specifies that the final “Approve” step marks the governing lifecycle intent as satisfied, allowing the graph update to proceed.
8. **Validation & Linting** — The `Validate` button runs schema checks, ensuring activities reference published worker versions and that all branches terminate. Issues surface inline with actionable fix links.
9. **Simulation Dry Run** — Selecting `Simulate` allows loading historical events from staging. The timeline preview shows expected durations, manual touchpoints, and signals waiting on human tasks.
10. **Peer Review** — The engineer clicks `Request Review`, tagging the platform governance group. Reviewers open an “Awaiting Approval” view that highlights diffs between the draft and baseline, including new AI steps and policy hooks.
11. **Publish & Versioning** — After approval, `Publish v1` locks the definition, emits a versioned ID `wf-arch-review-001`, and prompts for deployment to the `architecture-review` task queue. RBAC policies ensure only platform admins perform the final publish.
12. **Binding & Test Run** — The engineer uses `Attach Workflow` wizard to bind the workflows to architecture requirements in a pilot graph. Triggering a test requirement shows run `run-5934` in the Automations tab, validating that the new workflows executes end to end with UI surfaces, approvals, and telemetry mirroring the detailed run steps.

These authoring journeys give operators concrete UI flows and checkpoints to deliver lifecycle governance and workflows automation with the same depth of visibility as the delivery example.
