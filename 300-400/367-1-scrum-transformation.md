> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/367-1-scrum-transformation – Scrum Governance Profile Enablement

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)

> **Contract alignment**
> - **Canonical contract:** `transformation-scrum-*` protos under `ameide_core_proto.transformation.scrum.v1` (Scrum nouns only: Product Backlog, Product Backlog Item, Sprint, Sprint Backlog, Increment, Product Goal, Sprint Goal, Definition of Done).  
> - **Canonical integration:** intents + facts via bus; Process (Temporal) orchestrates timeboxes and governance; no runtime RPC coupling between Process and the Scrum domain.  
> - **Non-Scrum extensions:** explicitly listed (e.g., impediments, readiness checklists, board columns, sign-off workflows) and treated as optional policy/UX metadata, not Scrum artifacts.

## Purpose
Implement the Scrum methodology profile within the Transformation service so backlog intake (Stage 0) can flow into Scrum-native artifacts: Product Goal, Sprint Goal, Product Backlog, Sprint Backlog, Increment, and optional extensions such as impediment tracking. This profile becomes the reference implementation for other governance models and must remain aligned with the canonical Scrum contract in `506-scrum-vertical-v2.md` and `508-scrum-protos.md`.

> **Related:**  
> - [506-scrum-vertical-v2.md](../506-scrum-vertical-v2.md) – canonical Scrum domain/process contract and Temporal orchestration that consume this data model.  
> - [505-agent-developer-v2.md](../505-agent-developer-v2.md) – Ameide’s multi-agent execution that relies on the Scrum events emitted here.  
> - [507-scrum-agent-map.md](../507-scrum-agent-map.md) – map of how Scrum → Process → Agent layers connect.

**Authority & supersession**

- This backlog is **authoritative for the Scrum methodology profile inside Transformation**: schema/proto changes, profile YAML, UI workflows, and how Stage 0 intake elements become Scrum artifacts.  
- It assumes the **Scrum-specific Transformation contract** under `ameide_core_proto.transformation.scrum.v1` (Product Backlog, Product Backlog Item, Sprint, Sprint Backlog, Increment, Product Goal, Sprint Goal, Definition of Done) with no neutral `WorkItem`/`Timebox`/`Goal` abstractions.  
- Earlier ideas about a separate `ameide.scrum.v1` service/namespace, a neutral Transformation runtime, and a neutral RPC surface are superseded by the Scrum‑specific protos and message contracts described in `506-scrum-vertical-v2.md` and `508-scrum-protos.md`.

## Scope
- Methodology profile definition (`profiles/scrum.yaml`) describing terminology, cadence, and optional policy hooks, all expressed in **Scrum vocabulary** (Product Backlog, Product Backlog Item, Sprint, Sprint Backlog, Increment, Product Goal, Sprint Goal, Definition of Done).
- Database/proto implementation of the `ameide_core_proto.transformation.scrum.v1` aggregates and messages defined in `508-scrum-protos.md`, plus any clearly-labeled non-Scrum extensions such as impediments.
- Noun alignment: the canonical contract nouns in Transformation and in the event model are Scrum Guide nouns (Product Backlog Item, Sprint, Increment, Product Goal, Sprint Goal, Definition of Done); any “Requirement” wording is treated as a deprecated alias only.
- Transformation UI workflows for backlog ordering, Sprint Planning, Daily Scrum views (including optional impediment tracking), Sprint Review (Increment inspection), and Retro action capture, all driven from Scrum artifacts and commitments.
- Role enforcement: explicit Product Owner (ordering and Product Backlog decisions), Developers (human/agent), Scrum Master (facilitates Scrum; may manage impediment extensions) with permissions surfaced via RBAC/config and captured in the profile `role_map`.
- Flow-metric collection (cycle time, throughput, WIP) and Monte Carlo-ready data exports per Sprint/Product Goal, derived from Scrum domain facts and process facts.
- DoR note: team “Definition of Ready” checklists may be shipped as **optional working-agreement policies**; Scrum Guide compliance centers on Definition of Done and these checklists are not Scrum artifacts.

### Graph vs. Transformation vs. Process
- **Transformation (Scrum domain)** is the system of record for Scrum artifacts and commitments: it stores Product Backlog, Product Backlog Items, Sprints, Sprint Backlogs, Increments, Product Goals, Sprint Goals, and Definitions of Done, and emits **Scrum domain intents and facts** as messages. It is reactive only (no timers/orchestration).
- **Process (Temporal)** is the timebox governor: it orchestrates Scrum event timeboxes (Sprint, Sprint Planning, Daily Scrum, Sprint Review, Sprint Retrospective) and governance by emitting **process facts** and publishing domain intents; it never directly mutates Scrum state via RPC.
- **Graph** is the architecture knowledge system: it persists elements/relationships (current + future state) and remains the source for analytics, lineage, and cross-domain reasoning. Scrum artifacts from Transformation are projected into the graph, but the graph service does not execute ceremonies itself.

## Deliverables
1. **Schema & proto updates** – Implement and wire the `ameide_core_proto.transformation.scrum.v1` protos (artifacts, intents, facts, and query service) into the Transformation service as the canonical Scrum domain contract.
2. **Profile configuration** – Versioned YAML describing ceremonies and optional policy hooks (e.g., readiness checklists) while keeping Scrum Guide compliance centered on Definition of Done; validation CLI + CI check.
3. **Service & SDK changes** – Domain intent publishing and read-only query RPCs for Scrum artifacts (no lifecycle/acceptance RPCs), client updates, and wiring into workflows/agents.
4. **UI/UX** – Scrum boards, Sprint Planning forms (goal-first), Review screen showing Increments and inspection outcomes, Retro capture.
5. **Observability** – Dashboards for Sprint Goal health, flow metrics, impediment resolution times.

## Non-goals
- Automation plan/run orchestration (Stage 2).
- Alternate methodologies (handled in sibling Stage 1 docs).

## Dependencies
- Stage 0 feedback elements with optional `methodology_profile_id` references.
- Unified element/relationship schema from [backlog/300-ameide-metamodel.md](./300-ameide-metamodel.md) and migration guidance in [backlog/303-elements.md](./303-elements.md).
- Existing Transformation service infrastructure.

## Exit Criteria
- Teams can run Scrum end-to-end inside Transformation: backlog ordering, Sprint Planning, Daily Scrum visualization, Increment inspection and Product Backlog adaptation, Retro follow-up.
- Coding/other agents appear as “Developers” within Scrum boards, with their work contributing to increments automatically.
- Flow metrics generated per Sprint and used for forecasting.

## Implementation guide

### 1. Persistence & proto schema
- **Unify table naming before layering Scrum objects.** The live read path for Transformations still runs inside the Graph service (`services/graph/src/transformation_data.ts`) and expects plain `transformations`, `transformation_workspace_nodes`, and `transformation_milestones` tables, while the schema artifacts under `db/flyway/sql/transformation/V1__initial_schema.sql` and the TypeScript service code (`services/transformation/src/transformations/service.ts#L279-L544`) still target `transformation.initiatives/*`. Close that drift first so a single Flyway path defines the canonical tables.
- **Add Scrum nouns as first-class tables.** Extend the transformation schema with Product Backlog, Product Backlog Item, Sprint, Sprint Backlog, Increment, Product Goal, Sprint Goal, and optional Impediment records that align with the `ameide_core_proto.transformation.scrum.v1` contract and the Scrum profile YAML. Keep FK/UUID links back to `graph.elements`, and store methodology metadata (team readiness checklists, cadence) in JSONB for fast policy evaluation.
- **Methodology profile resolution.** Add `methodology_profile_id` + `effective_methodology_profile_id` columns on `transformations`, `transformations_timeboxes`, and `work_items`. Persist resolution order (initiative → product → tenant → platform) so Stage 0 (backlog/367-0) and Stage 1 profiles share the same audit trail. Seed the Scrum profile rows through the migration so Ameide-on-Ameide has defaults.
- **Lifecycle DSL storage.** Reuse the shared policy bundle format (same `policy_bundle_versions` tables the workflows runtime uses) to store Scrum lifecycle DSL definitions. Reference those bundle IDs from the new Sprint/ProductGoal tables so guards can be swapped without schema churn. Persist Monte Carlo inputs (arrival rate, cycle time history) in `transformation_metric_snapshots` once new events land.

### 2. Proto & Buf contracts
- **Implement the Scrum Transformation protos.** Add the `transformation-scrum-*` files from `backlog/508-scrum-protos.md` under `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/` and register the package `ameide_core_proto.transformation.scrum.v1`. This package owns the Scrum artifacts, intents, facts, and query service.
- **Buf + SDK plumbing.** After regenerating with `buf generate`, re-export the new descriptors from `packages/ameide_core_proto/src/index.ts` and `packages/ameide_sdk_ts/src/proto.ts`, then wire convenience clients through the shared `AmeideClient` (`packages/ameide_sdk_ts/src/client.ts`, `packages/ameide_sdk_go/client.go`, `packages/ameide_sdk_python/ameide_sdk/client.py`). Update the thin-wrapper tests (e.g., `packages/ameide_sdk_ts/__tests__/integration/ameide-client.integration.test.ts`) to assert the Scrum query/intent types exist so Dependabot bumps stay honest (per backlog/365).
- **Gateways + routing.** Update Envoy/Gateway manifests so `/ameide_core_proto.transformation.scrum.v1.ScrumQueryService` resolves to the service that actually knows about the Scrum RPCs. Today the Connect routes in `gitops/ameide-gitops/sources/values/*/apps/platform/gateway.yaml` point at the Graph pod; once the dedicated `transformation` deployment is authoritative you can move the route or register both services and split traffic by method.

### 3. Services & RBAC
- **Pick a single implementation.** The Graph service (`services/graph/src/transformation_handlers.ts`) still owns some transformation RPCs, while the dedicated service under `services/transformation/src/server.ts` only handles CRUD and marks everything else as `Code.Unimplemented`. Decide whether Scrum logic lives inside the transformation service (preferred) and retire the duplication, or formally keep Graph as the façade and treat the `transformation` pod as a future worker. Either way, the Scrum query/intent adapters must reuse the shared logging/metrics helpers (`services/transformation/src/common/logger.ts`, `services/transformation/src/common/metrics.ts`) so tracing spans and OTEL metrics line up with backlog/305 workflow observability.
- **Role enforcement.** The platform RBAC helpers already exist inside the portal API (`services/www_ameide_platform/app/api/transformations/route.ts` uses `requireOrganizationRole`). Mirror those checks server-side: load the role map declared in the profile YAML, enforce PO-only backlog ordering, and restrict impediment resolution and any organizational “sign-off” extensions to Scrum Master + PO actors. Hook the checks into the handler layer before DB mutations so both the Graph-based handlers and the new service return `Code.PermissionDenied` consistently.
- **Lifecycle orchestration.** Keep the Scrum domain write surface aligned with `ScrumDomainIntent` messages and read-only query RPCs; avoid “advance lifecycle” or “acceptance” RPCs in the core Scrum contract. Use the existing transaction helper in `services/transformation/src/db.ts` (wrap everything in `withTransaction`) and emit domain facts so workflows/agents can react.

### 4. SDKs, automation, and LangChain agents
- **SDK coverage.** Extend the SDK surface so browser/server components can call the new RPCs. For TS, expose helper methods (e.g., `client.transformation.recordIncrementEvidence`) and type-safe builders for Scrum lexicon data. Do the same on Go/Python by mirroring the generated clients and wrapping metadata helpers (auth headers, retries) described in backlog/365. Update integration suites so `services/workflows` e2e tests (`services/workflows/__tests__/integration/workflows-service.e2e.test.ts`) can create test Sprints/Increments via the SDK instead of manual SQL.
- **Automation hooks.** The workflows runtime already keys rules by `transformationId` (`services/workflows/src/repositories/rules-graph.ts`). Extend the trigger payload so workflow definitions can subscribe to `timebox.events.*` and `impediment.state_changed`. Coding agent plans (LangChain migration in backlog/313 and automation backlog/367-2) should fetch the Scrum context via the SDK before calling `create_agent`. Provide helper builders so the plan template (likely under `services/threads` or `packages/ameide_core_agents`) can attach evidence back to Increment records automatically.
- **Element graph binding.** When Scrum artifacts are created/updated, write the corresponding `graph.elements` rows through the Graph service client (`packages/ameide_sdk_ts` → `graphService.GraphService`). That keeps the metamodel (backlog/303) and transformation data in sync, enabling dependency heatmaps, Monte Carlo metrics, and ADM crosswalks without duplicate modeling. Minimum projection set:
  - Sprint CRUD + lifecycle transitions.
  - Product/Sprint Goal creation and commitment, plus outcome recording.
  - Increment creation and evidence attachment.
  - Impediment creation, state changes, and resolution (as an optional extension).
  - Policy/lifecycle events that change Definition of Done status or evidence requirements on Product Backlog Items.

### 5. Web & workflow UX
- **API layer.** The portal still proxies transformation CRUD through `/api/transformations` (`services/www_ameide_platform/app/api/transformations/route.ts`) and the `useTransformations` SWR hook (`services/www_ameide_platform/lib/api/hooks.ts`). Add new API routes (e.g., `/api/transformations/[id]/sprints`) that call the new RPCs via `getServerClient`. Enforce PO-only POST/PATCH operations server-side and surface human-readable errors from Connect.
- **Screens.** Extend the org-level routes under `services/www_ameide_platform/app/(app)/org/[orgId]/transformations/*` so they render Scrum-native experiences: backlog board with ordering controls, Sprint Planning modal, Daily Scrum board, Review/Retro pages, and Increment evidence detail views. Pull lexicon strings from the profile payload rather than hard-coding “Initiative.” Reuse the existing layout primitives (`ListPageLayout`, `ActivityPanel`) but drive the cards from `timebox` + `goal` data.
- **Workflow tooling.** The portal already wraps workflow rules in `useWorkflowRules.ts` and the `workflows` client. Extend those components so operators can author Scrum automation (e.g., auto-create impediments when a workflow run fails) directly from the UI, using the `transformationId` filters now available in the workflows rule schema.

### 6. Observability, infra, and tests
- **Telemetry.** Reuse the OTEL bootstrap in `services/transformation/src/telemetry.ts` and the request-scope helpers in `services/transformation/src/common/observability.ts` to emit spans/metrics for every Scrum RPC. Attach tenant/rule metadata so flow dashboards (backlog/305) can chart throughput/WIP/impediments per Sprint.
- **Secrets & deployment.** Keep the Helm chart (`infra/kubernetes/charts/platform/transformation/*`) aligned with backlog/362 guardrails: all DB credentials come from `transformation-db-credentials`, and ConfigMaps must define the `DATABASE_*` envs that `@ameideio/configuration` requires. When new services/processors are added (e.g., Monte Carlo workers), register them in `infra/kubernetes/helmfiles/50-product-services.yaml` with the same guardrails.
- **Regression coverage.** Mirror the existing Graph → transformation integration helpers (`services/graph/__tests__/integration/helpers.ts`) for the Scrum tables, and add Jest/Vitest suites for the portal components plus Connect integration tests so `/api/transformations` flows remain green. CI jobs under `services/transformation/__tests__/integration/run_integration_tests.sh` already inject `DATABASE_SSL`; extend that harness to seed Scrum fixtures and exercise Increment/Impediment RPCs end-to-end.
