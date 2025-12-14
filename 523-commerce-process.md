# 523 Commerce — Process component

## Layer header (Application, with Business Process alignment)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Process primitives), Application Events (domain facts + process facts), Application Services (commands/requests), Data Objects.
- **Out-of-scope layers:** Strategy/Business definitions (Capability/Value Streams/Business Processes live in `523-commerce.md`), except where referenced for traceability.

Disambiguation: “Business Process” (Business layer) ≠ “Process primitive” (Application Component). This doc specifies Process primitives that orchestrate commerce workflows.

## Intent

Define the deterministic workflows (Temporal) that make commerce operable at fleet scale:

- storefront BYOD domain onboarding (claim/verify/issue/attach),
- store-site provisioning and rollouts (scale unit),
- replication control (downsync/upsync, backfills, recovery),
- explicit handling of real-time required operations.

Process protos define workflow envelopes and process facts; operators remain operational only (`520-primitives-stack-v2.md`).

## Stack alignment (proto / generation / operator)

- Proto: workflow envelopes + process facts in `ameide_core_proto.process.commerce.v1`.
- Generation: Temporal worker wiring + determinism/versioning guardrails and tests.
- Operator: reconciles workers/HPAs/ingress and surfaces Temporal dependency status via conditions.

## EDA responsibilities (496-native)

Processes implement sagas and cross-domain invariants under eventual consistency:

- Consume domain facts (at-least-once) and orchestrate workflows per `496-eda-principles.md` Principle 5.
- Request changes through domain commands (RPC) or domain intents (topic); never bypass single-writer domains.
- Emit process facts to `commerce.process.facts.v1` so orchestration progress is observable and debuggable.
- Use explicit workflow IDs/idempotency keys; expect duplicates and retries everywhere.

See `523-commerce-proto.md` for the proposed process fact envelope and topic family.

For an IT4IT value-stream lens (R2D/R2F/D2C) over the same primitives/process model, see `525-it4it-value-stream-mapping.md`.

## Real-time required surface (v1)

Default is async replication. The system MUST label and handle “real-time required” operations explicitly:

- gift cards (if external)
- loyalty redemption/earn (if external)
- customer create/update (if centrally governed)
- cross-store returns/exchanges
- inventory adjustments that affect other sites
- fraud checks / risk holds (provider-dependent)

## Business processes (Business layer) → workflow families (Application layer)

These are the “truth tests” the platform must answer end-to-end; each maps to workflows below:

1. Browse & price discovery
2. Cart & checkout
3. Pay (authorize/capture/void/refund)
4. Fulfill (ship/pickup)
5. Returns/exchanges/refunds
6. Store selling (POS: shift/register/cash)
7. Catalog/price/promo distribution (HQ → channels/sites/stores)

## Business: roles and business objects (v1 anchor)

Roles/actors (examples):

- Customer / shopper
- Store associate / cashier
- Store manager
- Merchandiser / category manager
- Pricing/promo manager
- Fulfillment / warehouse operator
- Customer support / CSR

Business objects (examples):

- Product, assortment, price book, promotion
- Cart, order, return, refund
- Payment intent, tender, receipt
- Inventory item, stock location, transfer, adjustment
- Register, device, shift

## Value streams (Strategy layer) → workflow sets (Application layer)

This section anchors Process design in retailer day-to-day operations (value streams), not just onboarding/infra.

### Cross-cutting context

All workflows assume:

- `tenant_id` (tenant boundary)
- `site_id` (web presence boundary; domains/base URLs)
- `sales_channel_id` (selling context; required for most commerce decisions)
- `stock_location_id` (physical fulfillment/inventory node)
- `store_site_id` (edge deployment unit; typically aligned to a store channel + stock location)

### 0) Platform onboarding & tenant/channel provisioning

Golden path:

- Create tenant → create site(s) + sales channel(s) → connect storefront domain → go live.

Initiating UISurface:

- `commerce-admin`

Process workflows (sketch):

- BYOD: `VerifyDomainClaim`, `ProvisionDomainMapping`, `RevokeDomainMapping`
- Store site: `ProvisionStoreSite`, `RolloutStoreFleet`

Projections:

- Hostname resolution: `hostname -> {tenant_id, site_id, default_sales_channel_id, uisurface_ref, status}`

Integrations:

- Gateway API + cert-manager + DNS verification

Facts (Application Events):

- `commerce.process.facts.v1`: `TenantProvisioned`, `SalesChannelProvisioned`, `DomainMappingReady`, `StoreSiteReady`

### A) Storefront custom domains

- `VerifyDomainClaim` (DNS TXT verification)
- `ProvisionDomainMapping` (cert issuance + gateway wiring + route)
- `RevokeDomainMapping` (day-2 cleanup)

Failure taxonomy (status/conditions should expose these):

- DNS target incorrect (wrong CNAME/A record)
- TXT mismatch or missing
- DNS propagation pending (retry with backoff)
- CAA blocks ACME issuance
- proxy/CDN interference
- rate limits / transient ACME errors

Transfers:

- revocation and re-claim MUST be explicit and auditable
- consider a cooldown period before a hostname can be claimed by a different tenant

### B) Store site provisioning (edge)

`ProvisionStoreSite`

- bootstraps store identity, credentials/certs
- configures replication bindings
- validates compatibility constraints
- emits a process fact on readiness

Channel binding:

- a store site MUST declare its `sales_channel_id` and `stock_location_id`; replication bindings are derived from that context.

### C) Replication control

- `RunDownsync(store_site_id)`
- `RunUpsync(store_site_id)`
- `RebuildStoreEdgeState(store_site_id)` (replay/backfill; includes caches and operational store rehydration)

### Two-lane replication model (required)

Replication is explicitly hybrid:

- **Lane 1 (async distribution):** publish/refresh master + configuration + effective-dated rulesets to edge (catalog subset, price books, promotions, tenders, device/register config).
- **Lane 2 (real-time escape hatch):** a small set of operations must call a real-time service (see “Real-time required surface”) and must be gated in UX when WAN is down.

Store edge mode assumes a transactional operational store deployed as Domains (write model) plus derived read caches/projections; do not treat the operational store as “just a projection”.

### 1) Plan-to-merchandise (Plan & Market)

Retail meaning:

- define products, assortments, price books, promotions, and content; publish to channels with effective dates.

Golden paths:

- Create product → publish to channel assortment → publish price book → activate promotion.

Initiating UISurface:

- `commerce-admin`

Process workflows (sketch):

- `PublishAssortmentToChannel`
- `ActivatePriceBook`
- `ActivatePromotion`

Projections:

- product discovery/search (channel-scoped)
- effective price read model (by channel + currency + time)
- availability read model (by channel + sourcing rules)

Integrations:

- PIM/MDM imports
- DAM/CMS for assets/content (optional early)

### 2) Sell-to-serve / Order-to-cash (POS + eCommerce)

Retail meaning:

- quote/cart → checkout → payment → receipt/invoice → post-order changes → returns/refunds → settlement.

Golden paths:

- POS cash-and-carry: build cart → price/promos → take payment → create order → issue receipt.
- eCommerce: checkout → authorize → create order → later capture on shipment (or immediate capture).
- return/refund: validate return policy → refund tender → update inventory (and sync).

Initiating UISurfaces:

- `commerce-pos`
- `commerce-storefront`
- `commerce-admin` (CSR/call-center workflows)

Process workflows (sketch):

- `Checkout` (reserve/allocate → authorize → create order → emit facts)
- `CapturePayment` / `VoidPayment` / `RefundPayment`
- `FraudReview` (if applicable)
- `ReturnOrder` (policy + tender + inventory adjustments)

Projections:

- order history (customer + CSR)
- daily sales summary (by store/register/shift)

Integrations:

- payment gateways + EFT terminals
- fraud/risk service (optional early)
- tax engine (optional early; can start as internal rules)

### 3) Inventory-to-deliver (Inventory & Fulfillment)

Retail meaning:

- inbound receiving, stock levels by location, allocation, pick/pack/ship, BOPIS/ship-from-store.

Golden paths:

- eCommerce fulfill: allocate from sourcing location → pick/pack → ship → confirm delivery.
- BOPIS: allocate to store → notify ready → pickup → complete.

Initiating UISurfaces:

- `commerce-admin` (warehouse/store ops UIs)
- `commerce-storefront` (customer choices)

Process workflows (sketch):

- `AllocateInventory`
- `FulfillOrder` (pick/pack/ship)
- `HandlePickup` (BOPIS)
- `ReconcileInventory` (cycle counts, adjustments)

Projections:

- ATP / availability (channel-scoped + location-aware)
- fulfillment KPIs (lag, SLA)

Integrations:

- WMS/3PL
- carrier label APIs
- warehouse devices/scanners (edge-local)

### 4) Source-to-pay (Procurement) (future/optional)

Retail meaning:

- purchase orders, vendor lifecycle, receiving, invoice/3-way match, supplier payment.

Golden paths:

- Create PO → receive goods → match invoice → approve → pay supplier.

Process workflows (sketch):

- `ApprovePurchaseOrder`
- `ThreeWayMatch`
- `PaySupplier`

Integrations:

- ERP/finance
- supplier EDI

### 5) Store operations (Store Ops)

Retail meaning:

- open/close store, register sessions, shifts, cash management, device health, offline behavior and sync health.

Golden paths:

- Open shift → sell → close shift → cash reconcile → end-of-day summary.

Initiating UISurfaces:

- `commerce-pos`
- `commerce-admin` (monitoring and exception handling)

Process workflows (sketch):

- `OpenShift` / `CloseShift`
- `CashReconcile`
- `DrainOfflineQueue` (edge mode)

Projections:

- store health (connectivity mode, last sync, backlog/lag)
- cash and sales summaries by shift/register

Integrations:

- hardware gateway (printers/scanners/cash drawer)
- EFT terminals/payment SDKs (often local)

## Degraded mode contract (edge/offline)

Define and publish an explicit capability matrix.

Connectivity modes:

- Cloud mode: WAN required.
- Store edge mode: WAN may be down; POS reaches edge services over LAN.
- Device offline mode (future): POS cannot reach edge; reduced capability + strict sync rules.

Minimum expectations (v1):

- Store edge mode MUST keep core selling operable for a store site sales channel (cart/pricing/receipt/shift), while clearly gating real-time-required operations.
- Offline payments MUST be treated as capability-limited and policy-driven; do not promise “all tenders work offline”.

## Compatibility + rollout policy (edge fleets)

Introduce an explicit compatibility contract:

- store site runs a bundle of component versions
- upgrades happen in a defined order (cloud domain → replication flows → edge domain/projection → POS UI)
- rollback is supported and observable

Deliverables:

- compatibility matrix (proto package majors + component versions)
- `RolloutStoreFleet(bundle, ring)` workflow with canary/observe/advance/rollback

## Reference analog (vendor serviceability constraints)

Commerce platforms impose real constraints on version compatibility and upgrade sequencing across “engine/runtime + storefront/POS + integrations”.
This backlog treats compatibility and rollout policy as first-class deliverables once edge fleets are introduced.
