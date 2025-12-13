# 523 Commerce — Domain component

## Intent

Define the headless commerce engine as a bounded context: stable APIs + facts/intents; UIs remain thin.

Start with a small number of subdomains:

1. Catalog & Merchandising
2. Pricing & Promotions
3. Cart/Checkout & Orders/Returns
4. Inventory & Stock Locations (may start as a projection-backed API)
5. Store Operations (register/device/shift/receipts)
6. SalesChannel / Site (cross-cutting axis: channel-scoped configuration and policy)
7. Customer & Loyalty (optional early; often external)

Payments/hardware drivers are integrations; the domain owns only payment intent state and idempotency.

## Reference analogs (D365 CRT / SAP OCC)

- D365 CRT is the “business logic engine” analog (logic lives server-side, not in clients).
- D365 CSU/Core is a hosting/deployment unit analog (cloud and edge).
- SAP OCC is a public REST API analog (stable contract surface).

## SalesChannel is required context (not just tenant)

Commerce behavior is channel-scoped. Make `sales_channel_id` (or `site_id`) a first-class identifier used by:

- assortment/catalog visibility,
- pricing/promo eligibility,
- tax/shipping/payment method eligibility,
- inventory sourcing/availability.

Rule of thumb:

- every customer-facing decision RPC should require `tenant_id` + `sales_channel_id` (and usually `currency`).

## Proto shape (v2-aligned)

Follow `509-proto-naming-conventions.md`.

Recommended starting packages/topics:

- `package ameide_core_proto.commerce.v1;`
- intents: `commerce.domain.intents.v1` → `CommerceDomainIntent`
- facts: `commerce.domain.facts.v1` → `CommerceDomainFact`

If this grows too broad, split into subcontexts (keep SalesChannel concept shared):

- `ameide_core_proto.commerce.catalog.v1`, `...commerce.pricing.v1`, etc.

## Stack alignment (proto / generation / operator)

- Proto: APIs + intent/fact schemas only (no endpoints, secrets, or runtime policy).
- Generation: `buf generate` produces SDKs and compile-time-enforcing server skeletons; drift is caught by regen-diff + compile failures.
- Operator: reconciles Deployments/Services/HTTPRoutes and injects DB/broker/config via Kubernetes; operators do not interpret business semantics (per `520-primitives-stack-v2.md`).

## Core invariants (examples)

- Pricing/promo evaluation is deterministic given `{sales_channel_id, currency, effective_at, ruleset_version}`.
- Checkout/order creation is idempotent (message-id / order-id safe retries).
- Store ops state machines are explicit (register/shift open/close, reconcile).
- Store site is a first-class identity boundary for configuration and replication.

## API sketch (non-exhaustive)

Sales channels:

- `CreateSalesChannel`, `UpdateSalesChannel`
- `AssignCatalogToChannel`, `AssignPriceBookToChannel`, `AssignPromotionsToChannel`
- `ConfigureChannelPolicies` (tax/shipping/payment eligibility as data)

Catalog:

- `GetProduct`, `GetAssortment` (search/browse typically via Projection query API)

Pricing:

- `EvaluatePrices(tenant_id, sales_channel_id, …)`, `EvaluatePromotions(tenant_id, sales_channel_id, …)`

Cart/Orders:

- `CreateCart(tenant_id, sales_channel_id, …)`, `AddLine`, `Checkout`, `CreateOrder`, `ReturnOrder`

Store ops:

- `OpenShift`, `CloseShift`, `GetRegisterStatus`

Customer/loyalty (optional):

- `GetCustomer`, `CreateCustomer` (may be real-time required)

## Facts/events (sketch)

- `ProductPublished`, `PriceBookActivated`, `PromotionActivated`
- `OrderCreated`, `OrderPaid`, `OrderRefunded`
- `ShiftOpened`, `ShiftClosed`, `ReceiptIssued`
- `CustomerCreated`, `LoyaltyRedeemed` (if owned)

These facts feed projections and replication; prefer outbox discipline per v2.

