# 619 — Kanban Domain Alignment Refactors (Sales / Transformation / SRE)

This backlog tracks the required refactorings across capabilities to align existing implementations to the **ideal Kanban architecture**:

> **Facts → Projection → UISurface** (projection-first, process-definition-centric Kanban)

## Cross-references (normative)

- `backlog/616-kanban-principles.md` (Kanban principles; normative)
- `backlog/614-kanban-projection-architecture.md` (architecture contract)
- `backlog/618-kanban-proto-contracts.md` (proto-first query + updates stream)
- `backlog/615-kanban-fullstack-reference.md` (implementation reference)
- `backlog/509-proto-naming-conventions.md` (process progress facts identity + minimal vocabulary)
- `backlog/511-process-primitive-scaffolding.md` (Temporal-backed Process contract)
- `backlog/520-primitives-stack-v2.md` (platform constitution)
- `backlog/496-eda-principles.md` (facts are not requests; outbox + idempotency)

## Decision (single option)

All capabilities MUST align to the platform Kanban contract:

- Kanban boards are **process-definition-centric** (`process_definition_id`), across many process instances.
- Kanban columns are derived from **process progress facts** (phase-first vocabulary).
- The Kanban UISurface reads only projection query APIs and subscribes to the projection updates stream (`board_seq`).
- Temporal visibility/Search Attributes are ops/debug only; not the product Kanban source of truth.

## Workstreams

### A) Platform kanban proto + service adoption

- [ ] Add `ameide_core_proto.platform.kanban.v1` protos to `ameide_core_proto` and generate SDKs.
- [ ] Implement the platform `KanbanQueryService` and `KanbanUpdatesService` in projection primitives.
- [ ] Remove (or deprecate) bespoke capability-specific Kanban APIs once the standard interface is adopted everywhere, to avoid dual contracts.
  - Capabilities may still run their own Projection deployments; the goal is to eliminate bespoke service shapes, not to centralize traffic.

### B) Process progress facts alignment (phase-first)

All Temporal-backed Processes must emit the minimal progress vocabulary defined in `backlog/509-proto-naming-conventions.md`:

- `RunStarted`
- `PhaseEntered(phase_key)`
- optional `Awaiting(kind, ref)` / `Blocked(reason)`
- terminal `RunCompleted|RunFailed|RunCancelled`

All emitted facts MUST include:

- `process_definition_id` (and optional version id)
- `process_instance_id` (WorkflowID; stable across Continue-As-New)
- `process_run_id` (RunID)
- `run_epoch + seq` ordering
- stable `message_id` suitable for inbox dedupe (stable under Activity retries)

### C) Projection alignment (board_id, board_seq, paging, deltas)

- [ ] Compute `board_id` as a deterministic function of `KanbanBoardScope` (including `board_kind` and `process_definition_id`).
- [ ] `board_seq` MUST be derived from a durable projection commit cursor/sequence and MUST advance only on effectful commits (not on inbox dedupe/no-ops).
- [ ] Implement `GetKanbanDeltas` (preferred) and `GetKanbanBoard` (paged). UI must be able to fall back to full refetch.
- [ ] Implement archival semantics (`archived_card_ids`) and default query exclusion of archived cards.

## Capability-specific refactors

### Sales

Goal: a platform-standard Kanban board for the Sales funnel process.

- [ ] Define the Sales funnel as a Process definition (`process_definition_id="sales.funnel.v1"`), and standardize phase keys.
- [ ] Refactor Sales Process primitive to emit phase-first progress facts (not just coordination-only process facts).
- [ ] Refactor Sales Projection to implement platform Kanban APIs for:
  - `board_kind=pipeline` (tenant/org scoped)
  - optional repo/workspace scoping when Sales runs are bound to a repository context

### Transformation

Goal: replace transformation-specific Kanban APIs with platform Kanban APIs.

- [ ] Ensure Transformation Process continues to emit phase-first progress facts with the canonical identity fields.
- [ ] Refactor Transformation Projection to implement platform Kanban APIs:
  - repo process boards (`board_kind=repository`)
  - initiative process boards (`board_kind=initiative`)
- [ ] Remove/replace `TransformationProcessQueryService.GetKanbanBoard/WatchKanbanUpdates` once the platform Kanban API is adopted.

### SRE

Goal: a platform-standard Kanban board for incidents and other SRE processes.

- [ ] Define an incident Process definition (`process_definition_id="sre.incident.v1"`) and phase keys.
- [ ] Refactor SRE Process primitive to emit phase-first progress facts with canonical identity fields.
- [ ] Implement SRE Projection platform Kanban APIs for incident boards (scoped by the chosen `board_kind`).

## Definition of done

- The platform Kanban proto contracts exist and are generated into Go/TS/Python SDKs.
- Sales, Transformation, and SRE projections serve Kanban boards via the platform `KanbanQueryService` and `KanbanUpdatesService`.
- Kanban boards render consistently across domains using `process_definition_id` + `phase_key` with no UI hard-coded mappings.
- `board_seq` semantics are consistent (durable commit cursor; effectful commit only).
- Projections are rebuildable and converge under at-least-once delivery (inbox dedupe by `message_id`).
