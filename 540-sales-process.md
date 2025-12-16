# 540 Sales — Process component

**Status:** Implemented (Temporal workflows + ingress router)
**Parent:** `backlog/540-sales-capability.md`

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Process primitive), Application Events (process facts), Application Interfaces (topic families).

## Responsibilities (496-native)

- Orchestrate cross-aggregate and cross-system workflows without owning domain state:
  - quote approval workflow (timeouts/escalation),
  - acceptance → ERP/order handoff,
  - remediation queues and operational visibility.
- Consume domain facts and integration facts and emit **process facts** only.
- Request domain writes via domain intents/RPC; never write the Sales domain database.

## Contract pointers

- Process facts: `packages/ameide_core_proto/src/ameide_core_proto/sales/process/v1/process_facts.proto` (`sales.process.facts.v1`)
- Domain intents/facts: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/intents.proto`, `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/facts.proto`
- Integration facts: `packages/ameide_core_proto/src/ameide_core_proto/sales/integration/v1/integration_facts.proto`

## Implementation status

- Primitive: `primitives/process/sales`
- Worker + ingress scaffold: `primitives/process/sales/cmd/worker/main.go`, `primitives/process/sales/cmd/ingress/main.go`
- Workflows: `primitives/process/sales/internal/workflows/workflow.go` (quote approval + ERP handoff)
- Ingress router: `primitives/process/sales/internal/ingress/router.go` (SignalWithStart per workflow)

## Implementation progress (checklist)

- [x] Worker + ingress + workflow + state scaffold passes ProcessShape verification.
- [x] Deterministic Temporal workflows (uses `workflow.Now(ctx)`; no `time.Now()` inside workflow code).
- [x] Quote approval workflow emits `SalesProcessFact` events (requested + timed-out) on domain fact signals.
- [x] ERP handoff workflow triggers on `QuoteAccepted` and calls Integration `Execute` for `sales.erp.create_order.v1`.
- [x] `ameide primitive verify --kind process --name sales --mode repo` passes.

## Next steps (checklist)

- [ ] Replace stdin/dev ingress with a real stream consumer (domain facts + integration facts) plus checkpointing.
- [ ] Publish process facts to the platform event bus (not log-only), with DLQ + retry/backoff policy documented.
- [ ] Implement escalation + assignment semantics for approvals (approver selection, escalation policy refs, remediation queues).
- [ ] Add operational read models (handoff status, retries, failures) to Projection inputs/outputs.

## Clarification requests (checklist)

- [ ] What is the “source of truth” for approval policy decisions (policy service, static rules, config refs)?
- [ ] What are the SLA defaults for quote approval timeouts (per tenant/policy), and how are they configured?
- [ ] Temporal conventions: namespace/task queue naming, workflow ID format, and retention/replay expectations per environment.
- [ ] ERP handoff contract: what constitutes “ack”, “reject”, “retryable failure”, and human remediation routing?
