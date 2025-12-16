# 540 Sales — Integration component

**Status:** Implemented (deterministic Execute adapters)
**Parent:** `backlog/540-sales-capability.md`

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Integration primitive), Application Events (integration facts), Application Interfaces (webhooks/adapters).

## Responsibilities (496-native)

- Adapt external systems (ERP, e-signature, email delivery, CRM) into Sales’ contract model.
- Provide idempotent inbound handling (webhooks) and safe retries/backoff for outbound calls.
- Emit **integration facts** only; never emit Sales domain facts.

## Contract pointers

- Integration facts: `packages/ameide_core_proto/src/ameide_core_proto/sales/integration/v1/integration_facts.proto` (`sales.integration.facts.v1`)

## Implementation status

- Primitive: `primitives/integration/sales`
- State: Implements stable `IntegrationV0Service.Execute` IDs:
  - `sales.erp.create_order.v1` (deterministic ERP order id)
  - `sales.quote.send_email.v1` (deterministic provider message id)

## Implementation progress (checklist)

- [x] Integration primitive exists and passes `ameide primitive verify --kind integration --name sales --mode repo`.
- [x] Stable `Execute` IDs implemented with deterministic outputs for repeatability in tests/dev.
- [ ] Integration facts emission (`SalesIntegrationFact` → `sales.integration.facts.v1`) is not wired yet (adapter currently returns outputs but does not publish facts).

## Next steps (checklist)

- [ ] Define and implement concrete external adapters (ERP create order, email provider, e-signature) behind `Execute` IDs with retries/backoff.
- [ ] Emit `SalesIntegrationFact` for each `Execute` outcome and ensure idempotency by `message_id`.
- [ ] Add inbound webhook handling shape (if needed) with dedupe + signature validation + replay protection.
- [ ] Add DLQ + alerting for repeated integration failures (per 496 operability expectations).

## Clarification requests (checklist)

- [ ] What external systems are authoritative for order creation and email delivery per environment (and how are endpoints/config bound outside proto)?
- [ ] What is the expected payload contract for `sales.erp.create_order.v1` (required fields, schemas, versioning)?
- [ ] Should integration “failures” be modeled as facts only, or also trigger process remediation workflows?
