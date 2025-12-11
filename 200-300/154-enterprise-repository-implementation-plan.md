# Enterprise Repository – Staged Implementation Plan

This plan sequences delivery from an MVP that unlocks essential graph curation to a fully governed, collaborative platform aligned with documents 150 (functional), 151 (data), and 152 (tech stack). Each stage lists goals, scope, success criteria, dependencies, and rollout guidance.

---

## Stage 0 – Foundations & Enablement
**Goals:** Stand up core scaffolding and ensure environments can host the new services.

- **Scope**
  - Generate protobuf clients (`RepositoryAdminService`, `RepositoryTreeService`, `RepositoryAssignmentService`, `RepositoryQueryService`).
  - Create initial Alembic migrations (`010_graph`) and seed script (`011_graph_seed`).
  - Provision Postgres schemas, Redis cache, event topics, Temporal namespace (if missing).
  - Enable required Postgres extensions (`ltree`, `btree_gist`) and baseline RLS policies (feature-flagged).
  - Add SDK plumbing in `@ameideio/ameide-sdk-ts` and Next.js app skeleton for Repository Browser.
- **Success criteria**
  - Local Tilt environment runs services with seed data.
  - Frontend loads mock tree via SDK (even if backend returns stub data).
  - Observability baseline (tracing, metrics) wired for new service(s).
- **Dependencies**: None beyond existing infrastructure.

---

## Stage 1 – MVP Repository Curation
**Goals:** Allow tenants to create repositories, manage hierarchy, and assign artifacts with optimistic locking.

- **Scope**
  - Implement Repository service CRUD, tree operations, assignment APIs, and tenant isolation (FR-1 through FR-8).
  - Wire Next.js `RepositoryBrowser` for tree/list/grid, assignment lists, artifact detail drawer (read-only metadata via `artifactQueries`).
  - Backfill initial templates (standards catalog, reference architecture) and import existing seed graph.
  - MVP search (tree traversal + server-side filters) and basic stats (node counts, assignment counts).
- **Success criteria**
  - Tenant can create graph from template, add/move nodes, assign artifacts, and browse via UI.
  - P95 tree fetch < 300 ms, assignment list < 500 ms on seeded data.
  - Audit events (`graph.assignment.*`, `graph.node.*`) emitted to event bus.
  - Operational readiness: cache stampede test at 5× seed load keeps P95 tree fetch under 300 ms; idempotent retries verified via chaos tests.
- **Dependencies**: Stage 0.

---

## Stage 2 – Lifecycle & Governance Lite
**Goals:** Introduce artifact lifecycle states and governance surfaces without complex automation.

- **Scope**
  - Integrate shared command bus for lifecycle transitions (draft → in_review → approved).
  - Add `FR-11` behaviors: labels, assignees, watchers, comments, activity timeline.
  - Implement optimistic check for pinned versions (auto-pin on approve, notify on stale references).
  - Provide governance dashboards (basic counts, pinned artifacts coverage) and success metrics instrumentation.
- **Success criteria**
  - Lifecycle transitions persist state changes, notify watchers, update timeline.
  - Governance metrics (adoption, coverage) populate dashboards with daily refresh.
  - Approved artifacts auto-pin versions and surface in UI.
- **Dependencies**: Stage 1; command bus per backlog/120.

---

## Stage 3 – Governance Deep Dive (Owners, Checks, Queue)
**Goals:** Deliver NodeOWNERS, required checks, transition queue, and rulesets to achieve GitHub-style gated approvals (FR-12 to FR-16).

- **Scope**
  - Add NodeOWNERS table + APIs for owner resolution and approval tracing.
  - Implement required checks & check run orchestration via CheckRun service (OPA, CI integration).
  - Introduce Transition Queue (Temporal worker) with revalidation and queue UI.
  - Stand up check registry + validation, enforce idempotency keys and duplicate guards on queue entries.
  - Launch rulesets (tenant/template scope) with versioning and policy bundles.
- **Success criteria**
  - Transition to `approved` blocked until minimum approvals + required checks satisfied.
  - Queue ensures serialized transitions; queue UI shows position/status.
  - Ruleset updates propagate to caches; audit logs capture ruleset version per transition.
  - Operational readiness: queue SLA (P95 wait < target minutes) observed under load, SoD exception workflows audited, OPA bundle rollback tested, idempotency/duplicate guards verified with chaos tests.
- **Dependencies**: Stage 2; Temporal/OPA infra; team alignment on policies.

---

## Stage 4 – Releases, Discussions, & Advanced Collaboration
**Goals:** Improve consumption and collaboration through releases, discussions, ADRs, and enhanced UI.

- **Scope**
  - Implement `artifact_releases` + `release_assets`, protected tags, release tab UI.
  - Add discussions/ADR tables, REST/gRPC endpoints, Next.js discussion views with Markdown editors.
  - Extend timeline with check run results, releases, approvals, discussions.
  - Introduce survey service (metadata prompts) and data quality scoring.
- **Success criteria**
  - Releases published with immutable assets; protected tags enforced.
  - Discussions/ADRs active with notifications; decisions link to nodes and transformations.
  - Data quality scores appear in governance dashboards; surveys close loop on missing metadata.
  - Operational readiness: release asset hash verification on download; protected tag rules exercised in staging.
- **Dependencies**: Stage 3; S3/Blob storage for assets; Markdown editor integration.

---

## Stage 5 – Time-Aware Modeling & Graph Analytics
**Goals:** Enable as-is/to-be analysis, scenario planning, and impact analysis (FR-18, graph edges).

- **Scope**
  - Extend assignments with `effective_from/to`, scenario tags, graph projection (`model_edges`).
  - Build graph consumer to populate edges, integrate with Neo4j or Postgres table.
  - Enhance transformation workspace with As-is/To-be toggles, scenario overlays, and extended boards (table/board/roadmap) driven by graph assignment metadata.
    - Story: Deliver transformation roadmap toggle that switches As-is/To-be/Scenario views using assignment effective windows.
    - Story: Extend transformation board/table cards to surface role assignments, governance status (checks/approvals/queue), and scenario badge filters.
    - Story: Provide transformation-level impact view powered by `model_edges` that highlights affected systems with role-based filters.
  - Implement impact analysis APIs leveraging graph queries.
- **Success criteria**
  - Users switch between as-is/to-be scenarios; UI updates assignments accordingly.
  - Impact queries (e.g., “affected systems”) return results under 2 seconds on seeded dataset.
  - Initiative boards show lifecycle status, risk scores, dependency counts.
  - Operational readiness: graph projection load test validates P95 impact query < 2 s on 95th percentile tenant size; scenario cache warmers verified.
- **Dependencies**: Stage 4; graph infrastructure; design for scenario UX. Coordinate with Transformation Initiatives Stage 3 delivery (Stage 3-lite may launch with tag-based scenarios until this stage lands).

---

## Stage 6 – Integrations & Continuous Governance
**Goals:** Mature integrations, automation, and monitoring to production-grade standards.

- **Scope**
  - Add webhook/websocket subscriptions; integrate with external notification service.
  - Enforce policy-as-code bundle deployment pipeline (OPA) with testing and rollback.
  - Implement automated recertification workflows, queue SLA alerts, ruleset cache monitoring.
  - Expand API surface for external consumers (webhooks for releases, discussions, governance events).
- **Success criteria**
  - Real-time notifications for check failures, approvals, releases.
  - Policy bundles CI pipeline with automated validation; immediate cache invalidation on deploy.
  - SLOs defined and monitored (queue depth, check run duration, ruleset cache hit rate).
- **Dependencies**: Stages 1–5; coordination with platform automation teams.

---

## Stage 7 – Full Enterprise Rollout & Continuous Improvement
**Goals:** Harden for enterprise adoption, ensure compliance, and plan iterative enhancements.

- **Scope**
  - Conduct pilot with representative tenants; gather feedback on governance UX.
  - Execute load tests and failover drills (queue worker, check service).
  - Document governance playbooks, NodeOWNER onboarding guides, and support runbooks.
  - Plan backlog enhancements (Git provider integrations, AI-assisted reviews, advanced analytics).
- **Success criteria**
  - Pilot sign-off from governance/legal/security stakeholders.
  - Platform meets targeted KPIs (adoption, coverage, check latency, queue SLA).
  - Hand-off materials and training delivered; backlog prioritized for next phase.
- **Dependencies**: Completion of previous stages, cross-functional alignment (support, compliance).

---

## Rollout Guidance
- **Incremental releases:** Each stage should ship behind feature flags per tenant; maintain backward-compatible APIs and migrations.
- **Change management:** For governance-heavy stages, run workshops with architects/governance leads; pilot on limited tenants before global rollout.
- **Telemetry-first:** Instrument metrics/logs from Stage 1 onward to ensure observability underpinning later guardrails.
- **Documentation:** Update specs and runbooks after each stage; highlight changes in release notes and internal wikis.

This staged path balances quick wins (graph curation) with structural investments (governance, policy, analytics) that make the enterprise graph resilient and auditable. Subsequent roadmap items (e.g., external source-control artifact integration, AI automation) can slot after Stage 7.
