> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/367-1-scrum-transformation – Scrum Governance Profile Enablement

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)

## Purpose
Implement the Scrum methodology profile within the Transformation service so backlog intake (Stage 0) can flow into Scrum-native artifacts: Product Goal, Sprint Goal, Product Backlog, Sprint Backlog, Increment, and Impediment tracking. This profile becomes the reference implementation for other governance models.

## Scope
- Methodology profile definition (`profiles/scrum.yaml`) describing terminology, cadence, allowed state transitions, and policy hooks.
- Database/proto extensions for `product_goals`, `sprint_goals`, `increments`, and `impediments`, each scoped by `methodology_profile_id`.
- Transformation UI workflows for backlog ordering, Sprint Planning, Daily Scrum board (aging WIP, impediments), Sprint Review (Increment acceptance), and Retro action capture.
- Role enforcement: explicit Product Owner (ordering + acceptance), Developers (human/agent), Scrum Master (impediment management) with permissions surfaced via RBAC/config and captured in the profile `role_map`.
- Flow-metric collection (cycle time, throughput, WIP) and Monte Carlo-ready data exports per Sprint/Product Goal.
- DoR note: Definition of Ready checklists are shipped as profile policies (optional); Scrum Guide compliance centers on Definition of Done.
- Protos & lifecycle – `ameide.scrum.v1` defines BacklogItem/Increment/Impediment and encodes the Scrum state machine (IDEA → READY → IN SPRINT → IN PROGRESS → READY FOR REVIEW → REVIEWED → DONE) directly so guards/policies update via configuration.
- Service surface – `ScrumLifecycleService` exposes `AdvanceItem`, `DescribeItem`, acceptance, and impediment RPCs; guards such as DoD enforcement live in the service rather than a neutral shared layer.
- Dependency + nesting support – allow Sprint (Timebox) to point to parent PIs or program-level Timeboxes and surface `Dependency` records on Scrum boards even if the SAFe profile is not active.

### Graph vs. transformation runtime
- **Transformation** is the platform governance runtime: it stores backlog metadata, runs Scrum/SAFe/TOGAF lifecycles, enforces role maps, and emits policy decisions in real time.
- **Graph** is the architecture knowledge system: it persists all elements/relationships (current + future state) and remains the source for analytics, lineage, and cross-domain reasoning.
- Every methodology artifact created or advanced inside Transformation must project back into the graph (element/relationship rows) so knowledge graph views stay complete, but the graph service does not execute ceremonies itself.

## Deliverables
1. **Schema & proto updates** – Scrum-specific tables/messages (BacklogItem, Sprint, ProductGoal, Increment, Impediment) that stand on their own, without shared neutral abstractions, and point to graph elements for lineage.
2. **Profile configuration** – Versioned YAML describing DoR/DoD templates, ceremonies, and governance rules; validation CLI + CI check.
3. **Service & SDK changes** – Neutral RPCs (`SetTimeboxGoal`, `RecordIncrementEvidence`, `AcceptIncrement`, `CreateImpediment`, `ResolveImpediment`) that map to Scrum semantics via lexicon but remain profile-agnostic.
4. **UI/UX** – Scrum boards, Sprint Planning forms (goal-first), Review screen showing increments + acceptance, Retro capture.
5. **Observability** – Dashboards for Sprint Goal health, flow metrics, impediment resolution times.

## Non-goals
- Automation plan/run orchestration (Stage 2).
- Alternate methodologies (handled in sibling Stage 1 docs).

## Dependencies
- Stage 0 feedback elements with optional `methodology_profile_id` references.
- Unified element/relationship schema from [backlog/300-ameide-metamodel.md](./300-ameide-metamodel.md) and migration guidance in [backlog/303-elements.md](./303-elements.md).
- Existing Transformation service infrastructure.

## Exit Criteria
- Teams can run Scrum end-to-end inside Transformation: backlog ordering, Sprint Planning, Daily Scrum visualization, Increment acceptance, Retro follow-up.
- Coding/other agents appear as “Developers” within Scrum boards, with their work contributing to increments automatically.
- Flow metrics generated per Sprint and used for forecasting.

## Implementation guide

### 1. Persistence & proto schema
- **Unify table naming before layering Scrum objects.** The live read path for Transformations still runs inside the Graph service (`services/graph/src/transformation_data.ts`) and expects plain `transformations`, `transformation_workspace_nodes`, and `transformation_milestones` tables, while the schema artifacts under `db/flyway/sql/transformation/V1__initial_schema.sql` and the TypeScript service code (`services/transformation/src/transformations/service.ts#L279-L544`) still target `transformation.initiatives/*`. Close that drift first so a single Flyway path defines the canonical tables.
- **Add Scrum nouns as first-class tables.** Extend the transformation schema with Sprint-aware timeboxes, ProductGoal/SprintGoal join tables, Increment metadata, and Impediment records that align with the `ameide.scrum.v1` protos. Keep FK/UUID links back to `graph.elements`, and store methodology metadata (DoR/DoD templates, cadence) in JSONB for fast policy evaluation.
- **Methodology profile resolution.** Add `methodology_profile_id` + `effective_methodology_profile_id` columns on `transformations`, `transformations_timeboxes`, and `work_items`. Persist resolution order (initiative → product → tenant → platform) so Stage 0 (backlog/367-0) and Stage 1 profiles share the same audit trail. Seed the Scrum profile rows through the migration so Ameide-on-Ameide has defaults.
- **Lifecycle DSL storage.** Reuse the shared policy bundle format (same `policy_bundle_versions` tables the workflows runtime uses) to store Scrum lifecycle DSL definitions. Reference those bundle IDs from the new Sprint/ProductGoal tables so guards can be swapped without schema churn. Persist Monte Carlo inputs (arrival rate, cycle time history) in `transformation_metric_snapshots` once new events land.

### 2. Proto & Buf contracts
- **Extend `Transformation` protos, don’t fork new services.** Add the Sprint/ProductGoal/SprintGoal/Increment/Impediment messages and RPCs to `packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation.proto` + `transformation_service.proto`. Keep the wire model neutral (`WorkItem`, `Timebox`, `Goal`, etc.) and add Scrum lexicon hints (`display_label`, `role_map`) via the common annotations package so other profiles can reuse the same RPCs.
- **Buf + SDK plumbing.** After regenerating with `buf generate`, re-export the new descriptors from `packages/ameide_core_proto/src/index.ts` and `packages/ameide_sdk_ts/src/proto.ts`, then wire convenience clients through the shared `AmeideClient` (`packages/ameide_sdk_ts/src/client.ts`, `packages/ameide_sdk_go/client.go`, `packages/ameide_sdk_python/ameide_sdk/client.py`). Update the thin-wrapper tests (e.g., `packages/ameide_sdk_ts/__tests__/integration/ameide-client.integration.test.ts`) to assert the Scrum RPCs exist so Dependabot bumps stay honest (per backlog/365).
- **Gateways + routing.** Update Envoy/Gateway manifests so `/ameide_core_proto.transformation.v1` resolves to the service that actually knows about the Scrum RPCs. Today the Connect routes in `gitops/ameide-gitops/sources/values/*/apps/platform/gateway.yaml` point at the Graph pod; once the dedicated `transformation` deployment is authoritative you can move the route or register both services and split traffic by method.

### 3. Services & RBAC
- **Pick a single implementation.** The Graph service (`services/graph/src/transformation_handlers.ts`) still owns all transformation RPCs, while the dedicated service under `services/transformation/src/server.ts` only handles CRUD and marks everything else as `Code.Unimplemented`. Decide whether Scrum logic lives inside the transformation service (preferred) and retire the duplication, or formally keep Graph as the façade and treat the `transformation` pod as a future worker. Either way, the Scrum RPCs must reuse the shared logging/metrics helpers (`services/transformation/src/common/logger.ts`, `services/transformation/src/common/metrics.ts`) so tracing spans and OTEL metrics line up with backlog/305 workflow observability.
- **Role enforcement.** The platform RBAC helpers already exist inside the portal API (`services/www_ameide_platform/app/api/transformations/route.ts` uses `requireOrganizationRole`). Mirror those checks server-side: load the role map declared in the profile YAML, enforce PO-only backlog ordering, and restrict impediment resolution/Sprint acceptance to Scrum Master + PO actors. Hook the checks into the handler layer before DB mutations so both the Graph-based handlers and the new service return `Code.PermissionDenied` consistently.
- **Lifecycle orchestration.** Surface neutral RPCs (`SetTimeboxGoal`, `AdvanceLifecycleState`, `RecordIncrementEvidence`, `Create/ResolveImpediment`) from the service layer, backed by SQL mutations + element graph writes. Use the existing transaction helper in `services/transformation/src/db.ts` (wrap everything in `withTransaction`) and emit events so workflows/agents can react.

### 4. SDKs, automation, and LangChain agents
- **SDK coverage.** Extend the SDK surface so browser/server components can call the new RPCs. For TS, expose helper methods (e.g., `client.transformation.recordIncrementEvidence`) and type-safe builders for Scrum lexicon data. Do the same on Go/Python by mirroring the generated clients and wrapping metadata helpers (auth headers, retries) described in backlog/365. Update integration suites so `services/workflows` e2e tests (`services/workflows/__tests__/integration/workflows-service.e2e.test.ts`) can create test Sprints/Increments via the SDK instead of manual SQL.
- **Automation hooks.** The workflows runtime already keys rules by `transformationId` (`services/workflows/src/repositories/rules-graph.ts`). Extend the trigger payload so workflow definitions can subscribe to `timebox.events.*` and `impediment.state_changed`. Coding agent plans (LangChain migration in backlog/313 and automation backlog/367-2) should fetch the Scrum context via the SDK before calling `create_agent`. Provide helper builders so the plan template (likely under `services/threads` or `packages/ameide_core_agents`) can attach evidence back to Increment records automatically.
- **Element graph binding.** When Scrum artifacts are created/updated, write the corresponding `graph.elements` rows through the Graph service client (`packages/ameide_sdk_ts` → `graphService.GraphService`). That keeps the metamodel (backlog/303) and transformation data in sync, enabling dependency heatmaps, Monte Carlo metrics, and ADM crosswalks without duplicate modeling. Minimum projection set:
  - Sprint/Timebox CRUD + lifecycle transitions.
  - Product/Sprint Goal creation, commitment, acceptance.
  - Increment creation, evidence attachment, acceptance.
  - Impediment creation, state changes, and resolution.
  - Policy/lifecycle events that change DoR/DoD status on WorkItems.

### 5. Web & workflow UX
- **API layer.** The portal still proxies transformation CRUD through `/api/transformations` (`services/www_ameide_platform/app/api/transformations/route.ts`) and the `useTransformations` SWR hook (`services/www_ameide_platform/lib/api/hooks.ts`). Add new API routes (e.g., `/api/transformations/[id]/sprints`) that call the new RPCs via `getServerClient`. Enforce PO-only POST/PATCH operations server-side and surface human-readable errors from Connect.
- **Screens.** Extend the org-level routes under `services/www_ameide_platform/app/(app)/org/[orgId]/transformations/*` so they render Scrum-native experiences: backlog board with ordering controls, Sprint Planning modal, Daily Scrum board, Review/Retro pages, and Increment evidence detail views. Pull lexicon strings from the profile payload rather than hard-coding “Initiative.” Reuse the existing layout primitives (`ListPageLayout`, `ActivityPanel`) but drive the cards from `timebox` + `goal` data.
- **Workflow tooling.** The portal already wraps workflow rules in `useWorkflowRules.ts` and the `workflows` client. Extend those components so operators can author Scrum automation (e.g., auto-create impediments when a workflow run fails) directly from the UI, using the `transformationId` filters now available in the workflows rule schema.

### 6. Observability, infra, and tests
- **Telemetry.** Reuse the OTEL bootstrap in `services/transformation/src/telemetry.ts` and the request-scope helpers in `services/transformation/src/common/observability.ts` to emit spans/metrics for every Scrum RPC. Attach tenant/rule metadata so flow dashboards (backlog/305) can chart throughput/WIP/impediments per Sprint.
- **Secrets & deployment.** Keep the Helm chart (`infra/kubernetes/charts/platform/transformation/*`) aligned with backlog/362 guardrails: all DB credentials come from `transformation-db-credentials`, and ConfigMaps must define the `DATABASE_*` envs that `@ameideio/configuration` requires. When new services/processors are added (e.g., Monte Carlo workers), register them in `infra/kubernetes/helmfiles/50-product-services.yaml` with the same guardrails.
- **Regression coverage.** Mirror the existing Graph → transformation integration helpers (`services/graph/__tests__/integration/helpers.ts`) for the Scrum tables, and add Jest/Vitest suites for the portal components plus Connect integration tests so `/api/transformations` flows remain green. CI jobs under `services/transformation/__tests__/integration/run_integration_tests.sh` already inject `DATABASE_SSL`; extend that harness to seed Scrum fixtures and exercise Increment/Impediment RPCs end-to-end.
