# 620 — Kanban Full‑Stack Implementation & Refactor Plan (Checklist)

This backlog is the **implementation plan** for delivering Ameide’s platform Kanban as the standard “progress view” for **all Temporal-orchestrated processes**, aligned to the **ideal** architecture (docs define the target; code/processes align — not the other way around).

It is intentionally **checklist-driven** and capability-spanning: it covers **proto/SDK**, **Process** (progress facts), **Projection** (Kanban read model + updates stream), **UISurface** (board + activity workbench), and **refactors across existing domains**.

## Cross‑references (normative)

- `backlog/616-kanban-principles.md` (normative Kanban UX principles)
- `backlog/614-kanban-projection-architecture.md` (architecture contract)
- `backlog/618-kanban-proto-contracts.md` (proto contracts)
- `backlog/615-kanban-fullstack-reference.md` (implementation reference)
- `backlog/619-kanban-domain-alignment-refactors.md` (per-domain deltas tracker)
- `backlog/520-primitives-stack-v2.md` (platform constitution)
- `backlog/520-primitives-stack-v2-projection.md` (Projection invariants)
- `backlog/511-process-primitive-scaffolding.md` (Temporal Process contract)
- `backlog/509-proto-naming-conventions.md` (topic families + progress fact identity/vocabulary)
- `backlog/496-eda-principles.md` (facts are not requests; outbox discipline)
- `backlog/430-unified-test-infrastructure.md` (test philosophy: unit + mock integration)

## Target (single “clean” end state)

### Platform invariant

- Kanban is a **product read model** implemented as a **Projection** over the fact log.
- UISurfaces read Kanban only via **Projection query APIs** and a **Projection Updates stream** (cursor-based notifications).
- UISurfaces do **not** read Temporal visibility/history as product truth.
- “Interactive streaming” (chat logs, agent tool streams, coding console output) is **not** transported as facts; it is served via **activity interaction surfaces** (out-of-band).

### Board model (standard)

- Boards are **process-definition-centric** (`process_definition_id` is part of scope identity).
- Cards represent **process instances**:
  - `card_id` defaults to `process_instance_id` (Temporal WorkflowID) and MUST NOT be reused.
  - `process_run_id` is “current run id (best effort)” for drill-in only; it may change (Continue-As-New).
- Columns are **derived** from facts using mapping rules; UI does not implement methodology logic.
- Live updates use `board_seq` (cursor derived from durable projection commit sequence) + idempotent refetch/deltas.
- Paging and deltas are first-class (boards must not assume “small lists”).

## Definition of Done (DoD)

- [ ] `ameide_core_proto.platform.kanban.v1` protos exist and SDKs are generated (`backlog/618-kanban-proto-contracts.md`).
- [ ] Each relevant Projection primitive implements the **standard** `KanbanQueryService` and `KanbanUpdatesService` (same RPC surface everywhere).
- [ ] Each relevant Process primitive emits **process progress facts** aligned to `backlog/509-proto-naming-conventions.md` (phase-first by default).
- [ ] Board updates stream is cursor-based and advances only on **effectful commits** (no-op/dedupe does not advance).
- [ ] UISurface renders boards from Projection APIs, subscribes to updates, and provides **activity workbench surfaces** (per `backlog/616-kanban-principles.md`).
- [ ] Unit tests + mock integration tests cover:
  - [ ] process progress emission idempotency
  - [ ] projection idempotent ingestion + deterministic apply order
  - [ ] board paging + deltas semantics
  - [ ] updates stream semantics + UI refetch loop
  - [ ] archival/boundedness behavior
- [ ] `backlog/619-kanban-domain-alignment-refactors.md` is fully checked off (all tracked domains refactored).

## Implementation plan (work packages)

### WP0 — Repo-wide “contract lock” (guardrails)

- [ ] Add a CI/doc gate that rejects:
  - [ ] UISurface code reading Temporal visibility/history for product Kanban truth
  - [ ] bespoke kanban proto/service shapes that diverge from `ameide_core_proto.platform.kanban.v1`
- [ ] Add a proto/API gate that rejects Kanban services without paging/deltas in request/response shapes.
- [ ] Add a compatibility note: this plan assumes **no backward compatibility**; remove/replace bespoke APIs.

### WP1 — Proto + SDK adoption (platform-wide)

Goal: the **same** Kanban query + updates interface is available to every capability via generated SDKs.

- [ ] Implement `packages/ameide_core_proto/src/ameide_core_proto/platform/kanban/v1/kanban.proto` as described in `backlog/618-kanban-proto-contracts.md`.
- [ ] Implement `KanbanQueryService` and `KanbanUpdatesService` protos in `ameide_core_proto.platform.kanban.v1`.
- [ ] Regenerate SDKs (Go/TS/Python) and enforce “SDK-only imports” in runtime code (`backlog/520-primitives-stack-v2.md`).
- [ ] Define and publish canonical `process_definition_id` strings for existing processes (multi-domain):
  - [ ] `transformation.r2r.v1`
  - [ ] `sales.funnel.v1` (or the approved sales process key)
  - [ ] `sre.incident.v1` (and other SRE process keys as needed)
  - [ ] `commerce.<process>.v1` (as applicable)

### WP2 — Projection: canonical Kanban read model (per capability)

Goal: each capability’s Projection can serve Kanban for its process-definition boards using the standard API.

- [ ] Implement a canonical Kanban storage schema in Postgres for operational boards (per `backlog/614-kanban-projection-architecture.md`):
  - [ ] `kanban_board_state` (includes durable commit cursor/sequence → `board_seq`)
  - [ ] `kanban_card_view` (includes per-card `seq`, `column_key`, `rank` optional, derived `is_archived`)
  - [ ] `kanban_projection_inbox` (dedupe by `message_id`)
- [ ] Implement deterministic apply rules for multi-stream ingestion (one explicit rule set; no “timestamp ordering”).
- [ ] Implement mapping configuration (phase → column) as projection-owned data (not UI logic):
  - [ ] versioned mapping rules per `(board_kind, process_definition_id)`
  - [ ] default mappings for Transformation/Sales/SRE examples
- [ ] Implement `GetKanbanBoard` with:
  - [ ] paging (`PageRequest/PageResponse`)
  - [ ] optional `column_key` scoping for per-column paging
- [ ] Implement `GetKanbanDeltas` with:
  - [ ] `since_seq` semantics (delta is scoped by `board_seq`)
  - [ ] `upserted_cards` plus `archived_card_ids` / `deleted_card_ids`
- [ ] Implement archival/boundedness:
  - [ ] derived archive rule (terminal phases or explicit archive facts)
  - [ ] default queries exclude archived cards

### WP3 — Projection updates stream (live refresh)

Goal: UI can subscribe to “board changed” notifications and refetch safely.

- [ ] Implement `WatchKanbanUpdates` (server streaming) in each Projection implementing Kanban:
  - [ ] output payload includes `{board_ref, board_seq}`
  - [ ] at-least-once delivery is tolerated by UI (idempotent refetch)
- [ ] Prefer a non-polling mechanism (Postgres `LISTEN/NOTIFY` or equivalent) for updates stream fanout.
- [ ] Provide a safe fallback for dev/test (polling allowed only in tests/dev mode).

### WP4 — Process: standardized progress facts (per capability)

Goal: every process emits **phase-first** progress facts in a way that Kanban can consume across domains.

- [ ] Adopt the minimal progress vocabulary from `backlog/509-proto-naming-conventions.md` across processes:
  - [ ] `RunStarted`
  - [ ] `PhaseEntered(phase_key)`
  - [ ] `Awaiting(kind, ref)`
  - [ ] `Blocked(reason)` (optional)
  - [ ] one terminal: `RunCompleted` | `RunFailed` | `RunCancelled`
  - [ ] optional opt-in: `StepCompleted/StepFailed` (with `step_id` + `step_instance_id`)
- [ ] Enforce idempotency:
  - [ ] process progress facts emitted via Activities/ports (at-least-once safe)
  - [ ] `message_id` stable under retries and across Continue-As-New boundaries
- [ ] Ensure workflows use correct Temporal identity naming (WorkflowID vs RunID):
  - [ ] `process_instance_id = WorkflowID`
  - [ ] `process_run_id = RunID`
  - [ ] `run_epoch + seq` for ordering (phase-first, avoid step-spam)
- [ ] Add “phase budget” rule to prevent history explosion:
  - [ ] phase transitions are the default; step-level evidence is opt-in per process/step type

### WP5 — Refactor existing domains/processes to the ideal Kanban model

This is the “make reality match the docs” tranche. Track completion in `backlog/619-kanban-domain-alignment-refactors.md`.

#### Transformation (R2R)

- [ ] Replace Transformation-specific Kanban APIs with the platform Kanban interface (`ameide_core_proto.platform.kanban.v1`).
- [ ] Ensure Transformation process emits phase-first progress facts for:
  - [ ] ideation/triage
  - [ ] stabilization
  - [ ] build (coding)
  - [ ] verify
  - [ ] release
- [ ] Ensure initiative-scoped boards are supported (`board_kind=initiative`, `initiative_id`).
- [ ] Projection builds Kanban card views from process progress facts + initiative facts:
  - [ ] `card_id = process_instance_id`
  - [ ] `board_id` derived from `(tenant, org, repo, board_kind, process_definition_id, initiative_id?)`

#### Sales (funnel/pipeline)

- [ ] Introduce a canonical Sales process-definition (`process_definition_id="sales.funnel.v1"`) and emit phase-first progress facts (e.g., prospecting → qualification → proposal → negotiation → closed).
- [ ] Refactor Sales process facts and subjects to carry the board scope axes required by `KanbanBoardScope` (at minimum: tenant/org/repository_id-like context id) so boards can be authorized/routed consistently.
- [ ] Implement Sales projection Kanban interface (board + deltas + updates) using the standard protos.

#### SRE (incident/runbook)

- [ ] Introduce canonical SRE process definitions (`sre.incident.v1`, `sre.runbook.v1`, etc.) and emit phase-first progress facts.
- [ ] Ensure SRE scope axes are represented consistently for Kanban routing and auth (tenant/org/repository_id-like context id, or a standardized mapping).
- [ ] Implement SRE projection Kanban interface using standard protos.

#### Commerce (example process boards)

- [ ] Identify which Commerce processes are Kanban-worthy (e.g., domain onboarding pipeline) and assign canonical `process_definition_id`.
- [ ] Emit phase-first progress facts and implement standard Kanban projection surface.

### WP6 — UISurface integration (platform app)

Goal: one consistent Kanban UX across domains, and boards as gateways to the right interaction surface.

- [ ] Render a Kanban board page/view from `KanbanQueryService` (board + paging).
- [ ] Subscribe to `KanbanUpdatesService` and refetch deltas (or fall back to full refetch).
- [ ] Implement “activity workbench” launch from a card (per `backlog/616-kanban-principles.md`):
  - [ ] ideation/triage: chat + artifact editor surface
  - [ ] build/coding: coding console stream + intervention controls
  - [ ] verify/release: logs + evidence + gate controls
- [ ] Support nested boards (“subprocess trees”) via `parent_card_id` and/or `initiative_id`:
  - [ ] board navigation to inner boards
  - [ ] breadcrumb and back navigation
- [ ] Keep interactions out of the fact system:
  - [ ] board shows derived progress + links
  - [ ] workbench streams come from dedicated APIs/transports

### WP7 — Testing (unit + mock integration, per 430)

Cluster is not assumed available for this backlog. “Done” is enforced by **unit tests and mock integration tests**.

- [ ] Process tests:
  - [ ] progress fact emission is stable under Activity retries (no duplicate user-visible progress)
  - [ ] Continue-As-New dedupe strategy is correct (where used)
- [ ] Projection tests:
  - [ ] idempotent inbox dedupe by `message_id`
  - [ ] deterministic apply with multi-stream ordering rule
  - [ ] `board_seq` advances only on effectful commits
  - [ ] paging + deltas correctness
  - [ ] archival semantics and default filtering
- [ ] UISurface tests:
  - [ ] board renders from mocked SDK client
  - [ ] updates stream triggers delta refetch loop
  - [ ] activity workbench launch routing per activity type

## Notes (keep the spec enforceable)

- “Standard interface everywhere” means:
  - shared messages + shared service interface (`ameide_core_proto.platform.kanban.v1`)
  - implemented by many Projection deployables (no single global bottleneck required)
- Do not retrofit the Kanban contract to match existing bespoke implementations:
  - refactor Sales/Transformation/SRE/Commerce to the contract
  - remove bespoke Kanban APIs once the standard interface is in place

