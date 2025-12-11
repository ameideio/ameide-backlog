# Transformation Initiatives Implementation Plan

## Stage 0 – Foundations
**Goals:** Establish baseline schema, service scaffolding, and SDK contracts.
- Provision database tables (`transformation_transformations`, milestones, role assignments, artifact links).
- Generate proto stubs for `InitiativeService` and `InitiativeWorkspaceService`; extend SDK.
- Configure Tilt environment with transformation service, graph mock, Temporal worker, directory stub.
- Success: CRUD prototype for transformations and milestones; basic board view renders mock data; observability baseline in place.

## Stage 1 – Initiative Lifecycle & Roles
**Goals:** Deliver core transformation management.
- Implement create/update/delete for transformations and milestones (template-driven).
- Build role assignment UX (manual selection + directory lookup placeholder) and API enforcement of uniqueness.
- Emit events on role/milestone changes; update Redis caches.
- Success: Initiatives can be created, templated milestones instantiated, roles assigned with audit trail.

## Stage 2 – Artifact Linking & Governance Consumption
**Goals:** Connect transformations to graph artifacts and governance state.
- Implement APIs/UI to link transformations to graph assignments (store `assignment_id`, milestone association, transformation metadata).
- Subscribe to graph governance events (check runs, approvals, releases) and persist status snapshots.
- Display governance banners (pending checks, approvals, queue) within transformation workspace cards.
- Success: Initiative workspace reflects governance status in real time; artifact links reference canonical assignment data.

## Stage 3 – Boards & Roadmaps (Scenario-lite)
**Goals:** Deliver transformation workspace views while Repository Stage 5 is still in progress.
- Ship board/table views grouped by lifecycle, role owner, risk flags.
- Implement roadmap timeline with As-is / To-be / Scenario toggles using tag-based scenarios (upgrade to assignment windows/`model_edges` once Repository Stage 5 lands).
- Provide role-based filters and saved views (e.g., “Needs approvals”, “Blocked checks”).
- Success: Initiative leads can manage delivery via boards/roadmaps; scenario toggles refresh tag-based overlays under 300 ms perceived latency. Full window/impact overlays ship in Stage 3+ (after Repository Stage 5).

## Stage 4 – Notifications, Surveys, KPIs
**Goals:** Automate reminders and quality tracking.
- Integrate with notification service to alert role owners (pending reviews, overdue milestones).
- Launch survey prompts for missing metadata; collect responses and update data-quality scores.
- Produce KPI dashboards (milestone completion %, approval cycle time, check success rate).
- Success: Notification/survey workflows operational; dashboards show live KPIs.

## Stage 5 – Scenario Upgrade, Directory Integration & Portfolio Views
**Goals:** Upgrade scenario capabilities, connect to corporate directory, and deliver portfolio insights.
- Integrate graph Stage 5 outputs (assignment windows, `model_edges`) into roadmap/impact views; enable full As-is/To-be/Scenario overlays with dependency highlights.
- Implement directory sync (batch or webhook) to reconcile role assignments with source of truth.
- Handle reassignment flows (e.g., departing sponsor) and surface conflicts.
- Provide tenant-level portfolio view combining multiple transformations (status heatmaps, capacity planning).
- Success: Scenario overlays reflect authoritative assignment windows/impacts; role data stays in sync with directory; portfolio dashboard supports executive reporting.
- Dependencies: Repository Stage 5 (time-aware modeling & graph analytics), identity integration readiness.

## Stage 6 – Operational Hardening
**Goals:** Ensure production readiness.
- Load test board/roadmap queries with 95th percentile tenant data; verify P95 < 500 ms.
- Chaos tests for event lag, directory outages, notification failures.
- Document runbooks, escalation paths, and retention policies for soft-deleted transformations.
- Success: SLOs agreed and met; incident response playbooks ready; compliance requirements (audit trail, legal hold) validated.

## Rollout Guidance
- Feature flag transformations per tenant; start with design partners.
- Coordinate messaging with enterprise architects/governance leads; provide training on role assignments and scenario planning.
- Gather feedback after each stage to adjust future work (e.g., additional KPIs, integration needs).
