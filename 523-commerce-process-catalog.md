# 523 Commerce — Process Catalog (Level 0/1)

This document adds the missing “retail value-stream / E2E process catalog” view for `523-commerce*`.

Goal: keep the primitive decomposition anchored in how retailers run commerce day-to-day (not just onboarding/infra).

## Level 0 value streams

0. Platform onboarding & tenant/channel provisioning
1. Plan-to-merchandise (plan/market)
2. Sell-to-serve / Order-to-cash (POS + eCommerce)
3. Inventory-to-deliver (inventory + fulfillment)
4. Source-to-pay (procurement)
5. Store operations (open/close, cash, devices, workforce)

Each value stream below maps to the six primitives and defines:

- golden path scenario(s),
- initiating UISurfaces,
- domain primitives involved,
- process workflows,
- projections required,
- integrations required,
- key facts/events (topics).

## Cross-cutting: identities and context

All flows assume:

- `tenant_id` (tenant boundary)
- `sales_channel_id` (site/channel boundary; required for most commerce decisions)
- `store_site_id` (edge deployment unit; typically aligns to a store channel)

## 0) Platform onboarding & tenant/channel provisioning

Golden path:

- Create tenant → create sales channel(s) → connect storefront domain → go live.

Initiating UISurface:

- `commerce-admin` (internal)

Process workflows:

- BYOD: `VerifyDomainClaim`, `ProvisionDomainMapping`, `RevokeDomainMapping`
- Store site: `ProvisionStoreSite`, `RolloutStoreFleet`

Projections:

- Hostname resolution: `hostname -> {tenant_id, sales_channel_id, uisurface_ref, status}`

Integrations:

- Gateway API + cert-manager + DNS verification

Facts/events:

- `commerce.process.facts.v1`: `TenantProvisioned`, `SalesChannelProvisioned`, `DomainMappingReady`, `StoreSiteReady`

## 1) Plan-to-merchandise (Plan & Market)

Retail meaning:

- define products, assortments, price books, promotions, and content; publish to channels with effective dates.

Golden paths:

- Create product → publish to channel assortment → publish price book → activate promotion.

Initiating UISurface:

- `commerce-admin`

Domain primitives (suggested split):

- `commerce-channels` (SalesChannel / site config)
- `commerce-catalog` (products, variants, attributes, assortments)
- `commerce-pricing` (price books, price rules)
- `commerce-promotions` (discounts, coupons, eligibility)

Process workflows:

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

Facts/events:

- `commerce.domain.facts.v1`: `ProductPublished`, `AssortmentActivated`, `PriceBookActivated`, `PromotionActivated`

## 2) Sell-to-serve / Order-to-cash (POS + eCommerce)

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

Domain primitives (suggested split):

- `commerce-orders` (carts, orders, returns)
- `commerce-payments` (payment intent/state machine + idempotency)
- `commerce-pricing` (price/promo evaluation)
- `commerce-customers` (profiles; optional early)
- `commerce-storeops` (register/shift/receipt context)

Process workflows:

- `Checkout` (reserve/allocate → authorize → create order → emit facts)
- `CapturePayment` / `VoidPayment` / `RefundPayment`
- `FraudReview` (if applicable)
- `ReturnOrder` (policy + tender + inventory adjustments)

Projections:

- order history (customer + CSR)
- daily sales summary (by store/register/shift)
- customer 360 (optional)

Integrations:

- payment gateways + EFT terminals
- fraud/risk service (optional early)
- tax engine (optional early; can start as internal rules)

Facts/events:

- `commerce.domain.facts.v1`: `CartCheckedOut`, `OrderCreated`, `PaymentAuthorized`, `PaymentCaptured`, `ReceiptIssued`, `OrderRefunded`

## 3) Inventory-to-deliver (Inventory & Fulfillment)

Retail meaning:

- inbound receiving, stock levels by location, allocation, pick/pack/ship, BOPIS/ship-from-store.

Golden paths:

- eCommerce fulfill: allocate from sourcing location → pick/pack → ship → confirm delivery.
- BOPIS: allocate to store → notify ready → pickup → complete.

Initiating UISurfaces:

- `commerce-admin` (warehouse/store ops UIs)
- `commerce-storefront` (customer choices)

Domain primitives (suggested split):

- `commerce-inventory` (stock, locations, adjustments)
- `commerce-fulfillment` (shipments, allocations, pick/pack/ship lifecycle)
- `commerce-orders` (order fulfillment state)

Process workflows:

- `AllocateInventory`
- `FulfillOrder` (pick/pack/ship)
- `HandlePickup` (BOPIS)
- `ReconcileInventory` (cycle counts, adjustments)

Projections:

- ATP / availability (channel-scoped + location-aware)
- promised delivery dates (optional)
- fulfillment KPIs (lag, SLA)

Integrations:

- WMS/3PL
- carrier label APIs
- warehouse devices/scanners (edge-local)

Facts/events:

- `commerce.domain.facts.v1`: `InventoryAdjusted`, `ShipmentCreated`, `ShipmentShipped`, `DeliveryConfirmed`, `PickupCompleted`

## 4) Source-to-pay (Procurement)

Retail meaning:

- purchase orders, vendor lifecycle, receiving, invoice/3-way match, supplier payment.

Golden paths:

- Create PO → receive goods → match invoice → approve → pay supplier.

Initiating UISurface:

- `commerce-admin` (or a future procurement UISurface)

Domain primitives (future/optional):

- `procurement-vendors`
- `procurement-purchaseorders`
- `procurement-receiving`
- `procurement-accounts-payable`

Process workflows:

- `ApprovePurchaseOrder`
- `ThreeWayMatch`
- `PaySupplier`

Integrations:

- ERP/finance
- supplier EDI
- invoice OCR/AP automation (optional)

Facts/events:

- `procurement.domain.facts.v1`: `PurchaseOrderApproved`, `GoodsReceived`, `InvoiceMatched`, `SupplierPaid`

## 5) Store operations (Store Ops)

Retail meaning:

- open/close store, register sessions, shifts, cash management, device health, offline behavior and sync health.

Golden paths:

- Open shift → sell → close shift → cash reconcile → end-of-day summary.

Initiating UISurface:

- `commerce-pos`
- `commerce-admin` (monitoring and exception handling)

Domain primitives:

- `commerce-storeops` (shifts/registers/receipts/cash mgmt)

Process workflows:

- `OpenShift` / `CloseShift`
- `CashReconcile`
- `DrainOfflineQueue` (edge mode)

Projections:

- store health (connectivity mode, last sync, backlog/lag)
- cash and sales summaries by shift/register

Integrations:

- hardware gateway (printers/scanners/cash drawer)
- EFT terminals/payment SDKs (often local)

Facts/events:

- `commerce.domain.facts.v1`: `ShiftOpened`, `ShiftClosed`, `CashReconciled`, `OfflineQueueDrained`

## Degraded mode contract (edge/offline)

Define and publish an explicit capability matrix.

Connectivity modes:

- Cloud-only: WAN required.
- Edge mode: WAN may be down; POS reaches edge services over LAN.
- Device offline (future): POS cannot reach edge; reduced capability + strict sync rules.

Minimum expectations (v1):

- Edge mode MUST keep core selling operable for a store site channel (cart/pricing/receipt/shift), while clearly gating real-time-required operations.
- Offline payments MUST be treated as capability-limited and policy-driven; do not promise “all tenders work offline”.

