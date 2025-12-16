> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/367-2-agent-orchestration-generic – Methodology-Aware Automation Core

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)

## Purpose
Deliver the shared scheduler/executor platform that runs automation plans for any methodology profile. Provides queueing, workspace prep, policy enforcement, telemetry, and interfaces for specialized agent personas.

## Scope
- Event-driven scheduler consuming `automation.run.requested` with context: methodology profile, cadence id, backlog item, role expectations.
- Workspace provisioning (repo adapters, secrets, toolchains), guardrails (command/path allowlists), and result capture (diffs, tests, docs, preview URLs) agnostic to agent type.
- Streaming run telemetry (`automation.run.event`) plus storage for artifacts, logs, and Increment evidence descriptors.
- Policy engine integration so methodology profiles can declare required evidence (e.g., Scrum Increment vs. TOGAF deliverable) validated automatically.
- Extensibility hooks so persona-specific docs can extend behavior without duplicating the core scheduler.
- Method-aware binding – every run records the set of related Scrum/SAFe/TOGAF identifiers and emits `governance.v1.Attestation`(s) plus method-specific side effects (e.g., impediment RPCs) so downstream profiles consume the same data without a shared neutral layer.

## Deliverables
1. **Infrastructure** – Executor service (Dockerfile/Helm) with tenancy isolation, workspace cache, repo adapter runner, secret management.
2. **Scheduler logic** – Queue workers, concurrency control, retry/fallback strategies, escalation hooks for humans.
3. **Telemetry & storage** – OpenTelemetry spans, logs, artifact uploads, preview metadata, run-to-increment linkage, and persistence of governance `Attestation` objects (Milestone M1: status updates register them atomically before Stage-4 release gating) plus method impediment calls.
4. **APIs** – RPC/REST endpoints to trigger runs, stream status, fetch artifacts, register attestations via `GovernanceService`, and call method-specific impediment/advance RPCs.
5. **Documentation** – Run lifecycle, guardrails, extension points for specialized agents, plus event-bus/Temporal bridge diagrams.

## Non-goals
- Persona-specific behaviors (handled in sibling Stage 2 docs).
- Methodology profile definitions themselves.

## Dependencies
- Stage 2 plan metadata (from previous docs) and validated repo adapters/overlays.
- Agents service for node definitions (LangGraph artifacts, prompts, etc.).
- Element persistence contracts from [backlog/300-ameide-metamodel.md](./300-ameide-metamodel.md) so run artifacts map cleanly back to graph elements/relationships.

## Exit Criteria
- Any methodology profile can trigger automation plans and receive structured run outputs referencing the correct governance context.
- Run telemetry feeds back into Transformation UI and methodology-specific dashboards.
- Specialized agent docs can extend behavior via configuration/plugins rather than forking the scheduler.

## Implementation guide

### Event ingress & scheduling
- Extend the existing WorkflowService skeleton (`services/workflows/src/grpc.ts` and `grpc-handlers.ts`) so the scheduler can consume `automation.run.requested` notifications in addition to direct RPC calls. Workflow rules already live in Postgres (`db/flyway/sql/workflows/V2__workflow_rules_schema.sql` and `services/workflows/src/repositories/rules-graph.ts`); reuse that schema to persist methodology profile filters (PI/Scrum/TOGAF) and to map incoming events to the correct Temporal task queue, queue depth, and escalation policies described in backlog/305.
- Build a NATS/Kafka consumer (depending on environment) that translates `automation.run.requested` events into `TemporalFacade.startWorkflow` invocations, copying tenant/profile identifiers into Temporal Search Attributes and run metadata. This closes the “auto-plan/auto-run” loop promised in Stage-2/Stage-5.
- Encapsulate event-to-run translation in a dedicated dispatcher that enriches every Temporal start with tenancy data, backlog identifiers, and methodology metadata before calling `TemporalFacade.startWorkflow` (`services/workflows/src/temporal/facade.ts`). Leverage `runWithRequestContext` to propagate `x-tenant-id`/`x-scope-orgid` headers in line with backlog/329 (no default tenants).

### Execution & Temporal control loop
- The Temporal worker in `services/workflows_runtime/src/workflows_runtime/workflows.py` already sequences validation, guard evaluation, action execution, and status reporting. Expand the payload (`models.WorkflowRunPayload`) to include methodology profile ids, cadence ids, and repo adapter settings so every `WorkflowStatusUpdate` persisted via `services/workflows/src/status-updates.ts` carries the governance context required by this backlog.
- Connect `automation.run.event` streaming to WorkflowService’s `ReportWorkflowStatus` RPC by wiring the worker callbacks (`services/workflows_runtime/src/workflows_runtime/callbacks.py`) through the same OTEL-aware request context used by the API tier. This keeps Transformation dashboards and CLI consumers synchronized while allowing retries and human escalation (pause/resume) through the Temporal APIs that `services/workflows/src/grpc-handlers.ts` already expose (`DescribeWorkflowRun`, `CancelWorkflowRun`).

### Contracts & SDK alignment
- Buf-managed protos (`packages/ameide_core_proto/src/ameide_core_proto/workflows_runtime/v1/workflows_service.proto` and `workflows_types.proto`) already cover definitions, runs, rules, and status updates. Add the automation-plan specific fields (`AutomationPlan`, workspace/evidence references) there first, regenerate the SDKs tracked in backlog/365 (`packages/ameide_sdk_ts/src/client.ts`, `packages/ameide_sdk_go/client.go`, `packages/ameide_sdk_python/src/ameide_sdk/client.py`), and then update UI + CLI entry points (`services/www_ameide_platform/lib/sdk/workflows/client.ts`, `packages/ameide_core_cli/internal/commands/workflow.go`) so methodology-aware payloads flow end to end.
- Keep the Connect server in `services/workflows/src/grpc.ts` the single ingress—Transformation, Automation UI, and Codex tooling should all call the same service descriptors (`workflowsRuntimeService`). Any REST/Webhook shims should be thin adapters that forward directly to this surface to avoid a second API.

### Workspace provisioning & repo adapters
- Reuse the agents control-plane/runtime stack instead of building bespoke workspace runners. The Agents service compiles persona artifacts to MinIO (`services/agents/internal/compiler/*.go`, `services/agents/internal/artifacts/storage.go`) and the LangGraph-inspired runtime (`services/agents_runtime/src/agents_service/runtime.py` + `registry.py`) already downloads, caches, and executes those bundles. Extend that runtime so `ActionBundle` executions from the workflows worker call into persona-specific repos (coding agent, enterprise architect) rather than the current stub logic in `services/workflows_runtime/src/workflows_runtime/activities.py`.
- Guard repo access with the secret story defined in backlog/362: refresh workspace credentials via the Existing ExternalSecrets the charts already provide (`agents-runtime-db-secret` in `vault-secrets-platform`, `scripts/vault/ensure-local-secrets.py`) and surface explicit allow/deny lists per adapter so GuardContext validation can fail fast before a Temporal run starts.

### Attestations & methodology hand-offs
- Persist automation outputs back into the element graph using the existing APIs under `services/graph/src/graph/*.ts` and the schema from backlog/303/300. Every run should emit `Element` updates (diffs, documents, diagrams) plus register `governance.v1.Attestation` records and call the appropriate methodology service (Scrum/TOGAF/SAFe) for impediments or lifecycle moves. Wire helper clients inside WorkflowService so that when `WorkflowStatusUpdate` transitions to COMPLETED it atomically calls the graph service, `GovernanceService.RegisterAttestation`, and the target methodology RPC to attach proof to increments, PI objectives, or ADM deliverables.
- Normalize reference IDs—`timebox_ids`, `goal_ids`, `work_item_ids`, `element_ids`—inside WorkflowService so downstream profiles can query by governance construct without scraping logs.

### Telemetry, artifacts & run records
- Keep end-to-end observability consistent with backlog/305: the API tier already emits OTEL traces/metrics through `services/workflows/src/telemetry.ts` and `common/request-context.ts`, and the worker has structured logging/exporters in `services/workflows_runtime/src/workflows_runtime/telemetry.py`. Propagate automation plan attributes (profile, cadence, persona) into both spans and metrics so platform dashboards can slice by methodology.
- Artifact storage exists today via MinIO/S3 (`services/agents/internal/artifacts/storage.go`, `services/agents_runtime/scripts/bootstrap_artifacts.py`). Reference those helpers instead of inventing a new uploader so diff bundles, test logs, and preview manifests land in the same buckets with consistent retention policies. Link stored URIs back to WorkflowService memo/search attributes for quick retrieval.

### Guardrails & operations
- Infrastructure pieces already live in Helm/Tilt: `infra/kubernetes/charts/platform/workflows` + `workflows_runtime` provide Dockerfiles, health probes, and ExternalSecret wiring; `gitops/ameide-gitops/scripts/validate-hardened-charts.sh` enforces the guardrails described in backlog/362. Extend those charts with automation-specific env vars (repo adapters, cache paths, guardrail allowlists) and follow the same pattern (configmap + secret + deployment) so platform SRE can operate the scheduler like every other service.
- Keep CI/local parity by updating `Tiltfile` resources for the scheduler, worker, agents, and MinIO so engineers can reproduce automation runs end to end before promoting configuration.

### Extensibility & persona hooks
- Personas already plug into the runtime through the Agents catalog (`services/agents/internal/catalog`, `services/agents_runtime/src/agents_service/routing.py`). Instead of bespoke plugins, register automation-specific nodes (Coding Agent, Solution Architect, Enterprise Architect) in that catalog and allow methodology profiles to select them via configuration. This keeps LangChain v1 adoption (backlog/313) in one place—the `LangGraphRuntime`—and lets automation plans point to compiled artifacts without duplicating routing logic.
- Document the configuration surface (methodology profile → automation plan → agent persona) alongside the existing `@ameideio/configuration` conventions so specialized docs (coding agent, EA agent) extend the shared scheduler without forking it.

### Outstanding work / risks
- **No automation event bus yet** – Need a message-bus subscriber translating `automation.run.requested` into Temporal runs; tracked as part of the event ingress milestone.
- **Workspace execution stubbed** – `execute_graph_actions` still simulates actions; replace with real repo/test/preview adapters wired to Codex CLI.
- **Evidence/Impediment persistence** – Kernel schema + APIs must land before Stage-4 gating; blocker milestone.
- **Methodology context fields** – Add `methodology_profile_id`, cadence, increment bindings to Workflow definitions/schema.
- **Tenant hygiene** – Eliminate `DEFAULT_TENANT_ID` fallbacks; enforce explicit tenant context.
- **LangChain v1 migration** – Runtime adoption of `create_agent` required; see coding-agent doc for exit criteria.
