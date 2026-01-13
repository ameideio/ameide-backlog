# 540 — Sales Capability (Define the capability, then realize it via primitives)

**Status:** Implemented (end-to-end v0 primitives + contracts)
**Audience:** Architecture, platform engineering, operators/CLI, agent/runtime teams
**Scope:** Define **Sales** as a capability realized by Ameide primitives (Domain/Process/Projection/Integration/UISurface/Agent) with 496-native EDA contracts (intents vs facts, outbox, idempotent consumers).

**Use with:**
- EDA contract rules: `backlog/496-eda-principles-v2.md`, `backlog/496-eda-protobuf-ameide.md`
- Proto conventions: `backlog/509-proto-naming-conventions.md`
- Primitives stack guardrails: `backlog/520-primitives-stack-v2.md`
- Primitive testing discipline: `backlog/537-primitive-testing-discipline.md`

## Layer header (Strategy + Business + Application realization)

- **Primary ArchiMate layer(s):** Strategy (Capability, Value Stream) + Business (Business Processes, Roles) with Application realization via primitives.
- **Primary element types used:** Capability, Value Stream, Business Process, Application Component, Application Service/Interface/Event, Data Object.
- **Prohibited unless qualified:** process, service, domain, event (qualify per `529-archimate-alignment-470plus.md`).

## 0) Problem

Sales workflows exist implicitly across CRM/ERP integrations and ad-hoc UIs, but we lack a single authoritative capability definition that:

- names the stable contracts (proto packages, topic families, query surfaces),
- makes primitive ownership explicit (who writes facts, who orchestrates, who projects),
- keeps environment bindings out of proto and keeps cross-domain invariants in Process primitives.

## 1) Definition

Sales is Ameide’s **sell-to-order capability**: it captures and governs the commercial lifecycle from lead/opportunity through quote approval and acceptance, and hands off to ERP/order systems without embedding ERP semantics in the Sales domain.

Non-goals (v1): invoicing/collections, customer support/case management, full activity capture (email/calls), territory/quota management (unless explicitly added).

## 2) Value streams (starter set)

- **Opportunity lifecycle:** create → qualify → stage changes → close won/lost
- **Quote lifecycle:** create draft → submit for approval → approve/reject → send → accept/decline
- **Approval workflow:** enforce policy gates and audit trail (Process primitive)
- **Acceptance → order handoff:** request ERP order creation and track outcomes (Process + Integration)

## 3) Primitive decomposition (ownership boundaries)

- **Domain (Sales):** canonical writer for Sales aggregates; exposes command RPC; emits *domain facts* via outbox.
- **Process (Sales):** orchestrates cross-aggregate and cross-system workflows (approvals, handoff); emits *process facts*; requests writes via domain intents/RPC (never writes domain DB).
- **Projection (Sales):** read model builder; idempotent consumer; exposes query RPC for UI/agents.
- **Integration (Sales):** boundary adapter to external systems (ERP/e-signature/email); idempotent webhook/execute; emits *integration facts* only.
- **UISurface (Sales):** UI/API surface; reads via projection; writes via domain commands/intents.
- **Agent (Sales Copilot):** reads via projection; proposes typed intents; no direct domain writes.

## 4) Contract set (proto-first)

See `backlog/540-sales-proto.md` for the canonical file index and invariants.

- Topics:
  - `sales.domain.intents.v1` → `ameide_core_proto.sales.core.v1.SalesDomainIntent`
  - `sales.domain.facts.v1` → `ameide_core_proto.sales.core.v1.SalesDomainFact`
  - `sales.process.facts.v1` → `ameide_core_proto.sales.process.v1.SalesProcessFact`
  - `sales.integration.facts.v1` → `ameide_core_proto.sales.integration.v1.SalesIntegrationFact`
- Services:
  - Domain: `ameide_core_proto.sales.core.v1.SalesCommandService`
  - Projection: `ameide_core_proto.sales.core.v1.SalesQueryService`

## 5) Implementation status (repo pointers)

Canonical protos:
- `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_command_service.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_query_service.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/intents.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/facts.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/sales/process/v1/process_facts.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/sales/integration/v1/integration_facts.proto`

Implemented primitives (runtime logic + tests):
- Domain: `primitives/domain/sales`
- Projection: `primitives/projection/sales`
- Process: `primitives/process/sales`
- Integration: `primitives/integration/sales`
- UISurface: `primitives/uisurface/sales`
- Agent: `primitives/agent/sales-copilot`

Backlogs:
- Domain: `backlog/540-sales-domain.md`
- Process: `backlog/540-sales-process.md`
- Projection: `backlog/540-sales-projection.md`
- Integration: `backlog/540-sales-integration.md`
- UISurface: `backlog/540-sales-uisurface.md`
- Agent: `backlog/540-sales-agent.md`

## Implementation progress (checklist)

- [x] Canonical Sales protos exist and define topic families + RPC surfaces (`backlog/540-sales-proto.md`).
- [x] Sales Domain primitive exists and passes repo verification (`primitives/domain/sales`).
- [x] Sales Process primitive exists and passes repo verification (`primitives/process/sales`).
- [x] Sales Projection primitive exists and passes repo verification (`primitives/projection/sales`).
- [x] Sales Integration primitive exists and passes repo verification (`primitives/integration/sales`).
- [x] Sales UISurface primitive exists and passes repo verification (`primitives/uisurface/sales`).
- [x] Sales Agent primitive exists and passes repo verification (`primitives/agent/sales-copilot`).

## Next steps (checklist)

- [ ] Replace dev/log-only ingress/egress with real event bus wiring (outbox publisher, stream consumers, DLQ policy).
- [ ] Ship integration-fact emission end-to-end (`SalesIntegrationFact` on real adapter outcomes) and apply those to projections.
- [ ] Implement a minimal UISurface workflow (read pipeline/quotes + proposal-first write flows).
- [ ] Implement agent “read skills” against projection + typed intent proposal generation with confirmation and policy checks.

## Clarification requests (checklist)

- [ ] Leads: in-scope for v1, or external CRM integration only (and should the RPC surface be gated/removed)?
- [ ] Quote approvals: where does policy evaluation live (Process, a policy service, or Domain validation + Process orchestration)?
- [ ] External systems: which ERP/e-signature/email providers are v1 targets and what are the required contract fields for handoff.
- [ ] Operational conventions: topic retention, replay/backfill expectations, and consumer checkpoint storage choice.
