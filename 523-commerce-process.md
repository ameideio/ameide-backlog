# 523 Commerce — Process component

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

## Real-time required surface (v1)

Default is async replication. The system MUST label and handle “real-time required” operations explicitly:

- gift cards (if external)
- loyalty redemption/earn (if external)
- customer create/update (if centrally governed)
- cross-store returns/exchanges
- inventory adjustments that affect other sites
- fraud checks / risk holds (provider-dependent)

## Retail value streams (Level 0/1)

This section anchors Process design in retailer day-to-day operations (value streams), not just onboarding/infra.

### Cross-cutting context

All workflows assume:

- `tenant_id` (tenant boundary)
- `sales_channel_id` (site/channel boundary; required for most commerce decisions)
- `store_site_id` (edge deployment unit; typically aligns to a store channel)

### 0) Platform onboarding & tenant/channel provisioning

Golden path:

- Create tenant → create sales channel(s) → connect storefront domain → go live.

Initiating UISurface:

- `commerce-admin`

Process workflows (sketch):

- BYOD: `VerifyDomainClaim`, `ProvisionDomainMapping`, `RevokeDomainMapping`
- Store site: `ProvisionStoreSite`, `RolloutStoreFleet`

Projections:

- Hostname resolution: `hostname -> {tenant_id, sales_channel_id, uisurface_ref, status}`

Integrations:

- Gateway API + cert-manager + DNS verification

Facts/events:

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

- a store site MUST declare its `sales_channel_id` and replication bindings are derived from that channel.

### C) Replication control

- `RunDownsync(site_id)`
- `RunUpsync(site_id)`
- `RebuildStoreSiteProjection(site_id)` (replay/backfill)

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

- Cloud-only: WAN required.
- Edge mode: WAN may be down; POS reaches edge services over LAN.
- Device offline (future): POS cannot reach edge; reduced capability + strict sync rules.

Minimum expectations (v1):

- Edge mode MUST keep core selling operable for a store site channel (cart/pricing/receipt/shift), while clearly gating real-time-required operations.
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
