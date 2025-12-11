Below is a **method‑aware UI/UX structure** that plugs cleanly into the shell you already drafted, while giving Agile/Scrum and TOGAF/ADM their own native “mental models.” I’m building on your current layout, routing, and threads patterns so you can implement this without re‑architecting the app chrome. 

---

## 0) Guiding principles (fits your app shell)

* **One shell, many methods.** The *global header* stays fixed with scope trail, search, actions, and context‑aware tabs. The *subheader* (just under the tabs) is a compact, data‑driven strip for filters, view toggles, and quick “Create” actions. Your `HeaderClient` + `NavTabs` already set up the two‑row header; treat the *PageHeader* row on each page as the *section‑specific subheader*. 
* **Right-side “Sidecar”** = your existing **Chat panel** evolved: a single side panel that can swap between **Chat**, **Widgets**, and **Tabs** (Details, Links, History, Checks, Automation). When *Active*, it reduces main content width—exactly how your `ChatLayoutWrapper` works today. 
* **Artifacts behave like files.** Every deliverable, diagram, backlog item, capability, etc. is a **versioned file** with lifecycle, history, review, and checks (a GitHub‑like PR model). Your **Repository Browser** and **element modal** are the baseline UX here. 

---

> **STATUS — Guiding Principles:**
> **Impact:** High — foundational architecture patterns that shape all transformation UX
> **Implementation:** 60% complete
> - ✅ Global header with scope trail and tabs (`HeaderClient` + `NavTabs`)
> - ✅ Chat panel foundation exists (`ChatLayoutWrapper` with Active/Minimized/Closed states)
> - ✅ Repository Browser three-column pattern (tree + list + sidebar)
> - ✅ Element modal for artifact editing
> - ⚠️ **Subheader row not yet standardized** — PageHeader exists but lacks consistent filter/action strip
> - ❌ **Sidecar not generalized** — still threads-specific, not tab-based (Details/Links/History/Checks/Widgets)
> **Deviations:** Current implementation treats threads as standalone feature rather than one tab in a unified sidecar
> **Considerations:** Generalizing ChatLayoutWrapper to Sidecar is a quick architectural win that unblocks multiple features

---

## 1) Method Packs: a pluggable contract

Introduce a *Method Pack* interface so each methodology registers its own navigation, views, artifact templates, and lifecycle—without breaking the shell.

```ts
type MethodPack = {
  key: 'agile' | 'togaf' | 'safe' | 'custom';
  label: string;
  // Tabs that appear under the global header in this context
  tabs: { href: string; label: string; icon?: any }[];
  // Page templates this method exposes
  views: {
    key: 'backlog'|'board'|'gantt'|'canvas'|'matrix'|'catalog'|'browser';
    route: string;                    // e.g., /planning/backlog
    chrome: 'table'|'board'|'diagram'|'timeline';
    sidecar: ('details'|'links'|'history'|'checks'|'threads'|'widgets')[];
    defaultFilters?: Record<string, any>;
  }[];
  // Artifact kinds & lifecycles
  artifacts: {
    kind: string;                     // 'epic','story','vision','catalog','diagram'
    templateId: string;
    lifecycle: ('draft'|'in_review'|'approved'|'deprecated')[];
    checks?: string[];                // governance checks to run/show
  }[];
};
```

Bind a `MethodPack` to each **Initiative** at creation (and allow mixing advanced packs later). It plugs into your **server‑side navigation descriptor** so tabs and breadcrumbs are resolved *before* render (RBAC still applies).

> **STATUS — Method Packs:**
> **Impact:** High — enables multi-methodology support (Agile, TOGAF, SAFe, custom)
> **Implementation:** 0% — concept not yet implemented
> - ✅ Database supports `transformation_cadence` enum (ADM, AGILE, HYBRID) in V2 migration
> - ✅ Server-side navigation descriptor pattern exists (`NavigationDescriptor` with RBAC)
> - ❌ No MethodPack registry or pluggable tab system
> - ❌ No UI to select method at transformation creation
> - ❌ Tabs are currently hardcoded: Workspace | Planning | Architect | Build | Measure | Governance | Settings
> **Deviations:** Current approach uses fixed tab structure for all transformations regardless of methodology
> **Considerations:** Method packs would require refactoring navigation builders to be data-driven; backend already has cadence enum as foundation

---

## 2) Agile/Scrum: "Product‑backlog–driven" flow

**Where it lives:** transformation scope under `/planning` and `/build`, reusing your existing transformation tab rail: *Workspace · Planning · Architect · Build · Measure · Governance*. 

**Tabs (transformation level for Agile transformations)**
`[Workspace] [Planning] [Build] [Measure] [Governance] [Settings]` (Architect can remain if you want hybrid EA + Agile).

**Sub‑nav (section‑specific) under Planning:**

* **Backlog** (table view; primary place)
* **Board** (Kanban: Sprint / Status)
* **Sprints** (list of sprints; draw from Backlog scope)
* **Roadmap** (timeline; high‑level epics)
* **Releases** (cut releases, link to deployments)

**Page templates**

1. **Backlog (table chrome)**

* **Header/Subheader:** search, filters (Epic, Status, Component, Priority), quick “New” menu (Epic/Story/Bug), “Bulk edit,” “Rank mode.”
* **Main:** dense, keyboard‑friendly grid (inline assignee, points, sprint).
* **Right Sidecar tabs:** *Details* (YAML/meta), *Links* (trace to graph elements & TOGAF deliverables), *History*, *Checks* (DoR/DoD), *Chat*.
* **Artifact behavior:** Epics/stories are files with lifecycle (*draft → ready → in_progress → done* mapped to generic states). “Propose change” opens a PR‑style review using your element modal pattern. 

2. **Board (board chrome)**

* Columns by Status or by Sprint; cards show title, points, badges; drag‑drop updates state/sprint.
* Sidecar mirrors Backlog (Details/Links/History/Checks/Chat).

3. **Sprints (simple data page)**

* Sprint list + velocity trend mini‑charts. Clicking a sprint routes to `/planning/sprints/[sprintId]` with Backlog subset + *Burndown* widget in the sidecar *Widgets* tab.

4. **Roadmap (timeline chrome)**

* Epics across quarters. Sidecar: *Dependencies* graph widget, *Checks* (date guardrails), *Chat* (“summarize critical path”).

5. **Releases**

* Release page links to `/build/deployments` you already envisioned. 

**Routing (adds onto your current transformation subtree)**

```
/org/:orgId/transformations/:id/planning/backlog
/org/:orgId/transformations/:id/planning/board
/org/:orgId/transformations/:id/planning/sprints
/org/:orgId/transformations/:id/planning/sprints/:sprintId
/org/:orgId/transformations/:id/planning/roadmap
/org/:orgId/transformations/:id/planning/releases
```

This mirrors your "Common Layout Pattern" and transformation tab rail.

> **STATUS — Agile/Scrum Flow:**
> **Impact:** High — critical for product teams and software delivery organizations
> **Implementation:** 5% — placeholder routes only
> - ✅ `/planning` route exists with placeholder page
> - ❌ No backlog table implementation
> - ❌ No board/kanban view
> - ❌ No sprint management
> - ❌ No roadmap timeline
> - ❌ No release tracking
> - ❌ No workflows integration for story/epic lifecycle
> - ❌ No points, assignee, rank, or sprint fields in database
> **Deviations:** Spec assumes Planning is equal peer to Architect; current UI shows all 6 tabs but only Architect has content
> **Considerations:** Requires separate backlog schema (stories/epics/sprints tables) or extending `transformation_workspace_elements` with agile metadata; workflows system could drive status transitions

---

## 3) TOGAF/ADM: "Phase‑based, project/diagramming" flow

**Where it lives:** your **Initiative Workspace page** already presents TOGAF phases in a collapsible left tree with a deliverables browser center pane and sidebar—keep that. It’s the perfect Visio/Asana hybrid. 

**Tabs (transformation level for TOGAF transformations)**
`[Workspace] [Architect] [Governance] [Measure] [Settings]` (Planning/Build optional).

**Phase navigation (left rail)**

* Prelim, A … H + Requirements Mgmt (your structure is spot on). Selecting a phase filters the **Deliverables Browser** center pane. The *subheader* offers New (Document/Diagram/Catalog), Import, and View toggles. 

**Triad views per phase (toggle in subheader)**

* **Catalog (table)** — principles, stakeholders, capability catalogues.
* **Matrix (matrix chrome)** — relationships (e.g., Capability ↔ Application).
* **Diagram (canvas chrome)** — ArchiMate/BPMN canvas.

**Diagram canvas (Visio‑style)**

* Left palette: ArchiMate/BPMN stencils; top toolbar: alignment, group, theme; mini‑map.
* Right Sidecar: *Properties* (graph properties), *Links* (trace to catalog rows & graph elements), *History* (diagram version diffs), *Checks* (graph validation), *Chat* (“explain this view,” “suggest missing relations”).
* Use your **element editor modal** conventions for diagram nodes: the node inspector is the *Details* tab of the sidecar. 

**Deliverables browser**

* Exactly your three‑column pattern (tree · files · sidebar), but the left tree is the **ADM tree** (phase → folders). The file rows show type badges: *Catalog*, *Matrix*, *Diagram*, *Document*. Click opens preview or full editor page; hover still prefetches (keep your perf pattern). 

**Routing (aligned to your existing)**

```
/org/:orgId/transformations/:id (workspace)
/org/:orgId/transformations/:id/architect/canvas/:diagramId
/org/:orgId/transformations/:id/architect/catalogs/:catalogId
/org/:orgId/transformations/:id/architect/matrices/:matrixId
```

These live under your `/architect` section and reuse the "PageHeader + content + sidecar" pattern.

> **STATUS — TOGAF/ADM Flow:**
> **Impact:** High — core value prop for enterprise architects
> **Implementation:** 40% complete
> - ✅ Initiative Workspace page with three-column layout (ADM tree + deliverables browser + sidebar)
> - ✅ Database: `transformation_workspace_nodes` table with sample ADM phase hierarchy (Prelim, A-C)
> - ✅ Database: `transformation_workspace_elements` links elements to workspace nodes (V24 migration)
> - ✅ Backend: `listWorkspaceNodes()` and `listWorkspaceElements()` APIs
> - ✅ Frontend: Tree navigation with phase collapse/expand
> - ✅ WorkspaceSidebar with Phase Progress, Deliverables Status, Team
> - ⚠️ **Read-only** — no create/edit/organize workspace nodes
> - ❌ No Catalog/Matrix/Diagram view toggles (spec's "triad views")
> - ❌ No diagram canvas editor integration
> - ❌ No matrix chrome for relationship views
> - ❌ No catalog table view with inline editing
> **Deviations:** Current implementation is browser-centric rather than editor-centric; element modal opens as overlay but doesn't link back to workspace context
> **Considerations:** Workspace editing requires CRUD APIs for nodes; diagram/catalog/matrix views could reuse existing element editor infrastructure with workspace awareness

---

## 4) Unifying idea: the **Artifact Contract** (GitHub‑style files)

No matter the method, **everything is a file**:

**Header strip (inside PageHeader)**

* `Title · Type badge · State badge (Draft/In review/Approved/Deprecated)`
* Primary actions: *Edit*, *Propose change*, *Request review*, *Export*

**Core tabs (center body)**

* `Overview` (content/preview) · `Discussions` (inline comments; PR threads) · `Checks` (governance status) · `Versions` (diffs between commits) · `Activity` (events)
* Each “Propose change” opens a **review** that looks and feels like a PR and can be merged back into a **Repository** (for as‑is truth). Your Repository and modal editor already align with this. 

**Sidecar tabs (right)**

* `Details` (metadata fields per artifact type)
* `Links` (traceability graph: *Backlog item ↔ Capability/Component ↔ Diagram nodes ↔ Standards ↔ Deployments*)
* `Checks` (status checks; e.g., “All stories have DoD”, “Diagram passes graph rules”)
* `History` (audit trail)
* `Chat` / `Widgets` (AI help or small dashboards)

This stays consistent whether you're inside **Backlog**, **Board**, **Catalog**, **Diagram**, or **Repository Browser**.

> **STATUS — Artifact Contract (GitHub-style Files):**
> **Impact:** Very High — unifies all deliverables under consistent lifecycle and review model
> **Implementation:** 20% complete
> - ✅ Database: All artifacts stored as elements with type badges
> - ✅ Element modal pattern exists with edit mode
> - ✅ Repository browser supports file-like navigation
> - ⚠️ **Lifecycle states** — element status exists but not formalized as Draft/In Review/Approved/Deprecated
> - ❌ No "Propose change" workflows (PR-style review flow)
> - ❌ No "Request review" action
> - ❌ No inline comment/discussion threads
> - ❌ No version diff viewer
> - ❌ No merge/approval actions
> - ❌ No Checks tab (governance validation)
> - ❌ No Activity tab (audit trail)
> - ❌ Links tab not implemented (traceability)
> **Deviations:** Current model is direct edit rather than propose-review-merge; elements don't have explicit review state
> **Considerations:** Requires workflows integration for review lifecycle; version history exists in database (`element_versions` table from threads context) but no UI; traceability needs graph storage and query layer

---

## 5) Section‑specific **Subheader** patterns

Place this immediately below your tab row (can be the first row of `PageHeader` content):

* **Left cluster:** `View toggle` (Table/Board/Canvas/Matrix/Timeline), `Scope filter` (phase/sprint), `Quick filters` (assignee, status, priority), `Search` (local to page).
* **Right cluster:** `New` (contextual), `Bulk`, `Export`, `Sidecar switcher` (Details | Widgets | Chat).
  This mirrors your **PageHeader** API and keeps pages simple and data‑driven.

> **STATUS — Subheader Patterns:**
> **Impact:** Medium — improves usability and reduces page clutter
> **Implementation:** 30% complete
> - ✅ PageHeader component exists with title, badges, actions
> - ⚠️ No standardized subheader row with view toggles and filters
> - ❌ View toggle UI not implemented (Table/Board/Canvas/Matrix/Timeline)
> - ❌ Scope filter not standardized (phase/sprint selection)
> - ❌ Quick filters not implemented (assignee, status, priority dropdowns)
> - ❌ Local search not scoped to page context
> - ❌ Contextual "New" menu not wired (would show relevant artifact types)
> - ❌ Bulk operations not supported
> - ❌ Sidecar switcher not in header (Details | Widgets | Chat)
> **Deviations:** Current PageHeader is single-row title + badges; filtering happens in dedicated sections rather than compact subheader strip
> **Considerations:** Subheader could be a slot in PageHeader component; would reduce vertical space and make filters always accessible; needs design system work for consistent filter UI

---

## 6) The **Sidecar** (right panel) as a first‑class layout

You already implement three threads modes; generalize it to **Sidecar modes**:

* **Closed / Minimized / Active:** exactly your threads layout states. When Active, render a tab bar at the top: `Details | Links | History | Checks | Widgets | Chat`. Persist the last tab per route in URL or local state; your `ChatLayoutWrapper` can continue owning the width + z‑index transitions.

> **STATUS — Sidecar (Right Panel):**
> **Impact:** High — enables contextual information without navigation disruption
> **Implementation:** 35% complete (threads-specific only)
> - ✅ ChatLayoutWrapper with three states (Closed/Minimized/Active)
> - ✅ Width transitions and z-index management
> - ✅ Main content area adjusts when sidecar active
> - ✅ Footer with threads starters when closed
> - ⚠️ **Chat-only** — not generalized to tabbed sidecar
> - ❌ No Details tab (metadata fields)
> - ❌ No Links tab (traceability graph)
> - ❌ No History tab (audit trail)
> - ❌ No Checks tab (governance status)
> - ❌ No Widgets tab (small dashboards)
> - ❌ Tab state not persisted in URL
> - ❌ WorkspaceSidebar is separate component, not integrated with sidecar pattern
> **Deviations:** Current architecture has two sidebars: WorkspaceSidebar (left/embedded in workspace) and Chat (right/overlay); spec envisions single right sidecar with multiple tabs
> **Considerations:** Refactoring ChatLayoutWrapper to SidecarLayout with <SidecarTabs> would be high-value; could migrate WorkspaceSidebar content into Details tab; minimal breaking changes if done carefully

---

## 7) Page library (drop‑in templates)

All pages use **PageHeader → Subheader → Content → Sidecar**:

1. **Table page**: Backlog, Catalogs, Standards, Repositories.
2. **Board page**: Kanban for stories or governance approvals.
3. **Timeline page**: Roadmaps, Releases, Milestones.
4. **Canvas page**: ArchiMate/BPMN editor.
5. **Matrix page**: n×m relations with inline counts and drill‑downs.
6. **Browser page**: Your **Repository Browser**—tree + list + sidebar. 

Each template wires the sidecar's default tab and a minimal set of filters so pages stay "simple & data‑driven."

> **STATUS — Page Library (Templates):**
> **Impact:** Medium — accelerates feature development with consistent patterns
> **Implementation:** 50% complete (patterns exist but not formalized as reusable templates)
> - ✅ **Table page:** Repository Browser uses table/list pattern; element browser in transformations
> - ⚠️ **Board page:** UI components exist (from navigation) but no Kanban drag-drop implementation
> - ❌ **Timeline page:** No Gantt/roadmap chrome implemented (needed for sprints, milestones, releases)
> - ✅ **Canvas page:** Element editor modal has canvas support (ArchiMate/BPMN)
> - ❌ **Matrix page:** No n×m relationship grid component
> - ✅ **Browser page:** Repository Browser three-column pattern (fully implemented)
> - ⚠️ InitiativeSectionShell provides data-fetching template but not layout chrome
> **Deviations:** Pages are built ad-hoc rather than from template library; no shared chrome components for Board/Timeline/Matrix
> **Considerations:** Creating formal PageTemplate components (TablePageTemplate, BoardPageTemplate, etc.) would reduce duplication; could be part of design system with slots for header/filters/content/sidecar

---

## 8) Cross‑method traceability (the real superpower)

* **Link anything to anything** using the Sidecar *Links* tab (stored as graph edges).
* **Show related items inline**: e.g., a Story shows the *Capabilities* it impacts and the *Diagrams* where those components appear.
* **Checks** can span methods: “All stories linked to at least one Capability,” “All Phase A deliverables have owners,” “All approved diagrams map to standards.”

The **Context Cache** you already populate per page can enrich Sidecar and Chat with "what's nearby."

> **STATUS — Cross-Method Traceability:**
> **Impact:** Very High — differentiating feature that connects Agile delivery with EA governance
> **Implementation:** 10% complete
> - ✅ Database: `transformation_workspace_elements` links elements to workspace nodes (foundation for traceability)
> - ✅ Context cache populates transformation metadata for threads enrichment
> - ⚠️ Element metadata includes workspace_node_ids but no UI to display/navigate
> - ❌ **No Links tab in sidecar** — core traceability UI missing
> - ❌ No graph storage for arbitrary relationships (Story ↔ Capability ↔ Diagram Node ↔ Standard)
> - ❌ No traceability queries or APIs
> - ❌ No inline "Related items" sections in artifact views
> - ❌ No graph visualization component
> - ❌ No cross-method checks (e.g., "All stories linked to capability")
> - ❌ No impact analysis ("What depends on this capability?")
> **Deviations:** Current focus is single-artifact views without relationship context
> **Considerations:** Requires graph database or adjacency table for flexible relationships; Links tab would query graph and render interactive network; could leverage existing element_references pattern from threads; big architectural decision on graph storage (Neo4j, Postgres with recursive CTEs, or simple link table)

---

## 9) Routing & nav glue (minimal changes)

You already have the **org → transformation → graph/architect** hierarchy and server‑side **NavigationDescriptor**: extend it with *MethodPack* awareness so the **tabs** and **sub‑tabs** reflect Agile vs TOGAF automatically. (RBAC and feature flags continue to filter tabs server‑side.)

> **STATUS — Routing & Nav Glue:**
> **Impact:** High — enables method-aware navigation and subnav
> **Implementation:** 60% complete
> - ✅ Org → Initiative → Repository hierarchy established
> - ✅ Server-side NavigationDescriptor with RBAC filtering
> - ✅ Breadcrumbs (scope trail) working
> - ✅ Main tabs: Workspace | Planning | Architect | Build | Measure | Governance | Settings
> - ⚠️ **Tabs are fixed** — not driven by MethodPack or transformation.cadence
> - ❌ No subnav under sections (e.g., Planning → Backlog | Board | Sprints)
> - ❌ No dynamic tab registration based on method
> - ❌ NavigationDescriptor doesn't consume transformation cadence field
> **Deviations:** Current nav is static and shows all tabs to all transformations; spec envisions Agile transformations hide Architect, TOGAF transformations hide Planning/Build
> **Considerations:** Extending navigation builders to read transformation.cadence would enable method-aware tabs; requires refactoring InitiativeNavigation to be data-driven; subnav could be additional NavigationLevel in descriptor pattern

---

## 10) Governance as status checks (GitHub‑like)

Every artifact and every “change proposal” runs **Checks** (lint rules, templates, review gates). Surface the aggregate status in:

* **PageHeader** (a compact “Checks: ✅/⚠️/❌” pill with a dropdown).
* **Sidecar » Checks** (full list with pass/fail and links).
* **Lists/Boards** show a small dot/badge per row/card to telegraph readiness.

This pattern matches your PR‑style review and keeps pages simple but guided.

> **STATUS — Governance as Status Checks:**
> **Impact:** Very High — makes governance proactive rather than reactive
> **Implementation:** 15% complete
> - ✅ Database: `transformation_alerts` table with severity, alert_type, triggered_at fields
> - ✅ Backend: `listInitiativeAlerts()` API
> - ✅ Frontend: InitiativeSectionShell can fetch alerts
> - ⚠️ Alerts display-only, no triggering logic
> - ❌ No validation rules engine
> - ❌ No Checks tab in sidecar
> - ❌ No check status pills in PageHeader
> - ❌ No per-artifact check badges in lists/boards
> - ❌ No DoR/DoD (Definition of Ready/Done) enforcement for stories
> - ❌ No graph validation for diagrams
> - ❌ No template compliance checks for documents
> - ❌ No review gate enforcement at milestones
> - ❌ No policy evaluation service
> **Deviations:** Alerts are manual/reactive; spec envisions automated checks that run on artifact save/submit
> **Considerations:** Checks system needs rule engine (could extend workflows system); each artifact type needs check definitions (stories require DoD fields, diagrams validate against graph); governance service could evaluate rules and write to alerts table; UI is straightforward once backend exists

---

## 11) Small but high‑impact UI details

* **Scope trail** (you already have) doubles as navigation chips: Org › Initiative › Repo › Element. Add hover cards with key metrics. 
* **Command palette (⌘K)** to jump to phases, sprints, artifacts (listed as a planned improvement—lean into it). 
* **Keyboard first** in Backlog and Catalog tables; *J/K* to move, *Enter* to open sidecar.
* **URL‑addressable state** for selected artifact + sidecar tab (you already sync Chat IDs; extend the same idea).

> **STATUS — Small but High-Impact UI Details:**
> **Impact:** Medium-High — polish that elevates UX significantly
> **Implementation:** 40% complete
> - ✅ Scope trail (breadcrumbs) with Org › Initiative › Repo navigation
> - ⚠️ No hover cards with metrics on breadcrumb items
> - ❌ **Command palette (⌘K)** — not implemented (planned improvement mentioned in codebase)
> - ❌ No keyboard navigation in tables (J/K, Enter to open sidecar)
> - ⚠️ URL includes transformationId and page route but not selected artifact or sidecar tab
> - ❌ Chat ID syncs in URL but artifact/element selection doesn't
> - ❌ No keyboard shortcuts documented or implemented (Vim-style navigation)
> **Deviations:** Navigation is mouse-centric; no power-user keyboard workflows
> **Considerations:** Command palette would be high-value quick win (many UI libraries provide this); keyboard nav requires focus management and event handlers in table/list components; URL state for artifact selection enables deep linking and back/forward navigation

---

## 12) How this maps to what you already have (quick wins)

* **Header & tabs** — keep your `HeaderClient` + `NavTabs`; add a slim subheader row inside `PageHeader` with view toggles and filters. 
* **Initiative Workspace** — your **ADM phases** tree + Deliverables Browser is the TOGAF “home”. Add the Catalog/Matrix/Diagram triad toggle. 
* **Repository Browser** — stays the canonical “as‑is” home for elements and documents; ensure *Propose change* opens a review (PR‑like). 
* **Sidecar** — generalize your Chat panel to a tabbed Sidecar. Keep the footer threads starters when closed; when open, Chat is just one tab. 
* **Routing** — add `/planning/*` for Agile and `/architect/*` detail routes for TOGAF artifacts; everything fits your existing route groups and layout hierarchy.

> **STATUS — Current Mapping (Quick Wins):**
> **Impact:** High — validates that spec is achievable with minimal architectural changes
> **Implementation:** Confirms alignment between spec and current codebase
> - ✅ Header + tabs structure matches spec exactly (`HeaderClient` + `NavTabs` + future subheader slot)
> - ✅ Initiative Workspace ADM tree + deliverables browser matches TOGAF home vision
> - ✅ Repository Browser is solid foundation for "as-is" truth and element files
> - ⚠️ Sidecar generalization is single highest-leverage refactor (threads → tabs)
> - ✅ Routing structure supports `/planning/*` and `/architect/*` additions (placeholder routes exist)
> - ✅ PageHeader pattern ready for lifecycle badges and check status
> - ✅ NavigationDescriptor pattern can extend to method-aware tabs
> **Deviations:** None — spec was written with deep understanding of current implementation
> **Considerations:** Spec provides clear implementation path; prioritize Sidecar refactor, then Backlog table, then Method Pack system; most pieces are "wiring and UI" rather than new architecture

---

### Example screen sketches (ASCII)

**Backlog (Agile)**

```
[Header + Tabs]
[Subheader: View=Table | Filters | New ▼ | Sidecar: Details|Widgets|Chat]

┌ Table: Backlog ----------------------------------------------------------┐ ┐
│  ▸ Rank  Title                Epic       Sprint     Assignee   Pts  ⚑  │ │
│  1       Checkout—add PayPal  Payments   Sprint 14  Alice      5    ✅ │ │
│  2       Service mesh POC     Platform   Backlog    Bob        8    ⚠︎ │ │
└─────────────────────────────────────────────────────────────────────────┘ │
                                                                            │
                          [Sidecar Tabs]  Details | Links | History | Checks │
                          ┌─────────────────────────────────────────────────┐│
                          │ Details: fields …                               ││
                          └─────────────────────────────────────────────────┘┘
```

**ADM Canvas (TOGAF)**

```
[Header + Tabs]
[Subheader: View=Canvas | Phase=A | New Diagram ▼ | Validate | Sidecar=… ]

┌ Left: ADM Tree ┐ ┌────────────── Canvas ──────────────┐ ┐ Sidecar Tabs ┐
│ Prelim         │ │ [Palette][Toolbar]  [Mini-Map]     │ │ Details …    │
│ ▾ Phase A      │ │                                      │ │ Links …      │
│   Vision       │ │  Diagram nodes + edges               │ │ Checks …     │
│ ▸ Phase B      │ └──────────────────────────────────────┘ └──────────────┘
```

---

## 13) Rollout order

1. **Sidecar generalization** (minimal code: reuse Chat panel container + add tab bar).
2. **Backlog table page** (reuses your list + sidecar patterns).
3. **ADM triad toggles** on the existing Initiative Workspace.
4. **Artifact Contract** (surface lifecycle + Checks in PageHeader and Sidecar).
5. **Method Packs** plumbing (server‑side nav descriptor extends to include per‑method tabs).

> **STATUS — Rollout Order:**
> **Impact:** High — provides clear prioritization for implementation
> **Assessment:** Well-sequenced based on dependencies and value delivery
> **Recommendations based on current state:**
> 1. ✅ **Sidecar generalization** — Highest leverage, unblocks multiple features, minimal risk (refactor existing ChatLayoutWrapper)
> 2. ✅ **Backlog table page** — High value for Agile users, moderate complexity (needs schema + APIs + table component)
> 3. ✅ **ADM triad toggles** — Good incremental win on existing workspace page (adds Catalog/Matrix/Diagram views)
> 4. ⚠️ **Artifact Contract** — High impact but requires workflows integration for PR-style review; consider phasing (lifecycle badges first, then review workflows)
> 5. ⚠️ **Method Packs** — Strategic but complex; consider starting with simpler cadence-based tab filtering before full pluggable architecture
> **Additional priorities not in spec:**
> - Initiative creation UI (prerequisite for user testing)
> - Workspace node CRUD APIs (enables organizing deliverables)
> - Traceability Links tab (high differentiating value)
> - Command palette (quick win for usability)
> **Considerations:** Rollout order balances architectural cleanup (Sidecar) with feature delivery (Backlog); consider parallel tracks for UI polish (keyboard nav, command palette) and core features

---

> **OVERALL IMPLEMENTATION STATUS — Initiatives Feature:**
> **Foundation:** 70% complete — Database schema, protobuf contracts, backend services, and basic UI structure fully implemented
> **User-Facing Features:** 25% complete — Initiative list and workspace browser working; Planning/Build/Measure sections are placeholders
> **Advanced Capabilities:** 10% complete — Traceability, governance checks, workflows integration, and method packs are conceptual
>
> **Critical Path to MVP:**
> 1. Initiative creation UI + workspace template auto-generation
> 2. Sidecar refactor (Chat → Details|Links|History|Checks|Widgets|Chat tabs)
> 3. Backlog table for Agile teams (stories/epics schema + APIs + UI)
> 4. Workspace editing (create/move/organize deliverables)
> 5. Links tab for traceability (graph storage + query + visualization)
>
> **Architecture Decisions Needed:**
> - Graph storage for traceability (Postgres recursive CTEs vs separate graph DB)
> - Workflow engine integration for artifact lifecycle and review
> - Rule engine for governance checks (extend workflows or separate service)
> - Method Pack extensibility model (config-driven vs plugin system)
>
> **No Blockers:** Current architecture supports all features in spec; work is primarily UI implementation and integration

---

If you'd like, I can turn this into a small **UI kit checklist** (components, props, and states) mapped to your file tree so engineering can pick up issues straight from the spec.
