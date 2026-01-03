# 615 — Kanban Full‑Stack Reference (Facts → Projection → UISurface)

This backlog is the **full-stack reference** for implementing Kanban in Ameide, end-to-end, using the platform’s standard architecture:

> **Facts → Projection → UISurface** (see `backlog/614-kanban-projection-architecture.md`).

This document is intentionally capability-agnostic, but uses **Transformation** as the concrete reference slice where helpful.

## Cross‑references (normative)

- `backlog/614-kanban-projection-architecture.md` (Kanban architecture contract)
- `backlog/520-primitives-stack-v2.md` (platform constitution)
- `backlog/520-primitives-stack-v2-projection.md` (Projection contract)
- `backlog/511-process-primitive-scaffolding.md` (Process + Temporal contracts)
- `backlog/513-uisurface-primitive-scaffolding.md` (UISurface layout patterns)
- `backlog/509-proto-naming-conventions.md` (process progress facts + identity)
- `backlog/496-eda-principles.md` (facts vs intents; outbox discipline)

## Goal

Provide a single, repeatable implementation recipe for:

1. emitting **process progress facts** and relevant **domain facts**
2. building a **Kanban projection** (idempotent + replayable)
3. serving the Kanban via a **Projection query API**
4. delivering **live updates** via a **Projection Updates stream**
5. rendering Kanban as a **UISurface canvas/page type** with reusable components/widgets

## Non‑goals

- Kanban is not sourced from Temporal visibility/search attributes.
- Kanban does not require step-level events for every BPMN node (phase-first default).
- Kanban does not introduce UI-owned imperative “status”; columns are derived by projection rules.

## Full‑stack contract (what every implementation must provide)

### A) Facts (inputs to the projection)

**Required inputs:**

- **Process progress facts** (coordination truth; phase-first)
- **Domain facts** (business truth; optional but typical for card titles/priority/assignee)

**Rules:**

- Facts are emitted **after persistence** via outbox when the SoR is a domain (`backlog/496-eda-principles.md`, `backlog/510-domain-primitive-scaffolding.md`).
- Process progress facts must follow identity requirements in `backlog/509-proto-naming-conventions.md` (WorkflowID/RunID, `run_epoch + seq`).
- Processes may define their own internal `phase_key` values; **UI must not hard-code mappings**. Mapping to Kanban columns lives in the projection rule/config.
- Every fact MUST carry a stable `message_id` suitable for inbox dedupe. For process-emitted progress facts, `message_id` MUST remain stable under Activity retries.

### B) Projection (build + store)

**Required behaviors:**

- Inbox dedupe on `message_id`.
- Deterministic apply: same fact log → same board.
- Rebuild support (replay from earliest cursor).

**Required outputs:**

- `board_seq` (monotonic cursor derived from the projection’s durable commit cursor/sequence; not timestamps and not “diff heuristics”)
- a query store that supports listing cards by column and fetching card details

**Required invariants:**

- Projection MUST define a deterministic apply order/convergence rule across all consumed topics/partitions that can affect a board (recommended: board-scoped partitioning so log order is the apply order).

### C) Query API

The Projection exposes read-only APIs:

- `GetKanbanBoard(board_scope)` → board model (columns + cards) + `board_seq`
- recommended: `GetKanbanDeltas(board_scope, since_seq)` → incremental updates (cursor-based)

### D) Projection Updates stream (live updates)

The Projection exposes a subscription that emits at least:

- `{ board_id/scope, board_seq }` (at-least-once)

**UI behavior is idempotent:**

- UI keeps `last_seen_board_seq`
- on any update where `board_seq` increases → fetch deltas (preferred) or refetch board (fallback)

### E) UISurface layout (platform app)

Kanban is a **canvas/page type** and a **reusable component**:

- **Canvas/page**: routing, auth, context (tenant/org/repo), subscription lifecycle
- **Component/widget**: board renderer used by the page and embeddable in dashboards/detail views

This matches `backlog/520-primitives-stack-v2.md` “Shell + Canvases + Widgets” and `backlog/513-uisurface-primitive-scaffolding.md` scaffold conventions.

## Recommended transport choice (single option)

**Projection query API = gRPC/Connect** (SDK-only rule).

**Projection Updates stream = server-streaming RPC to a server-side gateway, delivered to the browser as SSE.**

Rationale:

- Keeps browser free of infra/proto transport details.
- Works well with Next.js route handlers and auth middleware.
- Keeps the “update stream” thin (cursor only) and makes refetch behavior explicit.

## Reference implementation: Transformation (where each piece lives)

### 1) Process emits progress facts

- Temporal workflows emit phase-first progress facts via an Activity/port (idempotent).
- Facts include `process_instance_id` (WorkflowID), `process_run_id` (RunID), `run_epoch`, `seq`, `process_definition_id`.

### 2) Projection ingests facts and builds the board

- Consumer applies deterministic mapping config:
  - `(process_definition_id, phase_key, ...) -> column_key`
- Projection stores:
  - cards (one row per `card_id`)
  - board cursor (`board_seq`)
  - inbox dedupe (`message_id`)

### 3) Projection serves `GetKanbanBoard`

- Query returns:
  - `board_seq`
  - ordered columns (stable column order driven by config)
  - cards grouped by column

### 4) Projection serves updates stream

- `WatchBoardUpdates(after_seq)` emits new `{board_seq}` values.

### 5) UISurface renders Kanban

- Page/canvas:
  - fetches board
  - subscribes to updates
  - re-fetches when seq increases
- Reusable component:
  - renders columns/cards without embedding methodology rules

## Testing requirements (definition of done)

### Projection tests

- Dedupe: re-deliver same `message_id` → no duplicate card updates.
- Convergence: replay from empty → same board state.
- Cursor: board_seq advances based on the durable projection commit cursor/sequence for the board; retries do not create duplicate effects.

### Process tests

- Emission is idempotent under activity retry semantics (explicit idempotency keys).
- Ordering: facts ordered by `(run_epoch, seq)` not timestamps.

### UISurface tests

- Page renders from projection query API only.
- Live updates: when update arrives, UI refetches and board changes.

## Implementation checklist (copy/paste)

- [ ] Define card identity (`card_id`) and board scope (`tenant/org/repo/capability`).
- [ ] Define mapping config from phase keys to column keys (versioned).
- [ ] Emit process progress facts (phase-first) with correct identity fields.
- [ ] Build projection with inbox dedupe + replayable apply.
- [ ] Implement `GetKanbanBoard` (+ optional deltas).
- [ ] Implement update stream emitting monotonic `board_seq`.
- [ ] Implement Kanban as UISurface canvas + reusable component/widget.
- [ ] Add unit/integration/E2E tests covering refetch-on-seq and convergence.
