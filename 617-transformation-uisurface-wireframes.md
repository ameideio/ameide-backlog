# 617 — Transformation UISurface Wireframes (UI/UX Only)

**Status:** Active (target UX; implementation may lag)  
**Audience:** Product/UI engineers, platform engineers, capability teams  
**Scope:** Transformation routes, pages, interactions, and live-update behavior (no backend schema details)

This backlog defines the **UISurface experience** for Transformation (R2R) as a projection-driven product UI.

## Cross‑references (normative)

- `backlog/520-primitives-stack-v2.md` (primitives + planes)
- `backlog/513-uisurface-primitive-scaffolding.md` (UISurface shell/canvases/widgets)
- `backlog/527-transformation-e2e-sequence.md` (Transformation phases/columns + execution sequence)
- `backlog/614-kanban-projection-architecture.md` (Kanban as a projection)
- `backlog/615-kanban-fullstack-reference.md` (Kanban full‑stack reference)
- `backlog/616-kanban-principles.md` (Kanban principles; interaction streams are out‑of‑band)
- `backlog/496-eda-principles.md` (facts vs intents; outbox; idempotency)
- `backlog/509-proto-naming-conventions.md` (process progress facts identity + minimal vocabulary)

## UX invariants (non‑negotiable)

1. **All reads come from projections** (Kanban/timeline/status/details). The UI never uses Temporal visibility/history as product truth.
2. **Kanban is a view**: columns are derived; “move card” (if offered) emits a real domain/process intent and reconciles to projection truth.
3. **Live updates are cursor‑based**: UI subscribes to a “Projection Updates” stream and refetches deltas/board on cursor advance.
4. **Interaction streams are not facts**: chat/log streaming is out‑of‑band (SSE/WebSocket/server‑streaming RPC), linked by stable IDs.
5. **Activities are the interaction gateway**: clicking a card opens the correct “activity surface” (triage chat+editor, coding console, verification logs, release view).

## Concepts (as the UI sees them)

- **Repository**: the architecture context for work.
- **Initiative**: a change initiative governed in a repository context (future: Scrum/TOGAF profiles).
- **Board**: projection-backed view, scoped by `{tenant, org, repo, initiative?}` and `board_kind`.
- **Card**: a stable unit of work (default in process-driven boards: `card_id = process_instance_id`).
- **Activity**: a user-visible unit of work inside a process instance (triage, coding, verify, release).
- **Evidence**: links to artifacts/logs/PRs produced by activities (projection-rendered; not embedded streaming).

## Navigation and routes (target)

The UISurface is a **shell** with canvases and widgets (`backlog/513-uisurface-primitive-scaffolding.md`).

Recommended routes:

- Repository workspace: `/org/:orgId/repo/:repoId`
  - roll-up board: `board_kind=repository`
  - initiative list (create/open)
- Initiative workspace: `/org/:orgId/repo/:repoId/initiative/:initiativeId`
  - workspace board: `board_kind=initiative`
  - initiative timeline and activity feeds (projection)
- Change workspace (R2R “change” detail): `/org/:orgId/repo/:repoId/r2r/change/:changeId`
  - requirement/deliverables/evidence anchors + run controls
- Optional debug: “timeline” page (projection-only), not Temporal UI: `/org/:orgId/repo/:repoId/timeline`

## Page types and wireframes

### 1) Repository workspace (roll-up)

Purpose: orient the user, show active initiatives, and show a roll-up board without methodology coupling.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Ameide Platform                                                              │
│ Org ▾  Repo ▾                                                                │
├───────────────┬──────────────────────────────────────────────────────────────┤
│ Nav           │ Repository Workspace                                          │
│ - Repos       │ ┌──────────────────────────────────────────────────────────┐ │
│ - Initiatives │ │ Roll-up Board (board_kind=repository)                    │ │
│ - Timeline    │ │ [To Do] [In Progress] [Blocked] [Release] [Done]         │ │
│               │ └──────────────────────────────────────────────────────────┘ │
│               │ Initiatives                                                  │
│               │ ┌──────────────────────────────────────────────────────────┐ │
│               │ │ + New Initiative                                          │ │
│               │ │ - INIT-123: “R2R MVP”   (methodology: generic)            │ │
│               │ │ - INIT-124: “Sales…”    (methodology: scrum)              │ │
│               │ └──────────────────────────────────────────────────────────┘ │
└───────────────┴──────────────────────────────────────────────────────────────┘
```

Primary interactions:

- Create/open an initiative.
- Click a card to open Activity Detail (drawer).

### 2) Initiative workspace (Kanban-first)

Purpose: the main “work cockpit” for a change initiative in a repository.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Initiative INIT-123 (repo: primary-architecture)                              │
│ [Overview] [Kanban] [Timeline] [Evidence]                                     │
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

Card badges (projection-derived):

- “⏳ in progress” means latest process progress facts show a running activity.
- “Awaiting” means `Awaiting(kind, ref)` is active.
- “Blocked” means `Blocked(reason)` is active.
- Terminal states: completed/failed/cancelled.

Primary interactions:

- Click a card → Activity Detail Drawer (below).
- Optional: “Move card” UX is allowed only as a real intent (e.g., approval decision), never as a local status change.

### 3) Activity Detail Drawer (per card)

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

1. UI subscribes to Projection Updates stream for `{board_id, board_seq}`.
2. On cursor advance:
   - preferred: `GetKanbanDeltas(board_id, since_seq)`,
   - fallback: `GetKanbanBoard(board_id)`.
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

