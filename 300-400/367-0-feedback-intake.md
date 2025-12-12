> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

> **Contract alignment**
> - **Canonical contract:** `transformation-scrum-*` protos under `ameide_core_proto.transformation.scrum.v1` (Scrum nouns only) plus sibling packages for other methodologies.  
> - **Canonical integration:** intents + facts via bus; process orchestration in Temporal; no runtime RPC coupling between Process and Transformation.  
> - **Non-Scrum extensions:** explicitly listed (e.g., readiness checklists, impediments, board columns, acceptance workflows) and treated as optional policy/UX metadata, not Scrum artifacts.

# backlog/367-0 – Feedback Intake & Elements Wiring

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)

## Purpose
Stand up the capture + triage foundations so every piece of customer or internal feedback is persisted as an `element` with provenance, satisfying G1–G2 of the parent backlog and the “Everything is an Element” principle from backlog/303.

> **Runtime separation:** Elements/graph remain the authoritative knowledge graph for current/future architecture. Transformation is the system of record for Scrum and other methodology artifacts and emits domain facts. Process (Temporal) orchestrates timeboxes and governance by emitting intents and process facts and does not directly mutate the domain. Graph hosts projections, lineage, and analytics built from those facts.

**Authority & supersession**

- This backlog is **authoritative for Stage 0 intake and element wiring only**: capture channels, element creation, and methodology profile tagging.  
- Stage 1 Scrum/other methodology profiles are defined in `300-400/367-1-scrum-transformation.md` and siblings, and the **runtime event contracts** between Transformation and Process live in `506-scrum-vertical-v2.md` and the `transformation-scrum-*` protos in `508-scrum-protos.md`.  
- Earlier statements in this file about per‑methodology protos and the absence of a neutral `WorkItem` abstraction are superseded by the newer Scrum contract in 367‑1/506‑v2/508 and the per‑profile event surfaces described there.

## Scope
- Unified intake endpoints:
  - **Portal header feature** (existing “Request Feature” entry) upgrades to a guided modal with **agentic AI assistance**. The assistant prompts for: user roles, desired outcome, process context, tech preferences, acceptance criteria / “definition of success,” and optional user-story format.  
  - Public API and Slack bot continue to feed the same pipeline.
- Intake records land in `feedback_intake`; the worker converts them into Element rows with `element_kind=FEATURE_REQUEST` (not generic feedback), `type_key=feature.request`, and initial `TraceLink` relationships.
- Each element can store **team policy / working agreement metadata** (for example, team-defined readiness checklists) plus **definition of success/done** fields supplied in the modal, plus subscription metadata (requester auto-subscribed to status notifications, others can opt in). These readiness fields are optional and are not Scrum artifacts or required Scrum compliance.
- Lightweight triage UI inside Transformation/Elements workspace showing feature-request metadata, attachments, AI-suggested tags, and discussion threads.

## Methodology contracts (profile binding into per-method artifacts)
- **Profile binding at intake** – feedback is still linked to a declared methodology (e.g., Scrum/SAFe/TOGAF) via `methodology_profile_id`, but Stage 1 materializes this into the **method-specific domain contract** (for Scrum, `ameide_core_proto.transformation.scrum.v1` artifacts/messages) rather than a neutral `WorkItem`/`Timebox`/`Goal` model.  
- **Per-profile storage, shared semantics envelope** – Transformation persists runtime state in per-profile tables/protos (e.g., `transformation-scrum-*` for Scrum) and uses the methodology profile to interpret lexicon and policies. Intake does **not** decide which proto package to store into; it provides profile hints that Stage 1 uses when writing method-native artifacts.  
- **Lifecycle sources of truth** – each methodology profile still publishes its allowed transitions and guards, but enforcement happens inside the method-specific Transformation domain and the Process primitive using the event model from `506-scrum-vertical-v2.md` / `508-scrum-protos.md`.  
- **Governance envelope** – release/policy checks continue to consume `governance.v1.SubjectRef` + `governance.v1.Attestation` so promotion logic can stay shared while business semantics are profile-specific.  
- **Role maps & permissions** – profiles still ship `role_map` data (PO, Scrum Master, Architecture Board, etc.), but Stage 0 intake only captures *candidate owners*; Stage 1 applies the profile to materialize roles and permissions in the Transformation domain.

## Deliverables
1. **Schema & migrations** – Flyway scripts for `feedback_intake` + necessary enums, along with SDK/ORM updates.
2. **Ingestion services** – Connect RPCs and background worker that produces Elements + `element_relationships` entries.
3. **UI integration** – Replace the `RequestFeatureModal` stub with a guided, AI-assisted modal that captures success criteria/definitions of done, subscribes the requester for updates, and feeds the triage board view.
4. **Observability & audit** – Intake metrics, structured logs, and provenance fields (tenant id, persona, channel) stored on the element.

## Non-goals
- Any automation planning or Codex agent orchestration.
- Sprint governance or backlog prioritization—those land in Stage 1.

## Dependencies
- Existing Elements/Graph service APIs (`graph.ElementService`) and the unified metamodel decisions captured in [backlog/300-ameide-metamodel.md](./300-ameide-metamodel.md).
- Authentication + tenant context from the platform shell.

## Exit Criteria
- Submitting feedback through UI/API/Slack creates an element visible in Elements workspace within seconds.
- Architects can tag, comment, and mark team-defined readiness checklist items (working agreement metadata) directly on the feedback element.
- Telemetry dashboards show volume, channel mix, and processing latency for feedback intake.

## Implementation guide
- **Schema & migrations**
  - Add `db/flyway/sql/platform/V3__feedback_intake.sql` to create `platform.feedback_intake`, `platform.feedback_intake_subscriptions`, and `platform.feedback_events` with the workflow-friendly fields (tenant/org, channel, persona, team-defined readiness/done text, AI-enriched metadata, status timestamps, and `graph_element_id`). Reuse the Flyway pattern already in `db/flyway/sql/platform/V1__initial_schema.sql` so the migration image picked up by `infra/kubernetes/charts/platform/db-migrations` needs no special casing.
  - Seed or reference the repository that will hold `feature_request` elements. If we keep them in a dedicated graph, insert that repository/node hierarchy in `db/flyway/sql/graph/V1__initial_schema.sql` (or a follow-up migration) so the worker can safely target it.
  - Extend `packages/ameide_core_proto/src/ameide_core_proto/graph/v1/graph.proto` with a dedicated `ElementKind` for feature-intake WorkItems (e.g., `ELEMENT_KIND_WORK_ITEM_FEATURE`) and regenerate the DB adapters (`services/graph/src/graph/service.ts` uses `elementKindToDbString`). Update the UI serializers (`services/www_ameide_platform/lib/sdk/elements/mappers.ts`, `types.ts`) so the new kind stays visible without casting to `NODE`.
  - Document new secrets (Slack webhook, public API token, optional inference override) inside `scripts/vault/ensure-local-secrets.py` so local clusters and the Layer 15 bundles can hydrate the service before Flyway runs.

- **Proto & SDK contracts**
  - Introduce `packages/ameide_core_proto/src/ameide_core_proto/platform/v1/feedback.proto` with `FeedbackRequest`, `FeedbackSubmission`, `FeedbackSubscription`, and `FeedbackIntakeService` RPCs (submit, list, update state, add/remove subscribers). Run `pnpm --filter @ameide/core-proto generate` so the Buf-managed TS/Go/Python artifacts understand the service.
  - Wire the new descriptors through every client surface: `services/platform/src/proto/index.ts`, `services/transformation/src/proto/index.ts`, `services/www_ameide_platform/lib/sdk/core.ts`, and `services/www_ameide_platform/lib/sdk/ameide-client.ts`. Update `packages/ameide_sdk_ts/src/client.ts`, `packages/ameide_sdk_go/client.go`, and `packages/ameide_sdk_python/src/ameide_sdk/client.py` to expose `.feedback` clients with the same retry/auth interceptors used for the existing services.

- **Platform ingestion service & worker**
  - Add a `services/platform/src/feedback/service.ts` module that mirrors the structure of `organizations/service.ts`: validate `common.RequestContext`, persist into `platform.feedback_intake` via `withTransaction`, and emit OTEL metrics via `common/logger.ts` + `common/metrics.ts`.
  - Register the handler inside `services/platform/src/server.ts` (new `createGrpcServiceDefinition` call) so Connect clients can reach it through Envoy and the `/api/proto/[...path]` proxy.
  - The worker that turns intake rows into `graph.Element` records can either live in Temporal (reusing `services/workflows/src/temporal/facade.ts` + `services/workflows_runtime`) or as a lightweight poller inside `services/platform`. Whichever path we choose, the worker should:
    - Read pending rows from `platform.feedback_intake` (respecting tenant/org scoping) and call `client.graphService.createElement(...)` (`services/platform/src/clients/index.ts` already exposes Graph + Transformation clients).
    - Stamp `element.metadata` with team-defined readiness/done text, methodology hints, subscription IDs, and the cross-links needed for TraceLink edges. Use `graph.element_relationships` inserts (see `services/graph/src/graph/service.ts` for how relationships are persisted) to express the initial TraceLinks.
    - Publish status into `platform.feedback_events` and optional Temporal runs via the existing workflows status adapter (`services/workflows/src/status-updates.ts`), so notifications and dashboards can reuse the `workflows_execution_status_updates` feed.
  - Integration tests should follow the established pattern in `services/platform/__tests__/integration/platform-service.grpc.test.ts`: spin up the gRPC server, hit the new RPCs, assert Postgres rows, then delete the tenant.

- **Portal API & UI**
  - Replace the stub console logging in `services/www_ameide_platform/features/request-feature/components/RequestFeatureModal.tsx` with a multi-step flow that (1) captures persona/context, (2) boots an inference helper, and (3) POSTs to a new authenticated REST bridge (e.g., `services/www_ameide_platform/app/api/v1/feedback/route.ts`). Reuse `requireOrganizationMember` + `handleAuthError` from `lib/auth/middleware/index.ts` and pass a scoped `RequestContext` via `lib/sdk/request-context.ts`.
  - Build a triage board route such as `/org/[orgId]/feedback` using `ListPageLayout` + `ActivityPanel` (`services/www_ameide_platform/features/common/components/layouts/ListPageLayout.tsx`, `features/list-page/components/ActivityPanel.tsx`). Fetch data through a typed hook added to `services/www_ameide_platform/lib/api/hooks.ts`, which should call the new Next API and surface pagination/filter state similar to the repositories page.
  - Surface feature-request metadata inside the workspace UI: extend `ElementRepositoryBrowser` (`features/elements/graph/ElementRepositoryBrowser.tsx`) and the element preview cards to recognize the new `ElementKind`. If triage happens outside the main repository view, add a contextual-nav entry in `features/navigation/contextual-nav/*` and link it from the header/menu.
  - Keep `/api/elements` disabled (it intentionally returns 501 today) and go through the gRPC bridge instead; both the modal and the board should rely on the ingestion RPC you added above.

- **AI-assisted capture**
  - Add a dedicated agent/tooling profile under `services/inference/config/agents.py` + `services/inference/config/tools.py` so LangGraph/LangChain can ask clarifying questions and summarize definition-of-success text before the submission is persisted. If you prefer to drive this from the portal, reuse the streaming bridge in `services/www_ameide_platform/app/api/threads/stream/route.ts` and the `useInferenceStream` hook to pre-fill sections of the modal.
  - Document the prompts in `services/www_ameide_platform/features/ai/lib/prompts.ts`, and include repository or element context by setting the metadata envelope the inference service expects (`services/inference/context.py`).

- **Notifications & subscriptions**
  - Replace the mock data in `services/www_ameide_platform/features/navigation/components/NotificationsDropdown.tsx` with real events from the `feedback_intake` workflow (status changes, assignment, promotion). That requires a tiny REST proxy that reads `platform.feedback_events` and the subscriber list created by the new RPCs.
  - Use `services/threads/src/threads/service.ts` (SendMessage, ListThreadMessages) to power discussion threads per intake row; the worker can create the seed thread and link it via `element_relationships`.
  - Subscription endpoints should add/remove rows in `platform.feedback_intake_subscriptions` and enqueue notifications via the same pipelines the workflows service uses today (status updates recorded through `services/workflows/src/status-updates.ts`).

- **Observability & testing**
  - Leverage the existing OTEL plumbing (`services/platform/src/common/logger.ts`, `common/metrics.ts`, `services/workflows/src/status-updates.ts`, `services/www_ameide_platform/lib/threads-metrics.ts`) to emit request IDs, tenant scope, and workflow IDs end-to-end.
  - Extend the integration suites: add gRPC tests under `services/platform/__tests__/integration`, browser/API tests under `services/www_ameide_platform/__tests__/integration` (and Playwright flows in `services/www_ameide_platform/playwright`), plus SDK coverage in `packages/ameide_sdk_ts/__tests__`, `packages/ameide_sdk_go`, and `packages/ameide_sdk_python/tests`.
  - Include a Temporal/worker regression (if applicable) by adding a scenario to `services/workflows/__tests__` or the runtime fixtures in `services/workflows_runtime/__tests__`.

## Open Topics / Criticalities
- **Method-specific backlog items are undefined.** There are no `ameide_core_proto.transformation.scrum.v1`, `ameide_core_proto.transformation.safe.v1`, or `ameide_core_proto.transformation.togaf.v1` protos/tables wired into runtime yet, so the worker cannot materialize the eventual Product Backlog Item / SAFe Feature / TOGAF Deliverable rows described above. Stage 1 must land those contracts before intake can emit method-native records.
- **TraceLink edges are undefined in code.** The only references to `TraceLink` are textual (see `backlog/220-*/TraceLink` mentions); `graph.element_relationships` has no predefined `type_key` for it. We must lock down the relationship keys/directions before other services can rely on provenance queries.
- **Target repository + routing for feature requests is unclear.** `db/flyway/sql/graph/V1__initial_schema.sql` seeds only architecture/governance repositories. Decide whether feature requests belong there, in a new repository, or in tenant-scoped graphs so the worker knows where to create elements.
- **Portal notifications are placeholder-only.** `services/www_ameide_platform/features/navigation/components/NotificationsDropdown.tsx` still serves mock objects. Without a real notification feed, requesters will never see the “status change” events promised in this backlog.
- **Slack/API channel ingestion is net-new.** There is currently no Slack bot, command handler, or public REST endpoint that feeds the platform schema. Implementing those channels will require new services plus secrets in `scripts/vault/ensure-local-secrets.py`.
- **Background execution path needs a decision.** We do not yet have a queue dedicated to intake processing. Either piggyback the existing Temporal deployment (`services/workflows/src/temporal/facade.ts`, `services/workflows_runtime`) or add a poller/cronjob to `services/platform`; this choice impacts how we expose status, retries, and observability.
