# 618 — Kanban Proto Contracts (Projection Query + Updates)

This backlog defines the **proto-first contracts** required to implement Kanban across Ameide using the platform standard:

> **Facts → Projection → UISurface** (`backlog/614-kanban-projection-architecture.md`)

It is intentionally aligned with:

- `backlog/520-primitives-stack-v2.md` (platform constitution; SDK-only runtime imports)
- `backlog/520-primitives-stack-v2-projection.md` (Projection contract: idempotency + replay)
- `backlog/616-kanban-principles.md` (Kanban UX principles; interaction streams are out-of-band)
- `backlog/615-kanban-fullstack-reference.md` (full-stack reference)
- `backlog/509-proto-naming-conventions.md` (topic families + process progress fact identity/vocabulary)
- `backlog/496-eda-principles.md` (facts are not requests; outbox discipline)
- `backlog/511-process-primitive-scaffolding.md` (Temporal-backed Process contract; progress facts emitted via activities/ports)

## Decision (single option)

**Kanban is served by Projection query APIs and a Projection Updates stream, both defined in Protobuf and consumed only via generated SDKs.**

- Kanban boards are **Temporal-backed Process boards**: if a “board” is not backed by a Temporal ProcessDefinition (process progress facts), it must not be described or implemented as Kanban.
- UISurfaces MUST NOT read Temporal visibility/search/history for product Kanban truth.
- UISurfaces MUST NOT consume Kafka facts directly.
- Live updates are cursor-based notifications (`board_seq`) that trigger idempotent refetch (deltas preferred).

## Scope

This backlog defines:

1. A **capability-agnostic** Kanban view model (board/columns/cards).
2. A **capability-agnostic** query + delta API.
3. A **capability-agnostic** updates stream contract (server-streaming RPC payloads).

This contract is designed to serve multiple domains consistently (examples):

- Sales funnel boards (`process_definition_id="sales.funnel.v1"`)
- Transformation change boards (`process_definition_id="transformation.r2r.v1"`)
- SRE incident boards (`process_definition_id="sre.incident.v1"`)

It does **not** define:

- any particular capability’s mapping rules (phase → column) (projection config does this)
- storage schemas (Postgres tables, indexes) (see `backlog/615-kanban-fullstack-reference.md`)
- Temporal visibility/Search Attributes schemas (ops/debug only; out of scope)

## Where these protos live (recommended)

Kanban is a **platform-standard** projection surface. To avoid inventing per-capability Kanban message schemas, Kanban messages SHOULD live in the platform module:

- `packages/ameide_core_proto/src/ameide_core_proto/platform/kanban/v1/kanban.proto`

The platform SHOULD standardize a single Kanban Projection API **interface** (query + updates stream) that is implemented by Projection primitives and consumed by UISurfaces via generated SDKs.

This is a standard interface, not a central deployment:

- Capabilities MAY run separate Projection services, each implementing the same Kanban service interface.
- UISurfaces select the appropriate Projection endpoint for a given `KanbanBoardScope` via routing/service discovery (implementation detail).
- This avoids a global bottleneck while keeping one portable SDK contract.

## Identity (canonical fields)

These fields are required for rebuildable, methodology-agnostic boards:

- `board_kind` (stable discriminator for scope model)
- `process_definition_id` (stable identifier of the Process definition being visualized)
- `board_scope` (explicit fields; never opaque JSON; MUST include `board_kind` and `process_definition_id`)
- `board_id` (deterministic function of scope; projection-owned)
- `board_seq` (monotonic cursor derived from the projection’s durable commit cursor/sequence; not timestamps)
- `card_id` (stable id; default for process-driven boards: `process_instance_id` / WorkflowID)

**Workflow IDs used as `card_id` MUST NOT be reused** for new instances (see `backlog/614-kanban-projection-architecture.md`).

## Proto contracts (normative)

The following is the canonical shape for Kanban query + updates contracts.

### 1) Shared messages: `ameide_core_proto.platform.kanban.v1`

```proto
syntax = "proto3";

package ameide_core_proto.platform.kanban.v1;

import "ameide_core_proto/common/v1/pagination.proto";
import "ameide_core_proto/common/v1/request_context.proto";

message KanbanBoardScope {
  // Stable discriminator selecting which scope model to render for a given context.
  // Examples: "repository", "initiative", "pipeline", "subprocess".
  string board_kind = 1;

  // Recommended default scope axes (see 614/616).
  string tenant_id = 2;
  string organization_id = 3;
  string repository_id = 4;

  // The Process definition being visualized by this board.
  string process_definition_id = 5;

  // Optional axis for initiative-scoped boards.
  string initiative_id = 6;

  // Optional axis for “nested” boards (subprocess trees).
  string parent_card_id = 7;
}

message KanbanBoardRef {
  KanbanBoardScope scope = 1;

  // Deterministic id derived from scope by the projection.
  string board_id = 2;
}

message KanbanCardRef {
  string card_id = 1;
}

message KanbanColumn {
  string column_key = 1;
  string title = 2;
  int32 order = 3;
}

message KanbanCardView {
  string card_id = 1;
  string title = 2;
  string column_key = 3;

  // Display only; not used for correctness ordering.
  int64 updated_at_unix_ms = 4;

  // Correctness cursor: monotonic per-card update cursor.
  int64 seq = 5;

  // Optional user ranking key (LexoRank-style); not required.
  string rank = 6;

  // Optional drill-in link to the underlying process instance.
  string process_instance_id = 10; // WorkflowID (same as card_id for process boards)
  string process_run_id = 11;      // RunID (best-effort “current”; may change)
  string process_definition_id = 12;
  string phase_key = 13;
}

message KanbanBoard {
  KanbanBoardRef ref = 1;
  int64 board_seq = 2;

  repeated KanbanColumn columns = 3;

  // This list may be paginated by the query API (see GetKanbanBoardRequest.page).
  repeated KanbanCardView cards = 4;
}

message KanbanBoardDelta {
  KanbanBoardRef ref = 1;
  int64 board_seq = 2;

  // Changed cards since `since_seq`; either send full card views or ids + follow-up fetch.
  repeated KanbanCardView upserted_cards = 3;
  repeated string archived_card_ids = 4;
  repeated string deleted_card_ids = 5;
}
```

Notes:

- `board_seq` MUST be derived from a durable projection commit cursor/sequence (see `backlog/615-kanban-fullstack-reference.md`) and MUST advance only on effectful commits (not on inbox dedupe/no-ops).
- Boards are process-definition-centric; `process_definition_id` is part of board scope identity.

## Examples (multiple domains; same contract)

### Repository-scoped process board (Transformation)

```text
board_kind=repository
tenant_id=t-1
organization_id=o-1
repository_id=r-1
process_definition_id=transformation.r2r.v1
```

### Initiative-scoped process board (Transformation)

```text
board_kind=initiative
tenant_id=t-1
organization_id=o-1
repository_id=r-1
initiative_id=init-123
process_definition_id=transformation.r2r.v1
```

### Pipeline-scoped process board (Sales)

```text
board_kind=pipeline
tenant_id=t-1
organization_id=o-1
repository_id=sales-core
process_definition_id=sales.funnel.v1
```

### Repository-scoped incident process board (SRE)

```text
board_kind=repository
tenant_id=t-1
organization_id=o-1
repository_id=sre-core
process_definition_id=sre.incident.v1
```

### 2) Query + deltas: “Kanban Projection Query” service

```proto
syntax = "proto3";

package ameide_core_proto.platform.kanban.v1;

import "ameide_core_proto/common/v1/request_context.proto";
import "ameide_core_proto/common/v1/pagination.proto";
import "ameide_core_proto/platform/kanban/v1/kanban.proto";

service KanbanQueryService {
  rpc GetKanbanBoard(GetKanbanBoardRequest) returns (GetKanbanBoardResponse);
  rpc GetKanbanDeltas(GetKanbanDeltasRequest) returns (GetKanbanDeltasResponse);
}

message GetKanbanBoardRequest {
  RequestContext ctx = 1;
  KanbanBoardScope scope = 2;
  PageRequest page = 3;
  // Optional: restrict returned cards to a single column (useful for per-column paging).
  string column_key = 4;
}

message GetKanbanBoardResponse {
  KanbanBoard board = 1;
  PageResponse page = 2;
}

message GetKanbanDeltasRequest {
  RequestContext ctx = 1;
  KanbanBoardScope scope = 2;
  int64 since_seq = 3;
  PageRequest page = 4;
}

message GetKanbanDeltasResponse {
  KanbanBoardDelta delta = 1;
  PageResponse page = 2;
}
```

### 3) Updates stream: “Projection Updates” as server-streaming RPC

```proto
syntax = "proto3";

package ameide_core_proto.platform.kanban.v1;

import "ameide_core_proto/common/v1/request_context.proto";
import "ameide_core_proto/platform/kanban/v1/kanban.proto";

service KanbanUpdatesService {
  rpc WatchKanbanUpdates(WatchKanbanUpdatesRequest) returns (stream WatchKanbanUpdatesResponse);
}

message WatchKanbanUpdatesRequest {
  RequestContext ctx = 1;
  KanbanBoardScope scope = 2;
  int64 after_seq = 3;
}

message WatchKanbanUpdatesResponse {
  KanbanBoardRef ref = 1;
  int64 board_seq = 2;
}
```

The updates stream is:

- **at-least-once** (UI must be idempotent)
- **cursor-only** (UI refetches deltas/board; no attempt to stream the entire board state)

## Authorization (required)

Kanban query + update APIs MUST be authorization-aware:

- scope MUST include `{tenant_id, organization_id}` at minimum
- repository/initiative scoping MUST be enforced by the server
- query APIs MUST NOT allow cross-org data leakage by “board_id guessing”

This is one reason capability-owned Projection deployments are preferable: they can reuse existing auth policies while still exposing the standard Kanban service interface.

## Interaction streams are out of scope (by design)

Kanban is a projection view. It can show “in progress” state, but it MUST NOT transport interactive streaming (chat logs, tool logs, agent tokens).

Those belong to dedicated interactive surfaces (threads/agent runs) and dedicated streaming protocols, per:

- `backlog/616-kanban-principles.md` (Principle 8: interaction streams are out-of-band)
- `backlog/617-transformation-uisurface-wireframes.md` (activity surfaces)

## Implementation notes (SDK implications)

- UISurface imports only generated SDK clients; it never imports `.proto` sources directly (`backlog/520-primitives-stack-v2.md`).
- Projections implement the Kanban services and maintain `board_seq` as a durable cursor derived from projection commits.
- Processes emit progress facts (phase-first); projections consume those facts; Kanban derives columns via mapping config.

## Checklist (definition of done for “Kanban proto”)

- [ ] Shared `kanban.proto` exists and is reused by at least one capability.
- [ ] Query and updates services are defined in proto and consumed via generated SDKs.
- [ ] The updates stream is cursor-only and idempotent-refetch compatible (`board_seq`).
- [ ] Projections dedupe by `message_id` and converge under replay.
