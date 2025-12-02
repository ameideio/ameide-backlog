# Executive Summary
Transformation Initiatives represent tenant-led change programs (product naming for legacy “ADM cycles”) that orchestrate repositories and artifacts in Ameide. Initiatives coordinate proposed/WIP architectures, delivery progress, and roles, while governance (owners, approvals, required checks, rulesets, promotion queue) remains defined once at graph/tenant scope. This document outlines the functional scope required to plan, execute, and track transformations while reusing the graph infrastructure defined in `backlog/150-enterprise-graph-functional.md`.

# Objectives
- Provide a tenant-scoped portfolio view of transformations, their phases, and status.
- Link transformations bidirectionally with graph artifacts, standards, and assignments (including effective windows and scenario tags for proposed/WIP designs).
- Surface progress dashboards and transformation-specific views (boards/roadmaps) that reflect graph data while honoring graph-level governance outcomes (checks, approvals, queue status).
- Enable collaboration across Enterprise Architects, Initiative Sponsors, Initiative Leads, Delivery Teams, and Reviewers with artifact-centric discussions and review workflows.

# Key Capabilities (Draft)
- Initiative lifecycle management (create, plan, launch, close) with template-driven phase definitions.
- Association of repositories, nodes, and artifacts to transformation milestones (including effective windows and scenario tags for proposed/WIP architectures).
- Initiative workspace views (table/board/roadmap) with As-is / To-be / Scenario toggles derived from graph assignments.
- Status tracking and KPI dashboards leveraging graph stats, assignment metadata, and graph-level governance outputs (checks, approvals, queue status).
- Integration hooks for notifications, workflows automation, and change logs.
- People & roles (and agents) assignment (future integration with company directory and agent registry) to capture transformation sponsors, program managers, architects, delivery leads, reviewers, contributors, and AI reviewers/assistants, including RBAC alignment and availability.
- Release readiness and blocker reporting for Product Owners, aggregating artifact approvals, required checks, queue status, and upcoming releases.

# Personas & Roles (Draft)
- **Initiative Sponsor**: Executive accountable for outcomes; sets KPIs, approves stage gates.
- **Program Manager**: Coordinates execution, updates timelines, manages risks and communications.
- **Product Owner**: Curates transformation backlog, defines acceptance criteria, runs release readiness checks, and aligns business stakeholders; partners with Lead Architect/Governance Lead to ensure quality gates are met.
- **Lead Architect**: Aligns transformation scope with graph architecture; curates artifacts and ensures governance adherence.
- **Delivery Lead**: Oversees implementation teams, links workstreams to transformation milestones.
- **Contributors**: Cross-functional specialists (engineers, analysts) assigned to specific milestones or artifacts.
- **AI Reviewer/Assistant**: Automated agent that performs preliminary checks, generates summaries, or proposes recommendations prior to human approval; participates in review workflows with clearly scoped permissions.

# Role Management & Directory Integration (Draft)
- Support assignment of people and AI agents to transformations with role tags (sponsor, program manager, architect, delivery lead, contributor, ai_reviewer).
- Store source-of-truth references (e.g., company directory GUID, agent registry ID, email/webhook) for each principal; allow sync from corporate directory/agent registry (future integration).
- Provide RBAC mapping so transformation roles inherit appropriate graph permissions (e.g., lead architect retains governance approval rights) and constrain AI agents to approved scopes (read-only, check execution, advisory comments).
- Surface ownership and accountability in transformation dashboards (e.g., cards display assigned leads or agent assignments, empty slots flagged for action).
- Expose APIs to list/search transformation members and agents, update roles, and emit events when assignments change (for downstream notifications or agent orchestration).

# Artifact-Centric Collaboration & Governance Alignment
- Discussions, ADRs, and request-for-review workflows remain anchored to artifacts in the graph; transformation workspaces deep-link to those timelines rather than duplicating threads.
- Initiatives surface consolidated status for artifacts (e.g., checks pending, approvals required, queue position, AI reviewer findings) by consuming graph governance outputs.
- Promotion decisions and guardrails stay governed at graph/tenant scope; transformations reflect outcomes for planning and reporting without redefining approval logic.

# UI Architecture
## Overview
Transformation transformation screens live inside the existing Next.js 15 SPA, sharing the AppShell, design system, and provider stack used by repositories. The workspace layers planning views (boards, tables, roadmaps, KPIs) atop graph assignments and governance outputs without duplicating governance logic.

## Existing Platform Context
- **App shell**: `app/(app)/layout.tsx` provides Header, Sidebar, Search, Tooltip, DataStream, UrlSyncGate, and Debug providers. Initiative routes integrate with this shell rather than redefining providers.
- **Header**: Contextual breadcrumbs/tabs rendered via `resolveHeaderDescriptor` expose Overview, Board, Roadmap, KPIs, and Settings for each transformation.
- **Sidebar**: `AppSidebar` gains transformation navigation entries and “New Initiative” quick action; history panel captures recently viewed transformations.
- **Providers & telemetry**: Hooks (`useSearch`, `useSidebar`, `useDebug`) and observability utilities are reused. Scenario/Realtime contexts compose inside `DataStreamProvider` for consistent lifecycle handling.
- **Routing**: Initiative workspace lives under `/transformations/:transformationId`, coexisting with graph paths. Feature flags (e.g., `transformation_boards`, `transformation_roadmap`) guard rollout per tenant.

## Component Topology
```
====================================================================================================================
| AppShell (app/(app)/layout.tsx)                                                                                  |
| ---------------------------------------------------------------------------------------------------------------- |
|  Providers: DebugProvider → TooltipProvider → SearchProvider → DataStreamProvider → UrlSyncGate → SDKClient      |
|  Shared UI: Header (breadcrumbs, scope tabs, actions) • AppSidebar (nav, history, quick-create modals)           |
|  Main Content: Routed Initiative Workspace                                                                      |
====================================================================================================================
          │
          │ layout.tsx renders transformation routes here (header spacing via --app-header-h)
          ▼
+------------------------------------------------------------------------------------------------------------------+
| Initiative Workspace ( /transformations/:transformationId )                                                              |
|------------------------------------------------------------------------------------------------------------------|
|  ┌─────────────── Overview Pane ───────────────┐  ┌────────────── Board/Table View ───────────────┐             |
|  │ • Status summary cards                      │  │ • Columns grouped by lifecycle / role          │             |
|  │ • Upcoming milestones (timeline preview)    │  │ • Cards: title, role owner, governance badges   │             |
|  │ • Role roster & directory sync status       │  │ • Drag-and-drop with optimistic updates         │             |
|  │ • Recently updated artifacts (deep-link)    │  │ • Filters: role, lifecycle, risk, scenario      │             |
|  └─────────────────────────────────────────────┘  │ • Saved views (e.g., “Needs approvals”)         │             |
|                                                   │ • Quick actions: link artifact, assign owner    │             |
|                                                   └─────────────────────────────────────────────────┘             |
|                                                                                                                  |
|  ┌────────────── Roadmap / Scenario View ───────────────┐  ┌────────────── KPI / Dashboard Pane ───────────────┐ |
|  │ • Timeline bars (planned vs actual)                  │  │ • Charts: milestone completion %, cycle time     │ |
|  │ • Toggles: As-is | To-be | Scenario (Plan A/B)       │  │ • Check success rate, queue latency, data quality │ |
|  │ • Dependency overlays from graph `model_edges`  │  │ • Role coverage heatmap, survey completion        │ |
|  │ • Hover tooltips: affected systems, owners           │  │ • Export/report shortcuts                          │ |
|  └─────────────────────────────────────────────────────┘  └─────────────────────────────────────────────────────┘ |
|                                                                                                                  |
|  ┌────────────── Settings / Admin Drawer ──────────────┐                                                         |
|  │ • Milestone template management                     │                                                         |
|  │ • Role assignment (directory search + manual)       │                                                         |
|  │ • Notification & survey preferences                 │                                                         |
|  │ • Scenario definitions (tags, descriptions)         │                                                         |
|  └─────────────────────────────────────────────────────┘                                                         |
+------------------------------------------------------------------------------------------------------------------+

Legend:
- Overview anchors summary/KPIs; Board/Table manage execution; Roadmap handles scenario planning; KPI pane aggregates governance/delivery metrics; Settings configures milestones, roles, notifications.
```

## Data Flow & State Management
- **Server interactions**: `InitiativeService` (transformation/milestone/role CRUD), `InitiativeWorkspaceService` (board/table/roadmap datasets), graph governance APIs (checks, approvals, queue), notification/survey endpoints.
- **React Query caches**: transformations, milestones, role assignments, artifact links, board columns, roadmap datasets, governance snapshots. Keys scoped by tenant+transformation; invalidated on mutations and `transformation.*`/`graph.*` events.
- **Local state**: view mode (board vs table), active filters (role, lifecycle, risk), scenario selection, saved view definitions, column/order preferences.
- **Realtime updates**: SSE/WebSockets stream `transformation.role.updated`, `transformation.milestone.updated`, governance events; board/roadmap re-render optimistically. Presence indicators (optional) for concurrent users editing milestones.

## Module Responsibilities
- **Initiative Overview**: Summaries (status, KPIs, upcoming milestones), role roster, latest releases, quick links into graph artifacts/discussions.
- **Board/Table View**: Kanban columns, list layout, drag-and-drop progression, per-card governance badges (checks, approvals, queue), inline actions (assign role/agent, link artifact), saved views for Product Owners (e.g., Release Readiness, Blocked checks, Needs approvals).
- **Roadmap & Scenario View**: Timeline visualization with As-is/To-be toggles, scenario overlays, dependency highlights from graph graph; supports zoom, grouping by workstream.
- **KPI & Dashboard Pane**: Charts/tables for milestone progress, approval cycle time, check success rate, data-quality scores, release readiness metrics; filters by role, phase, scenario.
- **Settings Drawer**: Manage milestones, assign roles (people + AI agents via directory/registry search), configure notifications, define scenarios, manage surveys, scope AI agent permissions (read-only, comment, check-run).
- **Product Owner Views**: Release readiness reports (artifact, version, required checks, approvals, queue status), blockers funnel ranked by outstanding approvals/check failures, exportable Go/No-Go packet.

## UI Patterns & Components
- Reuse shadcn/ui primitives (Tabs, Drawer, Dialog, Dropdowns) plus custom components: transformation-card, governance-badge, timeline-bar, scenario-badge, role-roster list.
- Boards use virtualized lists for large datasets; roadmap uses D3/visx timeline component with tooltip overlays.
- Quick links deep-link to graph artifact drawer/timeline (new tab or side panel); AI agent summaries or recommendations display alongside human comments when available.

## Governance & Collaboration UX
- Cards display governance badges (checks pending, approvals outstanding, queue position) derived from graph events; tooltips list required actions and responsible owners.
- Discussions remain artifact-centric—board cards provide “Open discussion” CTA that deep-links to graph artifact timeline (human + AI agent contributions distinguished via badges).
- Governance badges differentiate human approvals vs AI advisory checks to maintain transparency and trust.
- Role avatars indicate assignment health (e.g., missing role highlighted). Directory sync warnings appear inline when principal changes are detected.

## Performance & Offline Strategy
- Incremental data fetch per board column (lazy loading) with optimistic UI updates on drag-and-drop.
- Prefetch artifact/governance details on hover to minimize drawer latency.
- Offline indicators when mutations fail; queued actions retried automatically with exponential backoff.
- SLO alignment: board column load and scenario toggle interactions target <300 ms perceived latency; P95 <500 ms under 95th percentile tenant load (matching graph SLO wording).

## Telemetry & Feature Flags
- Track board interactions, scenario toggle usage, KPI filter changes, role assignment edits, roadmap zoom actions.
- Feature flags: `transformation_board_view`, `transformation_table_view`, `transformation_roadmap`, `transformation_kpi_dashboards`, `transformation_directory_sync` control rollout per tenant.

# Next Steps
- Validate personas and workflows with product stakeholders.
- Detail transformation data model and API contracts.
- Align transformation templates with graph templates for consistent taxonomy.
- Engage identity team to design directory integration API contract for people/role assignments.
- Plan transformation workspace Stage 5 stories for As-is / To-be / Scenario toggles and role-driven boards leveraging graph assignment metadata and governance outputs.
