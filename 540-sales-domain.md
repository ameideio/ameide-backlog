# 540 Sales — Domain component

**Status:** Implemented (Postgres + inbox/outbox + dispatcher)
**Parent:** `backlog/540-sales-capability.md`

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Domain primitive), Application Service/Interface/Event, Data Object.

## Responsibilities (496-native)

- Own and persist Sales aggregates (single-writer boundary).
- Expose command RPC (`SalesCommandService`) as the primary write interface.
- Publish **domain facts** (`SalesDomainFact`) via transactional outbox after persistence.
- Accept intents either via RPC or via `sales.domain.intents.v1` (deployment choice); facts are always bus-first.

## In-scope aggregates (v1 starter set)

- Lead (optional module)
- Opportunity
- Quote

## Contract pointers

- Command RPC: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_command_service.proto`
- Domain intent envelope: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/intents.proto`
- Domain fact envelope: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/facts.proto`

## Implementation status

- Primitive: `primitives/domain/sales`
- State: Postgres-backed domain with `domain_inbox` idempotency, transactional `domain_outbox`, and a dispatcher loop.
- Key files: `primitives/domain/sales/cmd/main.go`, `primitives/domain/sales/cmd/dispatcher/main.go`, `primitives/domain/sales/internal/handlers/handlers.go`, `primitives/domain/sales/internal/dispatcher/dispatcher.go`
- Events catalog: `primitives/domain/sales/events/catalog.go`

## Implementation progress (checklist)

- [x] Durable persistence (Postgres) with schema migrations under `primitives/domain/sales/migrations/`.
- [x] Inbox dedupe (`domain_inbox`) keyed by `(tenant_id, message_id)` to satisfy 496 idempotency expectations.
- [x] Transactional outbox writes (`domain_outbox`) emitted alongside state changes.
- [x] Dispatcher loop that publishes outbox rows and records attempts/errors.
- [x] `ameide primitive verify --kind domain --name sales --mode repo` passes.

## Next steps (checklist)

- [ ] Replace dev/log publisher with the platform event bus publisher (Kafka/NATS) and wire it into the dispatcher.
- [ ] Add “contract-level” integration tests: command → persisted state → fact in outbox → published.
- [ ] Add explicit domain constraints (stage transition rules, quote status transitions) if required by the business spec.
- [ ] Add metrics/tracing (outbox lag, publish failures, command latency, inbox hit-rate).

## Clarification requests (checklist)

- [ ] Leads: are they truly optional at runtime (feature-flagged/disabled), or always part of core Sales?
- [ ] Opportunity stage transitions: is any stage-to-stage move allowed, or is there a strict DAG?
- [ ] Money semantics: currency rules, rounding/normalization policy, and whether multi-currency opportunities are supported.
- [ ] Quote lifecycle governance: is approval policy evaluation purely Process-owned, Domain-owned, or split (Domain validates; Process orchestrates)?
