# backlog/367-5 – Ameide-on-Ameide Journey Narrative

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)  
**Related:** [backlog/367-3-ameide-on-ameide.md](./367-3-ameide-on-ameide.md)

This document walks through the end-to-end experience inside the Ameide-on-Ameide tenant, showing how the per-method protobuf packages (Scrum/SAFe/TOGAF), governance envelope, automation, and release governance feel to a user. The journey highlights forks for Scrum (default squad experience), SAFe (program initiatives), and TOGAF (architecture governance) without relying on a shared business layer. For Scrum, the canonical contract is the `transformation_scrum_*` profile under `ameide_core_proto.transformation.scrum.v1` and the event seams in `506-scrum-vertical-v2.md` / `508-scrum-protos.md` (with proto naming conventions from `509-proto-naming-conventions.md`); any “DoR” or sign-off flows described here are optional extensions, not Scrum artifacts.

---

## Default Scrum Path – From Idea to Live

1. **Capture the idea (seconds).**  
   - From the portal header, click the **Request Feature** icon (or choose **➕ New → Feature Request** / post via Slack).  
   - The guided modal launches an **agentic AI assistant** that asks for goal/outcome, process context, user roles, technology preferences, story format, definition of success/done, and any attachments. The assistant pre-fills tags and DoR hints.  
   - Submission writes to `feedback_intake`; because the tenant’s default profile is Scrum, the worker later materializes a **Scrum Product Backlog Item** (`ProductBacklogItem` from `ameide_core_proto.transformation.scrum.v1`) with provenance metadata, success criteria, and automatic subscriptions for the requester, projected into the graph as an element with `element_kind=FEATURE_REQUEST`, `type_key=feature.request`.  
   - The feature request appears on the triage board for tagging/commenting, already linked to the default methodology profile.

2. **Promote to backlog (minutes).**  
   - In **Elements Workspace**, select the item → **Promote to Backlog**.  
   - During promotion, the architect links the emerging Product Backlog Item to the right context: existing **Epics/Product Goals**, parent initiatives/transformations, impacted repositories/services, and any architectural views. These become explicit `element_relationships`, so downstream agents inherit full context.  
   - The platform commits the Scrum Product Backlog Item (and matching graph element) with those relationships plus the original feedback link. Effective methodology profile resolution (`initiative.profile → product.profile → tenant.default → platform.default`) still happens, but it selects one of the concrete packages (Scrum here).  
   - The card shows up in **Transformation → Backlog**, now carrying traceability to the broader program/architecture.

3. **Sprint planning (goal-first).**  
   - Within **Planning**, choose the active **Sprint**.  
   - **PO** sets the **Sprint Goal** (Scrum commitment for the Sprint Backlog) using the Scrum lexicon; team-defined readiness items (DoR) may be checked before commitment but are treated as optional working-agreement metadata, not Scrum state.  
   - **Role map** enforces who can reorder backlog (PO), accept increments (PO), manage impediments (SM). **Daily** view highlights WIP, aging items, and open impediments.

4. **Automation plan auto-creation.**  
   - Once DoR is satisfied (context links complete, success criteria captured), the platform automatically instantiates a default Coding Agent plan using the Product Backlog Item’s repo adapter/profile defaults and resolved methodology context (`timebox_ids[]`, `goal_ids[]`, Scrum Product Backlog Item identifier).  
   - Users can still tweak plan parameters, but no manual “Create Plan” click is required.

5. **Auto-run + notifications.**  
   - The executor automatically kicks off the Coding Agent run when the plan is ready (manual reruns remain available). The orchestrator provisions workspace, applies guardrails, runs tests, and builds preview deployments.  
   - Logs stream in real time; on success the agent registers `governance.v1.Attestation`(s) (tests, docs, preview, telemetry), opens a PR, aliases the preview, and notifies all subscribers (including the original requester) with run results + preview URL. Failures raise impediments via the Scrum service and notify subscribers about the blockage.

6. **Review with evidence.**  
   - Navigate to **Automation → Runs → #<id>**. See diffs, CI results, preview link, and the **Scrum DoD checklist** fed by the newly registered Attestations.  
   - Any missing attestation surfaces as policy warnings; humans can attach additional files manually (still via `governance.v1.Attestation` helpers).

7. **Sprint Review & Increment inspection.**  
   - During Sprint Review, the **Increment** view lists all attestations per Product Backlog Item that meets the Definition of Done and is included in an Increment.  
   - The Scrum Team inspects the Increment against the Sprint Goal and Definition of Done and adapts the Product Backlog as needed. Any optional “sign-off” workflow is modeled as a governance extension (e.g., additional `governance.v1.Attestation` entries), not as a Scrum lifecycle RPC.

8. **Promotion with governance.**  
   - Open **Release → Promotions**. Each PR/run produces a **release bundle** (branch, PR metadata, Attestation references, preview URL).  
   - Pick a lane (Bronze/Silver/Gold). The policy engine evaluates blocking vs advisory rules (DoD, approvals, Architecture Board sign-off, etc.). Exceptions require rationale + expiry and show as banners on boards.  
   - Promotion re-aliases the approved preview to production and records the signed manifest (`release/agents/manifest.yaml` entry) for rollback/audit.

9. **Aftercare & learning.**  
   - Dashboards update lead time, change failure rate, automation success, exception volume (built off method events/attestations flattened by the indexer).  
   - Retrospective actions link back to backlog items; any friction automatically opens feedback items (closing the dogfooding loop).

### Click trail summary
1. `Home → New → Feedback/Feature`  
2. `Elements → <item> → Promote to Backlog`  
3. `Planning → Sprint` → set Sprint Goal / confirm DoR  
4. `Automation (auto plan/run)` → monitor run  
5. `Runs → #<id>` → review diff/tests/preview/evidence + notifications  
6. `Sprint Review → Increment`  
7. `Release → Promotions → Promote`  
8. `Dashboards` for telemetry & follow-ups

---

## SAFe Fork – Program-Level Feature

When the initiative lives in a Program Increment:

- **Link to PI/Iteration/Objectives.** During backlog promotion, associate the SAFe backlog item/Feature with a PI (parent Timebox), Iteration (child Timebox), and PI Objectives (Goal objects).  
- **PI Planning boards** show WSJF, capacity, and dependencies. Solution/Sys Demo readiness pulls from `governance.v1.Attestation`(s) (integration tests, contract testing) produced by the Solution Architect agent.  
- **Predictability & lanes.** As runs finish, they update PI Objective metrics (planned vs achieved). Release promotion verifies PI-level gates in addition to Sprint-level DoD.

---

## TOGAF Fork – Architecture Governance

For initiatives needing ADM oversight:

- **Phase tagging.** Assign the initiative (and its `togaf.v1.Deliverable` subjects) to the current ADM phase (`Timebox` subtype = phase).  
- **EA agent** generates required deliverables (Architecture Definition updates, roadmaps) and posts them as `governance.v1.Attestation` entries; Architecture Board reviews run through the profile-defined workflow.  
- **Phase gate policy.** Architecture Board acceptance (or dispensation) records as gating evidence; release promotion respects these blocking/advisory policies before production pushes.

---

## When automation struggles

- Failed runs emit `Impediment` records with diagnostic context and optional patch bundles for human takeover (`.codex/followup.md`).  
- Scheduler supports partial retries (only failed targets) without re-running successes; impediments stay linked to their subjects (Scrum backlog items, SAFe Features, TOGAF phases) plus release bundles until resolved.

---

## Metrics & transparency

- All telemetry (lead time, CFR, predictability, compliance) uses per-method events: lifecycle transitions, attestations registered, gates approved, exceptions granted.  
- Parity dashboards (internal vs. customer tenants) highlight any SLO deviations (e.g., automation coverage, exception rate). Side-channel merges are disallowed; violations trigger backlog items automatically.

---

## Implementation Guide

### 1. Intake & backlog bootstrap
- **Surface the CTA + modal.** `services/www_ameide_platform/features/header/components/HeaderActions.tsx` already renders the “Request Feature” affordance and lazy-loads the modal, so keep that entry point and extend `RequestFeatureModal` (`features/request-feature/components/RequestFeatureModal.tsx`) to capture definition-of-success/done fields that backlog/367-0 calls out.  
- **Server handler modeled after existing Graph APIs.** Follow the pattern in `services/www_ameide_platform/app/api/repositories/route.ts` (auth via `requireOrganizationMember`, Connect client hydration via `getServerClient`) to introduce `/api/feature-requests`. Use this handler to accept the modal payload, enrich it with session metadata (tenant/org IDs, requester) and enqueue persistence work.  
- **Persist as Elements immediately.** Until a dedicated `feedback_intake` table exists, write feature requests through the Graph service using the Buf SDK described in backlog/365. Reuse the `AmeideClient` plumbing in `packages/ameide_sdk_ts/src/client.ts` plus the metamodel conversion helpers in `services/graph/src/graph/elements/model.ts` to create an `element_kind=DOCUMENT`, `type_key=feature.request` row with trace metadata (per backlog/300 and backlog/303). The underlying schema in `db/flyway/sql/graph/V1__initial_schema.sql` already supports repository/tenant scoping and versioning.  
- **Seed policy-ready metadata.** Store methodology hints (desired profile, DoR evidence) as JSON in the element metadata so the coming Scrum/SAFe/Togaf backlog rows (for example, `ameide_core_proto.transformation.scrum.v1` PBIs) can inherit them once Stage 1 ships. Capture subscribers up front so notifications can flow as soon as concrete backlog IDs exist.  
- **Guard secrets + local loops.** Intake workers (whether they run inside `services/platform` or a lightweight queue consumer) must use the shared Vault bootstrap in `scripts/vault/ensure-local-secrets.py` (backlog/362) so local/staging clusters expose the same `INTEGRATION_*` secrets the Buf SDK clients expect. This keeps automated smoke tests and Codex agents from hard-failing the first time a tenant enables dogfooding intake.

### 2. Context linking & backlog promotion
- **Reuse the Elements workspace primitives.** The repository list + detail flows in `services/www_ameide_platform/app/(app)/org/[orgId]/repo` rely on `useRepositories`/`useRepositoryData` (`lib/api/hooks.ts`) and the serialization helpers in `lib/sdk/elements`. Extend those surfaces so promoted feature requests can be picked from the Elements workspace and linked to Epics/products.  
- **Persist relationships through Graph service.** When an architect selects “Promote to Backlog,” call `graphService.GraphService` (already available on the client via `useAmeideClient`) to create `element_relationships` records that bind the new backlog element to initiatives, repositories, and architectural views. The mapping helpers in `services/graph/src/graph/elements/model.ts` make it straightforward to convert Connect responses back into UI-friendly objects.  
- **Expose promotion UX in the Transformation shell.** `services/www_ameide_platform/features/transformations/components/TransformationSectionShell.tsx` demonstrates how we hydrate initiative context, milestones, and metrics via the transformation service. Add a “Promote request” action to that shell so PO/Architect roles can select a request, assign it to the active transformation, and capture DoR checks inline.  
- **Backend alignment.** The transformation service handler (`services/transformation/src/transformations/service.ts`) already owns tenants/transformations/milestones; extend it to record the backlog item ID (Scrum Backlog Item, SAFe Feature, etc.) produced by the Graph/API pipeline so upcoming methodology tables can reference the same row.

### 3. Planning & methodology controls
- **Session + role context.** backlog/329’s requirements are satisfied by the NextAuth stack in `services/www_ameide_platform/app/(auth)/auth.ts`, which refreshes Keycloak tokens, stamps `hasRealOrganization`, and harmonizes user IDs. Always call `requireOrganizationMember`/`requireOrganizationRole` (`lib/auth/middleware/authorize.ts`) before allowing edits to backlog state so Scrum/SAFe roles stay enforced.  
- **Profile-aware surfaces.** Use the `TransformationSectionShell` + `buildInitiativeBadges` utilities to render Scrum-specific lexicon (Product Goal, Sprint Goal) while storing neutral identifiers in metadata. Until the method-specific tables (`scrum_backlog_items`, `safe_features`, `togaf_deliverables`, `timeboxes`, `goals`) exist, persist the effective profile ID on transformation rows and backlog element metadata so SAFe/TOGAF overlays know which lexicon to display.  
- **Context fetchers.** The SDK already exposes `transformationService` (via `packages/ameide_sdk_ts/src/client.ts`), so planners can query PI/Iteration scaffolding from a single place. Extend the service to expose parent-child timeboxes once backlog/367-1 data structures are added.  
- **Tighten RBAC in UI.** Several planning pages still bypass policy (e.g., TODO in the workflow runs page); mirror the patterns in `app/api/repositories/route.ts` to keep backlog changes gated to PO/SM/Architect roles until policy bundles (DoR/DoD) are wired in.

### 4. Automation plan creation & agent runtime
- **Scheduler trigger path.** Define automation rules in the workflows service (`services/workflows/src/grpc-handlers.ts` + `repositories/definitions-graph.ts`). Rules already support repository/transformation scoping, so extend them to watch for “Scrum Backlog Item moved to Ready,” “SAFe Feature committed,” or “TOGAF deliverable requested” events emitted by the new methodology services.  
- **Agent catalog + LangChain runtime.** The Go agents service (`services/agents/internal/service/service.go`) broadcasts catalog snapshots and enforces tenant routing, while the Python runtime (`services/agents_runtime/src/agents_service/runtime.py`) compiles/executes LangChain v1 plans per backlog/313. Use the existing registry/routing layers to automatically provision a default Coding Agent plan whenever a backlog subject reaches DoR.  
- **Connect clients + retries.** Call automation services through the Buf SDK (backlog/365) so auth headers, retries, timeouts, and tracing are consistent. The interceptors in `packages/ameide_sdk_ts/src/interceptors.ts` already record OTEL spans and enforce metadata, so automation plan creation inherits platform guardrails for free.  
- **Temporal integration.** Workflows executions funnel through the Temporal facade in `services/workflows/src/temporal/facade.ts`, which attaches tenant/search attributes before starting runs. When auto-planning, populate the `searchAttributes` block with `work_item_id`, `goal_ids`, and `timebox_ids` so downstream promotion + dashboards can filter by governance context (per backlog/305).  
- **Status + evidence hooks.** Executors should stream step-level updates through `processWorkflowStatusUpdate` (`services/workflows/src/status-updates.ts`) so impediments/open evidence items appear in the Scrum DoD checklist automatically.

### 5. Review, evidence & release
- **Operator UI.** `services/www_ameide_platform/features/workflows/components/WorkflowCatalog.tsx` and `ExecutionRuns.tsx` already list definitions and live runs using the `useWorkflowClient` hook. Embed these components under `Automation → Runs` so reviewers can open `Runs → #<id>` exactly as the journey describes.  
- **Data access.** The React Query hooks in `services/www_ameide_platform/lib/api/hooks.ts` hit `workflowsRuntimeService` via the Buf SDK, so they already honor the retry/timeout/telemetry policies from backlog/365. Extend those hooks to fetch Attestation metadata once the backend emits it.  
- **Persistence + audit.** The workflows schema in `db/flyway/sql/workflows/V1__workflow_service_schema.sql` and the status table created in `services/workflows/src/repositories/status-updates-graph.ts` store run history separate from Temporal. Add release bundle rows (branch, preview URL, attestation URIs) there so promotion lanes can evaluate policies before mutating prod.  
- **Promotion + manifests.** Until the `release/agents/manifest.yaml` format from backlog/367-4 is added, rely on the existing `release/publish_*.sh` scripts for SDKs and capture promotion metadata in Git (e.g., under `release/`). Once the manifest lands, have the promotion UI write to it and attach signatures for rollback.

### 6. Telemetry, guardrails & parity
- **OTel everywhere.** Automation services already emit spans (`services/agents/internal/service/service.go`, `services/workflows/src/status-updates.ts`); hook new intake/promote endpoints into the same logger/tracer helpers so lead-time dashboards inherit data automatically.  
- **Consistent secrets + fixtures.** Keep local/staging clusters aligned with production guardrails by running `scripts/vault/ensure-local-secrets.py` (backlog/362) before exercising automation routes; this ensures Envoy/Temporal/SDK clients have the credentials they expect.  
- **Client-side enforcement.** The interception layer in `packages/ameide_sdk_ts/src/interceptors.ts` attaches tenant/user IDs, request IDs, retry policies, and OTEL spans to every RPC. Use that client in UI routes, queue workers, and automation orchestrators so policy enforcement stays uniform.  
- **Dashboards & parity checks.** Feed the status/attestation events captured above into the existing dashboard widgets under `services/www_ameide_platform/features/dashboard` so the Ameide-on-Ameide tenant exposes the same SLOs that customer tenants see (lead time, CFR, automation coverage), closing the dogfooding loop described in backlog/367-3.

---

## Open Topics & Criticalities

- `services/www_ameide_platform/features/request-feature/components/RequestFeatureModal.tsx:54` still contains a `TODO` stub and only simulates submission; there is no `/app/api/feature-requests` route, so feature intake never persists to Graph/feedback_intake despite the journey requiring it.  
- Neither Flyway nor the transformation service define the new `scrum_backlog_items`/`safe_features`/`timeboxes` tables yet—`services/transformation/src/db/transformation-schema.sql` only creates `transformations`, `transformation_milestones`, and `transformation_metrics`. As long as those abstractions are missing, we cannot attach DoR/DoD metadata or methodology profile IDs the journey depends on.  
- Workflow governance currently bypasses RBAC: `services/www_ameide_platform/app/(app)/org/[orgId]/settings/workflows/runs/page.tsx:11` hardcodes `canManage: true` with a `TODO`, so any authenticated user can inspect automation runs even though backlog/329 demands role-scoped access.  
- Run records never capture methodology context—`services/workflows/src/types.ts:33` defines `WorkflowRunSummary` with only IDs, status, memo, and Temporal search attributes. Without additional fields, promotion policies cannot enforce “goal-first” or PI/ADM linkage.  
- The release manifest referenced in this journey (`release/agents/manifest.yaml`) does not exist; the `release/` directory only contains SDK publish scripts. Promotion steps therefore lack a canonical artifact/sigstore trail.  
- Attestation/impediment creation is still manual: `services/workflows/src/status-updates.ts` persists status rows but does not register `governance.v1.Attestation` or method-specific impediment records, so Scrum DoD checks and SAFe/TOGAF gates cannot be evaluated automatically yet.
