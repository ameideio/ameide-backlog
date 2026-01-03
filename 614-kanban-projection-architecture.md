# 614 — Kanban As A Projection (Facts → Projection → UISurface)

This backlog defines the **recommended** long-term architecture for Kanban views across Ameide capabilities.

It is intentionally aligned with:

- `backlog/520-primitives-stack-v2.md` (Primitives Stack v2 constitution)
- `backlog/520-primitives-stack-v2-projection.md` (Projection primitive contract)
- `backlog/511-process-primitive-scaffolding.md` (Temporal-backed Process primitive contract)
- `backlog/616-kanban-principles.md` (Kanban principles; projection-first UX)
- `backlog/509-proto-naming-conventions.md` (topic families + process progress fact conventions)
- `backlog/496-eda-principles.md` (facts are not requests; outbox discipline)
- `backlog/513-uisurface-primitive-scaffolding.md` (UISurface reads from projections; no infra coupling)

## Decision (recommended; single option)

**Kanban is a product read model, implemented as a Projection.**

- The UISurface renders Kanban **only** from Projection query APIs.
- Projections are built by consuming **domain facts** and **process progress facts** from the fact log (Kafka).
- Live Kanban updates are delivered via a **Projection Updates** stream (SSE/WebSocket or server-streaming RPC) that notifies the UI of a monotonic update cursor.

Temporal’s visibility/search APIs remain valuable for **ops/debug** and optional reconciliation tooling, but they are **not** the product Kanban source of truth.

## Why (avoid coupling Kanban to Temporal visibility)

Temporal visibility/search is designed for operational listing/filtering of workflow executions. It is not a durable, rebuildable product store:

- Visibility/listing is **eventually consistent** and may lag under load.
- Retention policies remove workflow data unless archival is configured.
- Search attribute limits and indexing constraints make “rich card state” impractical.

The Ameide platform requires projections to be **rebuildable, convergent, and correct under at-least-once delivery** (`backlog/520-primitives-stack-v2-projection.md`, `backlog/496-eda-principles.md`). Therefore Kanban must be backed by a projection store, not by Temporal visibility.

## Kanban contract (capability-agnostic)

### 1) What a “card” is

A **card** represents a stable business unit of work. In process-driven boards, the default is:

- `card_id = process_instance_id` (Temporal WorkflowID; stable for the workflow chain)

For drill-in, the card may also expose:

- `process_run_id` (Temporal RunID; changes on Continue-As-New)

**Workflow IDs used as card IDs MUST NOT be reused for new instances.** If a capability cannot enforce non-reuse, use a domain-owned stable entity ID as `card_id` instead.

### 1a) Board identity and membership (required)

Kanban is a projection-backed list view. The projection MUST define a stable board identity and a deterministic membership rule.

- `board_id` MUST be a deterministic function of board scope (recommended default: `{tenant_id, organization_id, repository_id, board_kind}`).
- Cards MUST be stored and queried by `(board_id, card_id)` (card membership is derived; no UI-owned “board membership” state).
- A capability MAY support multiple boards per scope (e.g., multiple `board_kind` values), but board IDs MUST remain stable over time.

Recommended `board_kind` values (capability-agnostic):

- `board_kind=repository`: repo-scoped roll-up board for an architecture context.
- `board_kind=initiative`: initiative workspace board for a change initiative bound to a repository.
  - Initiative boards MUST still be repository-scoped (initiative identity is not meaningful without its repository context).
  - For initiative boards, scope MUST include `initiative_id` in addition to `{tenant_id, organization_id, repository_id}` so `board_id` remains unique and stable.

### 2) Columns are derived (never imperative UI state)

The UI MUST NOT contain methodology-specific mapping logic.

Columns are derived by projection rules from facts, using a configurable mapping such as:

- `(process_definition_id, phase_key, ...) -> column_key`

The mapping must be versioned/configurable so each methodology/process can define its own phases while still producing a generic Kanban surface.

### 3) Ordering (separate “correctness cursor” vs optional user ranking)

Kanban needs two different “sequence” concepts:

- **Correctness cursor** (required): monotonic `board_seq` (or per-card `seq`) used for idempotent incremental updates and cache safety.
- **User ranking** (optional): `rank` key for drag/drop ordering (e.g., LexoRank-style). Do not overload `seq` for ranking.

If ranking is not needed, the board may sort by `updated_at` / domain priority / SLA.

### 4) Minimal view model

Each card returned by the projection MUST include:

- `card_id`
- `column_key`
- `title` (or `summary`)
- `updated_at` (display only)
- `seq` (monotonic per-card change cursor) or `board_seq` (board-level cursor)

Recommended:

- `assignee`, `priority`
- `blocked_reason`
- `links[]` (evidence refs, PR links, work request IDs)

### 5) Archival and deletion semantics (required)

Operational boards must remain bounded without relying on hard deletes.

- Cards MUST have a projection-derived terminal/archival rule (e.g., `is_archived` or `terminal_state` derived from facts).
- Board queries MUST default to excluding archived cards (unless explicitly requested).
- Hard deletion of card rows is NOT required for correctness; retention/cleanup is an operational policy, not part of the product contract.

## Fact sources (what the projection consumes)

### Domain facts (business truth)

Domains emit **domain facts** via transactional outbox after persistence (`backlog/510-domain-primitive-scaffolding.md`, `backlog/496-eda-principles.md`).

These facts can drive card attributes such as:

- title/summary
- priority
- assignee
- business-state transitions (which may be reflected in Kanban labels or terminal status)

### Process progress facts (coordination truth)

Processes emit **process progress facts** at phase/milestone transitions (phase-first by default).

- Workflow code remains deterministic.
- Emission happens via Activities/ports (at-least-once; idempotency required) per `backlog/511-process-primitive-scaffolding.md` and `backlog/520-primitives-stack-v2.md`.
- Identity fields must follow `backlog/520-primitives-stack-v2.md` / `backlog/509-proto-naming-conventions.md`:
  - `process_instance_id` (WorkflowID)
  - `process_run_id` (RunID)
  - `run_epoch + seq` for ordering (not timestamps)
  - `step_id` + `step_instance_id` when step-level facts are enabled

The canonical minimal progress vocabulary and identity rules live in `backlog/509-proto-naming-conventions.md`.

## Projection implementation requirements

### Deterministic apply order across fact sources (required)

If a board is built from multiple fact streams/topics, the projection MUST define a deterministic apply order and convergence rule.

Recommended default:

- Partitioning/keying ensures all facts that affect a board are ordered by a single durable log order (e.g., board-scoped partitioning).

If multiple partitions are used:

- Projection logic MUST converge under at-least-once delivery and cross-partition interleaving (commutative/last-write-wins rules per field), and MUST NOT rely on timestamps for correctness ordering.

### Idempotency + replay

Projections MUST be correct under at-least-once delivery:

- Use a projection inbox/dedupe table keyed by `message_id`.
- Track a durable consumer cursor/checkpoint.
- Rebuild/replay from the fact log must converge to the same board state.

See `backlog/520-primitives-stack-v2-projection.md`.

### Message identity (required)

Every ingested fact MUST carry a stable `message_id` suitable for inbox dedupe. For process-emitted progress facts, `message_id` MUST remain stable under Activity retries (no “new UUID per retry”).

### Storage selection (operational vs analytical)

- **Postgres** is the default for operational Kanban (current board state, low-latency queries, transactional updates).
- **ClickHouse** (or similar) is optional for long-retention analytics and heavy scans/aggregates.

This separation avoids mixing operational UI reads with analytical workloads and stays consistent with `backlog/520-primitives-stack-v2-projection.md`.

## Live updates (“Projection Updates” stream)

The UI should not poll full board state. Instead:

1. Projection maintains a monotonic `board_seq` that is derived from the projection’s durable commit cursor/sequence (not from timestamps and not from “diff heuristics”).
2. Projection exposes a subscription API that emits `{board_id, board_seq}` (at-least-once).
3. UISurface subscribes; on notification it performs an idempotent refetch:
   - preferred: fetch deltas “since seq” (cursor-based),
   - fallback: refetch the board when deltas are unavailable or too large.

Allowed transports:

- Server-streaming RPC from the Projection service
- SSE/WebSocket via a gateway (UISurface or Edge service)

The key constraint is **monotonic cursor + idempotent refetch**, not a specific transport.

## Hierarchies (subprocesses) without coupling Kanban to BPMN diagrams

Capabilities MAY represent multi-level processes (subprocess trees) by emitting/process facts that include:

- `parent_process_instance_id` (optional)
- `root_process_instance_id` (optional)
- `parent_step_instance_id` (optional)

The projection can then support:

- expandable cards (parent card shows child progress summary)
- swimlanes per root instance
- drill-in to a “sub-board” view

The Kanban surface remains a projection-backed list view; it does not attempt to render BPMN diagrams.

## Non-goals

- Kanban is not implemented by querying Temporal visibility as the authoritative store.
- Kanban is not implemented by UIs “deciding” phase/column mappings.
- Kanban does not require emitting step-level events for every workflow node (phase-first is the default to respect Temporal history constraints).

## Implementation checklist (for capability teams)

- Define `board_id` scope (tenant/org/repo/capability) and `card_id` identity.
- Define a mapping config from `(process_definition_id, phase_key, ...)` to `column_key` (versioned).
- Ensure Process workflows emit phase-first progress facts via Activities (idempotent).
- Ensure Domain emits required domain facts via outbox (idempotent consumers).
- Implement projection:
  - inbox dedupe on `message_id`
  - deterministic mapping + upsert
  - `GetKanbanBoard` query API
  - `WatchBoardUpdates` (or equivalent) update stream emitting `board_seq`
- UISurface:
  - reads from Projection only
  - subscribes to Projection Updates stream
  - refetches idempotently and renders without methodology-specific logic
