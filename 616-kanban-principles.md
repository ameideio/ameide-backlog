# 616 — Kanban Principles (Projection‑First, Activity‑Centric UX)

**Status:** Active (normative)  
**Audience:** Platform engineers, capability teams, UI engineers, agents  
**Scope:** Product Kanban behavior and contracts (not vendor UI, not infra dashboards)

This backlog defines the **platform-wide principles** for Kanban views in Ameide.

It is intentionally aligned with:

- `backlog/520-primitives-stack-v2.md` (platform constitution; primitives + planes)
- `backlog/520-primitives-stack-v2-projection.md` (projection contract)
- `backlog/511-process-primitive-scaffolding.md` (Temporal-backed Process contracts)
- `backlog/509-proto-naming-conventions.md` (process progress facts identity + minimal vocabulary)
- `backlog/496-eda-principles.md` (facts vs intents; outbox + idempotency)
- `backlog/614-kanban-projection-architecture.md` (Kanban as a projection; recommended architecture)
- `backlog/615-kanban-fullstack-reference.md` (full-stack reference implementation)
- `backlog/513-uisurface-primitive-scaffolding.md` (UISurface “Shell + Canvases + Widgets”)

## Decision (single option)

**Kanban is a product read model implemented as a Projection and rendered by a UISurface.**

Kanban is **not** implemented by querying Temporal history/visibility as the source of truth.

## Definitions

- **Board**: a scoped Kanban view (columns + cards) served by a Projection query API.
- **Card**: a stable unit of work derived from facts; displayed in one column at a time.
- **`board_kind`**: a stable discriminator that selects *which* board model is being rendered for a given scope. It enables multiple boards over the same `{tenant, org, repo}` context without inventing new UI logic.
  - Examples: `repository`, `initiative`, `capability`, `workflow_type`.
- **Projection Updates stream**: an at-least-once notification stream that carries a monotonic cursor (e.g., `board_seq`) so the UI can refetch safely.

## Principles (normative)

### Principle 1: UI reads come from Projections only

- UISurfaces MUST render Kanban from Projection query APIs only.
- UISurfaces MUST NOT depend on Temporal visibility/search APIs or Temporal history as product truth.
- Temporal UI/visibility MAY be linked as an ops/debug surface, but MUST NOT be a required dependency for product Kanban correctness.

### Principle 2: Facts drive the board; UI does not own status

- Columns and card state MUST be derived deterministically from **facts** plus versioned/configurable mapping rules.
- The UI MUST NOT maintain an imperative “card status” as the source of truth.
- If the UI offers a “move card” interaction, it MUST issue a real domain/process **intent/command** and then reconcile to the projection truth (per `backlog/496-eda-principles.md`).

### Principle 3: Phase-first progress is the default

- Process-backed boards MUST be **phase-first** by default to stay within Temporal constraints (history growth, retries).
- The minimal progress vocabulary defined in `backlog/509-proto-naming-conventions.md` SHOULD be sufficient to render a useful board/timeline without workflow-specific UI logic.
- Step-level progress is opt-in and MUST include `step_id` + `step_instance_id` when enabled.

### Principle 4: Board identity and membership are deterministic

- A Projection MUST define stable `board_id` and deterministic membership rules.
- `board_id` MUST be a deterministic function of the board scope and `board_kind`.

Recommended default scopes:

- **Repository roll-up board:** `{tenant_id, organization_id, repository_id, board_kind=repository}`
- **Initiative workspace board:** `{tenant_id, organization_id, repository_id, initiative_id, board_kind=initiative}`

Initiatives are change initiatives and MUST be interpreted in a repository (architecture context). Kanban is a view of activities executed in that initiative+repository context.

### Principle 5: “Live updates” are cursor-based and idempotent

- Projections MUST expose a monotonic cursor for updates (recommended: `board_seq`), derived from the projection’s durable commit cursor/sequence (not timestamps and not “diff heuristics”).
- UISurfaces MUST treat the updates stream as at-least-once and implement idempotent refetch:
  - preferred: deltas `since_seq`,
  - fallback: refetch the board.

### Principle 6: Ordering is separated from correctness

- `board_seq` / `seq` MUST be used for correctness, caching, and incremental fetch safety.
- User-defined ordering (drag/drop) MUST use a separate ranking key (e.g., `rank`), not `board_seq`.

### Principle 7: Projections are rebuildable and convergent

- Projections MUST be correct under at-least-once delivery.
- Projections MUST implement inbox dedupe keyed by `message_id`.
- Replay/rebuild from the fact log MUST converge to the same board state.

### Principle 8: Progress facts are for visibility; interaction streams are out-of-band

- Progress facts exist to make the board/timeline **rebuildable** and **auditable**.
- Human/agent interaction (chat, logs, tool streaming) MUST NOT be transported as facts.
- Interactive sessions MUST use dedicated streaming transports (SSE/WebSocket/server-streaming RPC) and be linked to cards/activities via stable IDs.

### Principle 9: “Activities” are the UX gateway

- Clicking a card MUST open an “activity surface” appropriate to the underlying runtime boundary:
  - triage/stabilization: chat + artifact editor,
  - coding: executor console/log stream + PR/evidence links,
  - verification: test logs + artifacts,
  - release: promotion status + evidence.
- The board MUST remain projection-driven; the activity surface may stream interaction, but the board state changes only when facts arrive and the projection updates.

### Principle 10: Methodologies define phases; the platform defines the contract

- Processes MAY define their own `phase_key` values (Scrum, TOGAF, generic BPMN).
- Kanban MUST remain methodology-agnostic:
  - the projection owns the mapping `(process_definition_id, phase_key, ...) -> column_key`,
  - the UI MUST NOT hard-code methodology-specific mappings.

## Non-goals

- Using Temporal visibility/search attributes as the Kanban query store.
- Emitting chat transcripts as broker facts.
- Requiring step-level events for every workflow node.

