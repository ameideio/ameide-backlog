# 617 — Transformation UISurface Wireframes (UI/UX Only)

**Status:** Active (target UX; implementation may lag)  
**Audience:** Product/UI engineers, platform engineers, capability teams  
**Scope:** Transformation routes, pages, interactions, and live-update behavior (no backend schema details)

This backlog defines the **UISurface experience** for Transformation (R2R) as a projection-driven product UI.

## Cross‑references (normative)

- `backlog/520-primitives-stack-v6.md` (primitives + planes)
- `backlog/513-uisurface-primitive-scaffolding.md` (UISurface shell/canvases/widgets)
- `backlog/527-transformation-e2e-sequence-v6.md` (Transformation phases/columns + execution sequence)
- `backlog/614-kanban-projection-architecture.md` (Kanban as a projection)
- `backlog/615-kanban-fullstack-reference.md` (Kanban full‑stack reference)
- `backlog/616-kanban-principles.md` (Kanban principles; interaction streams are out‑of‑band)
- `backlog/618-kanban-proto-contracts.md` (proto-first Kanban query + updates stream contracts)
- `backlog/496-eda-principles-v6.md` (facts vs intents; outbox; idempotency)
- `backlog/509-proto-naming-conventions-v6.md` (process progress facts identity + minimal vocabulary)
- `backlog/694-elements-gitlab-v6.md` (Enterprise Repository as GitLab repo; canonical artifacts as files)
- `backlog/701-repository-ui-enterprise-repository-v6.md` (Repository UI is Git-tree-first; inline-only relationships)
- `backlog/300-400/327-initiatives-ideas.md` (method-aware UI/UX structure; “sidecar” concept)

## UX invariants (non‑negotiable)

1. **All reads come from projections** (Kanban/timeline/status/details). The UI never uses orchestration-runtime visibility/history as product truth.
2. **Kanban is a view**: columns are derived; “move card” (if offered) emits a real domain/process intent and reconciles to projection truth.
3. **Live updates are cursor‑based**: UI subscribes to a “Projection Updates” stream and refetches deltas/board on cursor advance.
4. **Interaction streams are not facts**: chat/log streaming is out‑of‑band (SSE/WebSocket/server‑streaming RPC), linked by stable IDs.
5. **Activities are the interaction gateway**: clicking a card opens the correct “activity surface” (triage chat+editor, coding console, verification logs, release view).
6. **Contextual sidecar is always available** (chat/widgets/details) **except inside the artifact editor**, where chat is embedded in the editor.

## Concepts (as the UI sees them)

- **Repository**: the architecture context for work (Enterprise Repository; canonical artifacts are Git files).
- **Initiative**: a change initiative governed in a repository context (future: Scrum/TOGAF profiles).
- **Board**: a ProcessDefinition board rendered via the platform Kanban contract; scoped by `{tenant, org, repo, initiative?}` and `board_kind`.
- **Card**: a stable unit of work (default in process-driven boards: `card_id = process_instance_id`).
- **Activity**: a user-visible unit of work inside a process instance (triage, coding, verify, release).
- **Evidence**: links to artifacts/logs/PRs produced by activities (projection-rendered; not embedded streaming).
- **Artifact**: a Git file in the Enterprise Repository. Editing an artifact means working on a governed change branch/MR (no direct writes to `main`) (`backlog/694-elements-gitlab-v6.md`, `backlog/701-repository-ui-enterprise-repository-v6.md`).
- **Sidecar**: a unified right/bottom panel for contextual chat, widgets, and details (`backlog/300-400/327-initiatives-ideas.md`).

## Landing in the existing Ameide platform app (target)

This UX must land in the existing platform app layout and routing patterns:

- **Repository-first context**: canonical content lives in the Enterprise Repository (Git) scoped by `{tenant_id, organization_id, repository_id}` (`backlog/694-elements-gitlab-v6.md`).
- **Shell + Canvases + Widgets**: dashboards and boards are canvases composed of widgets (`backlog/513-uisurface-primitive-scaffolding.md`).
- **Sidecar**: a contextual panel (chat/widgets/details) is always available except inside the artifact editor (`backlog/300-400/327-initiatives-ideas.md`).

This backlog therefore defines **where the Transformation UX lives inside the app**, not a separate “Transformation app”.

## Navigation and routes (target)

The UISurface is a **shell** with canvases and widgets (`backlog/513-uisurface-primitive-scaffolding.md`).

Recommended routes:

- Transformation dashboard (widgets): `/org/:orgId/transformation`
  - widgets: repositories, stats, open initiatives
  - all widgets are projection-backed; no runtime visibility coupling
- Enterprise Repository (artifact browser): `/org/:orgId/repo/:repoId`
  - primary canvas is a Git-tree-first artifact browser over repo files (folders + files) per `backlog/701-repository-ui-enterprise-repository-v6.md`
  - canonical editors operate on repo files via governed change → commit → publish (branch/MR) per `backlog/694-elements-gitlab-v6.md`
- Initiatives index (all initiatives): `/org/:orgId/transformation/initiatives`
  - list/table view (not Kanban unless backed by a ProcessDefinition per `backlog/616-kanban-principles.md`)
  - initiatives are bound to repositories; items link into the repository context
- Initiative (one initiative, in repo context): `/org/:orgId/repo/:repoId/governance/initiative/:initiativeId`
  - initiative board (board_kind=initiative) + related artifacts view derived from Git files + inline references (no relationship artifacts; no “workspace assignments” write surface)
  - supports navigation into nested sub-initiative boards
- Change detail (R2R): `/org/:orgId/repo/:repoId/r2r/change/:changeId`
  - requirement/deliverables/evidence anchors + run controls
- Optional debug: “timeline” page (projection-only), not runtime UI: `/org/:orgId/repo/:repoId/timeline`
- Artifact editor (element editor): opened from any list/board by selecting an element
  - editor is full-screen and includes embedded chat; global sidecar is hidden while editor is open

## Page types and wireframes

### 0) Transformation dashboard (widgets)

Purpose: “enter Transformation” and see a dashboard of widgets (repositories, stats, open initiatives).

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Transformation                                                                │
│ [Dashboard] [Initiatives] [Settings]                                          │
├──────────────────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────┐ ┌──────────────────────────┐ ┌───────────────┐ │
│ │ Repositories             │ │ Stats                    │ │ Open initiatives│ │
│ │ - primary-architecture   │ │ - blocked: 3             │ │ - INIT-123      │ │
│ │ - sales                  │ │ - in progress: 7         │ │ - INIT-124      │ │
│ │ - finance                │ │ - release pending: 1     │ │                 │ │
│ └──────────────────────────┘ └──────────────────────────┘ └───────────────┘ │
│                                                                              │
│ Sidecar (always available): [Chat] [Widgets] [Details]                       │
└──────────────────────────────────────────────────────────────────────────────┘
```

Primary interactions:

- Open Enterprise Repository.
- Open initiatives Kanban index.
- Open a specific initiative (in repository context).

### 1) Enterprise Repository (artifact browser)

Purpose: browse and edit canonical repository artifacts (repo files; canonical in Git).

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Repository: primary-architecture                                              │
│ [Elements] [Kanban] [R2R] [Timeline] [Views]                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│ Search… [kind ▾] [type ▾]                                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│ Elements list                                                                 │
│ - Requirement.md   (doc)                                                      │
│ - Architecture view (archimate:view)                                          │
│ - Deliverables/ (folder)                                                      │
│                                                                              │
│ Sidecar (always available): [Chat] [Widgets] [Details]                        │
└──────────────────────────────────────────────────────────────────────────────┘
```

Primary interactions:

- Open file → Artifact editor (full-screen, embedded chat).
- Navigate to repo-scoped Kanban/timeline/R2R list.

**Normative (v6):**
- Repository hierarchy is the Git file tree (folders + files). No separate canonical “workspace node tree”.
- Relationships are inline-only; any relationship/impact views are derived projections (no relationship artifacts).

### 2) Initiatives index (Kanban across initiatives)

Purpose: see all initiatives (across repositories) as a Kanban and navigate into a chosen initiative.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Initiatives                                                                   │
│ Filters: [Repo ▾] [Method ▾] [Only blocked ☐]                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│ │ intake       │ │ triage        │ │ execution     │ │ release      │         │
│ │ INIT-123     │ │ INIT-124      │ │ INIT-125      │ │ …            │         │
│ │ (repo link)  │ │ (repo link)   │ │ (repo link)   │ │              │         │
│ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘         │
│                                                                              │
│ Sidecar (always available): [Chat] [Widgets] [Details]                        │
└──────────────────────────────────────────────────────────────────────────────┘
```

Primary interactions:

- Click initiative card → open its initiative page in repository context.

### 3) Initiative (Kanban + related artifacts)

Purpose: the main “work cockpit” for a change initiative in a repository.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Initiative INIT-123 (repo: primary-architecture)                              │
│ [Overview] [Kanban] [Artifacts] [Timeline] [Evidence]                         │
├──────────────────────────────────────────────────────────────────────────────┤
│ Filters: [Assignee ▾] [Phase ▾] [Only blocked ☐] [Search…]                   │
├──────────────────────────────────────────────────────────────────────────────┤
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│ │ intake       │ │ triage        │ │ execution     │ │ release      │         │
│ │ (derived)    │ │ (derived)     │ │ (derived)     │ │ (derived)    │         │
│ │ ┌──────────┐ │ │ ┌──────────┐  │ │ ┌──────────┐  │ │ ┌──────────┐ │         │
│ │ │ CARD A   │ │ │ │ CARD B   │  │ │ │ CARD C   │  │ │ │ CARD D   │ │         │
│ │ │ “Req…”   │ │ │ │ “Draft”  │  │ │ │ “Codex”  │  │ │ │ “Promo”  │ │         │
│ │ │ ⏳ in prog│ │ │ │ Awaiting │  │ │ │ ✅ tests │  │ │ │ ⛔ gate   │ │         │
│ │ └──────────┘ │ │ └──────────┘  │ │ └──────────┘  │ │ └──────────┘ │         │
│ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘         │
│                                                                              │
│ Live updates: subscribed (Projection Updates stream)                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

Artifacts tab behavior:

- Lists artifacts relevant to the initiative as derived projection outputs over Git files:
  - path-scoped files (when initiative scope is defined), and/or
  - files discovered via inline references/backlinks (relationships are inline-only).
- Opening an artifact opens the artifact editor (Git-backed; change/commit/publish).

Card badges (projection-derived):

- “⏳ in progress” means latest process progress facts show a running activity.
- “Awaiting” means `Awaiting(kind, ref)` is active.
- “Blocked” means `Blocked(reason)` is active.
- Terminal states: completed/failed/cancelled.

Primary interactions:

- Click a card → Activity Detail Drawer (below).
- Optional: “Move card” UX is allowed only as a real intent (e.g., approval decision), never as a local status change.

### 4) Activity Detail (sidecar panel; per card)

Purpose: provide a consistent “activity gateway” regardless of runtime (agent chat, codex console, verification logs).

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Kanban (left)                                          ┌───────────────────┐ │
│                                                        │ Card: CARD C      │ │
│                                                        │ Phase: execution  │ │
│                                                        │ Status: running   │ │
│                                                        ├───────────────────┤ │
│                                                        │ Timeline (facts)  │ │
│                                                        │ - PhaseEntered…   │ │
│                                                        │ - ToolRunRecorded │ │
│                                                        │ - Awaiting…       │ │
│                                                        ├───────────────────┤ │
│                                                        │ Evidence          │ │
│                                                        │ - PR link         │ │
│                                                        │ - logs (artifact) │ │
│                                                        │ - diff patch      │ │
│                                                        ├───────────────────┤ │
│                                                        │ Activity Surface  │ │
│                                                        │ [Open] [Stop]     │ │
│                                                        └───────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

Rules:

- Timeline is projection-driven (process facts + relevant domain facts).
- “Stop/Cancel” issues a process intent (Update/Signal/command), not a UI toggle.
- “Open” routes to the correct activity surface depending on the current activity kind.

### 4) Triage / stabilization surface (chat + artifact editor)

Purpose: interactive requirement stabilization as “editor-first” with chat assistance.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Triage: Requirement Stabilization (CARD B)                                    │
├───────────────────────────────┬──────────────────────────────────────────────┤
│ Chat (stream)                 │ Artifact Editor (requirement)                │
│ ┌───────────────────────────┐ │ ┌──────────────────────────────────────────┐ │
│ │ user: …                   │ │ │ Title: …                                 │ │
│ │ agent: … (streaming)      │ │ │                                          │ │
│ │                           │ │ │ [multimodal editor content…]             │ │
│ │ [input…]  [Send]          │ │ │                                          │ │
│ └───────────────────────────┘ │ │ [Save] [Propose changes] [Ask agent]      │ │
│                               │ └──────────────────────────────────────────┘ │
└───────────────────────────────┴──────────────────────────────────────────────┘
```

Rules:

- Chat is a streaming session; it is not emitted as facts.
- Saving the artifact creates/updates domain-owned records; the projection later reflects progress.
- “Propose changes” triggers a process intent to move into approval/execution (per methodology).

### 5) Coding surface (executor console)

Purpose: show live run output and allow user interventions (append guidance) during delegated coding.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Coding Activity (CARD C)                                                      │
│ Status: running  |  Executor: codex-runner  |  Attempt: 2                    │
├──────────────────────────────────────────────────────────────────────────────┤
│ Live output (stream)                                                         │
│ ┌──────────────────────────────────────────────────────────────────────────┐ │
│ │ [streaming logs / codex events / progress]                                │ │
│ │ …                                                                         │ │
│ └──────────────────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────────────────┤
│ Append guidance (out-of-band message to the run)                              │
│ [ text… ] [Send]   (idempotent; linked to activity/run id)                   │
├──────────────────────────────────────────────────────────────────────────────┤
│ Evidence (projection)                                                        │
│ - PR link                                                                     │
│ - patch                                                                        │
│ - test logs                                                                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

Rules:

- Live output uses a streaming protocol; evidence links are projection-rendered.
- “Send guidance” is not a Kafka fact; it is an activity/run interaction API.

### 6) Release surface (promotion + rollout evidence)

Purpose: show the release gate and promotion evidence (build/publish + GitOps promotion).

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Release (CARD D)                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ Gate: Ready to release?   [Approve] [Reject]                                  │
│                                                                              │
│ Build/Publish Evidence (projection)                                           │
│ - Image digest(s)                                                             │
│ - Build logs                                                                   │
│                                                                              │
│ GitOps Promotion (projection)                                                 │
│ - PR to gitops repo                                                           │
│ - ArgoCD sync status                                                          │
│ - Smoke checks                                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Interaction patterns (recommended)

### Pattern A: Kanban live updates

1. UI subscribes via `KanbanUpdatesService.WatchKanbanUpdates(scope, after_seq)` and receives cursor-only updates `{board_ref, board_seq}` (at-least-once).
2. On cursor advance:
   - preferred: `GetKanbanDeltas(scope, since_seq)`,
   - fallback: `GetKanbanBoard(scope)`.
3. UI never tries to interpret infra retries; it reflects projection truth.

### Pattern B: “in progress” badges

The board indicates “in progress” only when facts imply it:

- a running activity is visible via process progress facts (phase-first) and/or explicit “activity transitioned” facts (capability-specific).

### Pattern C: Human decisions (approvals)

Approvals are modeled as:

- UI shows `Awaiting(kind=approval, ref=...)`.
- UI action calls Process update/command.
- Projection reflects a `GateDecisionRecorded`/phase transition afterwards.

## MVP slice (generic process first)

To keep Scrum/TOGAF for later, the MVP uses:

- a generic R2R process definition with a small set of `phase_key` values (intake/triage/execution/release/terminal),
- initiative-scoped Kanban (`board_kind=initiative`) and change detail pages,
- triage surface (chat + requirement editor),
- coding surface (stream output + append guidance),
- evidence/timeline views (projection-only).

Scrum/TOGAF add only:

- different `phase_key` sets and mapping configs,
- additional “awaiting” gates and artifacts,
without changing the Kanban/UI contracts above.
