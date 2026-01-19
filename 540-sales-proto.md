# 540 Sales — Proto Contract Specification

**Status:** Implemented (canonical files landed)
**Parent:** `backlog/540-sales-capability.md`

This document specifies the **proto contracts** for Sales following `backlog/496-eda-principles-v6.md` and `backlog/509-proto-naming-conventions-v6.md`.

> **No embedded proto text.** This document is a **file index + invariants**. The canonical proto files are the source of truth.

## 1) Package structure (canonical paths)

```
packages/ameide_core_proto/src/ameide_core_proto/
└── sales/
    ├── core/
    │   └── v1/
    │       ├── sales.proto                 # Common types (meta, subject, enums)
    │       ├── opportunity.proto           # Lead + opportunity messages
    │       ├── quote.proto                 # Quote messages
    │       ├── intents.proto               # SalesDomainIntent envelope
    │       ├── facts.proto                 # SalesDomainFact envelope
    │       ├── sales_command_service.proto # Domain write boundary RPC
    │       └── sales_query_service.proto   # Projection query RPC
    ├── process/
    │   └── v1/
    │       └── process_facts.proto         # SalesProcessFact envelope
    └── integration/
        └── v1/
            └── integration_facts.proto     # SalesIntegrationFact envelope
```

## 2) Topic families

| Topic | Message Type | Producer | Purpose |
|------|--------------|----------|---------|
| `sales.domain.intents.v1` | `SalesDomainIntent` | Domain clients (UI/Process/Agent) | Requests for state changes |
| `sales.domain.facts.v1` | `SalesDomainFact` | Sales Domain | Immutable facts after persistence |
| `sales.process.facts.v1` | `SalesProcessFact` | Sales Process | Workflow state transition facts |
| `sales.integration.facts.v1` | `SalesIntegrationFact` | Sales Integration | External system outcomes (never domain facts) |

## 3) Envelope invariants (496-aligned)

All Sales envelopes MUST carry:

- **Tenant isolation:** `meta.tenant_id`
- **Idempotency key:** `meta.message_id` (unique per message)
- **Traceability:** `meta.correlation_id`, `meta.causation_id`, `meta.occurred_at`, `meta.producer`, `meta.schema_version`
- **Partitioning keys:** `{meta.tenant_id, subject.aggregate_id}` for domain intents/facts; `{meta.tenant_id, subject.workflow_id}` for process facts

## 4) Optional scope note (Leads)

Leads are treated as an **optional module** in v1:

- Domain RPC includes Lead methods, but deployments may choose to route leads to an external CRM and only use opportunities/quotes in Sales.
- If Leads are externalized, keep the envelope types for integration compatibility but gate the command handlers behind deployment configuration.

## Implementation progress (checklist)

- [x] Canonical proto files exist at the paths in §1/§5.
- [x] Package namespaces align: `sales/core/v1/*` → `ameide_core_proto.sales.core.v1`.
- [x] Package namespaces align: `sales/process/v1/*` → `ameide_core_proto.sales.process.v1`.
- [x] Package namespaces align: `sales/integration/v1/*` → `ameide_core_proto.sales.integration.v1`.
- [x] Envelopes declare partition keys and idempotency key as `meta.message_id` (496-aligned).
- [x] Leads are explicitly labeled as an optional module in the RPC surface (`SalesCommandService`).
- [ ] “Contract vs roadmap” is not explicitly separated anywhere (proto is v1; domain/process docs may list future work).

## Next steps (checklist)

- [ ] Add/confirm a versioning policy for Sales protos (breaking change rules, reserved fields, schema evolution).
- [ ] Add cross-capability consistency checks to CI (topic family naming, eventing options, required meta fields).
- [ ] Decide whether Sales “intent topic” is normative for v1 deployments or optional (RPC-only vs bus-first).

## Clarification requests (checklist)

- [ ] Are Leads truly optional for v1 deployments, or should Sales core contract exclude them until a CRM integration is defined?
- [ ] What is the canonical `schema_version` format and bump policy (semver, date-based, or generated)?
- [ ] Are Process facts required to carry a monotonic workflow step/version (beyond `meta.message_id`) for replay/debuggability?

## 5) Canonical file index

- Common types: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales.proto`
- Aggregates:
  - `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/opportunity.proto`
  - `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/quote.proto`
- Envelopes:
  - `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/intents.proto`
  - `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/facts.proto`
  - `packages/ameide_core_proto/src/ameide_core_proto/sales/process/v1/process_facts.proto`
  - `packages/ameide_core_proto/src/ameide_core_proto/sales/integration/v1/integration_facts.proto`
- RPC surfaces:
  - Domain: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_command_service.proto`
  - Projection: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_query_service.proto`
