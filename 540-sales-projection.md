# 540 Sales — Projection component

**Status:** Implemented (Postgres read models + idempotent apply)
**Parent:** `backlog/540-sales-capability.md`

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Projection primitive), Application Service (query RPC), Data Objects (read models).

## Responsibilities (496-native)

- Build read models from at-least-once streams; consumers must be idempotent and replay-safe.
- Expose query surfaces that UIs and Agents depend on (read-only).
- Never publish domain facts and never accept commands.

## Contract pointers

- Query RPC: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_query_service.proto`
- Inputs: `sales.domain.facts.v1`, `sales.process.facts.v1`, `sales.integration.facts.v1`

## Implementation status

- Primitive: `primitives/projection/sales`
- State: Postgres-backed read models for opportunities, quotes, pipeline summary, and approval work items.
- Key files: `primitives/projection/sales/internal/adapters/postgres/store.go`, `primitives/projection/sales/internal/handlers/handlers.go`
- Migrations: `primitives/projection/sales/internal/adapters/postgres/migrations/V1__sales_projection.sql` (embedded + applied on startup)
- Dev ingest tool: `primitives/projection/sales/cmd/ingress/main.go` (stdin base64 domain/process facts)

## Implementation progress (checklist)

- [x] Query RPC (`SalesQueryService`) backed by a concrete store (no placeholder responses).
- [x] Postgres schema + embedded migrations for read models and projection inbox.
- [x] Idempotent apply semantics via `projection_inbox` keyed by `(tenant_id, message_id)`.
- [x] Domain + process fact application implemented (opportunities, quotes, approval work items).
- [x] `ameide primitive verify --kind projection --name sales --mode repo` passes.

## Next steps (checklist)

- [ ] Replace stdin ingest with a real consumer loop (domain/process/integration facts) + checkpoint strategy (consumer group + offsets).
- [ ] Apply integration facts (`sales.integration.facts.v1`) and expose operational views if needed (email status, ERP statuses).
- [ ] Document replay/backfill procedure and expected convergence guarantees (at-least-once semantics).
- [ ] Add performance indexes once real query patterns are known (stage/owner filters, quote status filters, approval queue ordering).

## Clarification requests (checklist)

- [ ] What is the canonical checkpoint store (DB table, Kafka offsets, or platform-managed checkpoint service)?
- [ ] Which read models are “must ship” for v0 UI/agent flows beyond the current set (forecast rollups, audit timeline, handoff status)?
- [ ] Are multi-currency pipeline totals required, or is single-currency per tenant assumed?
