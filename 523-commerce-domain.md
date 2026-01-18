# 523 Commerce — Domain component

**Status:** Implemented (v1 BYOD storefront hostname writer + inbox/outbox + tests; functional scope outside BYOD pending)

Implementation (repo): `primitives/domain/commerce`

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Domain primitives), Application Services/Interfaces (RPC + topic families), Application Events (facts), Data Objects (proto messages).
- **Out-of-scope layers:** Strategy/Business catalogs (see `523-commerce.md`); Technology/runtime wiring (see `473-ameide-technology.md`).

## Intent

Define the headless commerce engine as a bounded context: stable APIs + facts/intents; UIs remain thin.

This is an umbrella for multiple Domain primitives under the “commerce” platform domain (retail suites always separate at least Catalog/Pricing/Orders/Inventory/Payments/StoreOps/Channels).

Start with a small number of subdomains:

1. Catalog & Merchandising
2. Pricing & Promotions
3. Cart/Checkout & Orders/Returns
4. Site / Storefront (domains/branding → channel selection)
5. SalesChannel (selling context/policy)
6. Inventory & Stock Locations (physical nodes; sourcing/ATP)
7. Store Operations (register/device/shift/receipts)
8. Customer & Loyalty (optional early; often external)

Payments/hardware drivers are integrations; the domain owns only payment intent state and idempotency.

## Implementation progress (current)

Implemented (v1 BYOD domains):

- [x] gRPC write service for claim/mapping lifecycle (business-verb RPCs).
- [x] Emits `commerce.domain.facts.v1` via transactional outbox (publisher wiring beyond “log” remains pending).
- [x] Postgres schema + Flyway migrations include outbox/inbox and BYOD claim/mapping tables.
- [x] Guardrails in place (tests, non-root container, `ameide primitive verify` passes).

Not yet implemented (still design-level in this backlog):

- [ ] SalesChannel/Site aggregates + facts (storefront routing policy becomes data, not code).
- [ ] Orders/OMS aggregates + facts (idempotent checkout, fulfillment/returns).
- [ ] Inventory/ATP aggregates + facts (stock locations, availability and sourcing).
- [ ] StoreOps aggregates + facts (register/device/shift/receipts).
- [ ] Payments domain state machine (PaymentIntent) and invariants (integration-first vs embedded minimal ledger).

## Clarification requests (next steps)

Confirm/decide:

- [ ] Whether hostname uniqueness is global across tenants (current schema enforces global uniqueness) and what transfer/cooldown rules apply.
- [ ] Whether wildcard hostnames are allowed in v1 (docs mention “exact hostname v1”, but the proto enum includes wildcard).
- [ ] Canonical `site_id`/`sales_channel_id` selection rule for storefront routing (resolved host → site → default channel vs explicit mapping).
- [ ] Which additional Domain subdomains are in v1, and the explicit “Domain vs Process ownership table” for each workflow.

Build-out checklist (Domain v1+):

- [ ] Define the v1 aggregate set (names, IDs, invariants, required facts) and freeze fact type identifiers.
- [ ] Implement outbox publisher to the platform broker (once broker target is confirmed).
- [ ] Add Domain-side validation surfaces for Integrations (webhook idempotency keys, provider correlation IDs).

## Reference analogs (D365 CRT / SAP OCC)

- D365 CRT is the “business logic engine” analog (logic lives server-side, not in clients).
- D365 CSU/Core is a hosting/deployment unit analog (cloud and edge).
- SAP OCC is a public REST API analog (stable contract surface).

## Commerce identity & scope model (tenant/site/channel/location)

Commerce behavior is scoped across distinct axes; do not overload one ID to mean everything:

- `tenant_id`: security/billing boundary
- `site_id`: web presence (domains/base URLs) and default channel selection
- `sales_channel_id`: selling context (currency, tax, price lists, assortment, eligible tenders/shipping)
- `stock_location_id`: physical fulfillment/inventory node
- `store_site_id`: edge deployment unit (store cluster) typically aligned to a store channel + stock location

Make `sales_channel_id` a first-class identifier used by:

- assortment/catalog visibility,
- pricing/promo eligibility,
- tax/shipping/payment method eligibility,
- inventory sourcing/availability.

Rule of thumb:

- every customer-facing decision RPC should require `tenant_id` + `sales_channel_id` (and usually `currency`), and inventory/fulfillment decisions must include `stock_location_id` (or return results keyed by location).

## EDA responsibilities (496-native)

Domains are the single-writer for their aggregates and the authoritative producers of **domain facts (Application Events)**:

- Commands/intents use business verbs (avoid CRUD) per `backlog/484a-ameide-cli-primitive-workflows.md` and the integration posture in `backlog/496-eda-principles-v6.md`.
- State change and fact emission is atomic via transactional outbox (or outbox-equivalent) per `backlog/496-eda-principles-v6.md`.
- Consumers assume at-least-once; emitted facts must support idempotency/ordering (aggregate ref + monotonic version).

Topic families (v1):

- commands/intents: `commerce.domain.intents.v1` (optional if RPC-first)
- domain facts: `commerce.domain.facts.v1` (required)

See `523-commerce-proto.md` for proposed envelopes and aggregator message shapes.

## Proto shape (v2-aligned)

Follow `backlog/509-proto-naming-conventions-v6.md`.

Recommended starting packages/topics:

- `package ameide_core_proto.commerce.v1;`
- intents: `commerce.domain.intents.v1` → `CommerceDomainIntent`
- facts: `commerce.domain.facts.v1` → `CommerceDomainFact`

If this grows too broad, split into subcontexts (keep SalesChannel concept shared):

- `ameide_core_proto.commerce.catalog.v1`, `...commerce.pricing.v1`, etc.

## Stack alignment (proto / generation / operator)

- Proto: APIs + intent/fact schemas only (no endpoints, secrets, or runtime policy).
- Generation: `buf generate` produces SDKs and compile-time-enforcing server skeletons; drift is caught by regen-diff + compile failures.
- Operator: reconciles Deployments/Services/HTTPRoutes and injects DB/broker/config via Kubernetes; operators do not interpret business semantics (per `backlog/520-primitives-stack-v6.md`).

## Core invariants (examples)

- Pricing/promo evaluation is deterministic given `{sales_channel_id, currency, effective_at, ruleset_version}`.
- Checkout/order creation is idempotent (message-id / order-id safe retries).
- Store ops state machines are explicit (register/shift open/close, reconcile).
- Store site is a first-class identity boundary for configuration and replication; stock locations are the physical availability boundary.

## API sketch (non-exhaustive)

Sales channels:

- `CreateSalesChannel`, `UpdateSalesChannel`
- `AssignCatalogToChannel`, `AssignPriceBookToChannel`, `AssignPromotionsToChannel`
- `ConfigureChannelPolicies` (tax/shipping/payment eligibility as data)
- `AssociateStockLocationsToChannel` (which locations can fulfill which channels)

Sites/storefronts:

- `CreateSite`, `UpdateSite`
- `SetDefaultChannelForSite` (and/or `MapSiteToChannels`)

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

These facts feed projections and replication; prefer transactional outbox discipline per v2.
