# 523 Commerce — Storefront + POS + Edge (research-aligned)

This backlog defines “commerce” as a proto-first, primitive-first system aligned with:

- `520-primitives-stack-v2.md` (normative primitives stack)
- `509-proto-naming-conventions.md` (proto package + topic conventions)

It is also informed by convergent industry patterns (D365 / SAP Commerce / Shopify + OSS commerce):

- one headless engine serving many surfaces,
- `Site` / `SalesChannel` / `StockLocation` as first-class axes (not just tenant),
- edge/offline as topology + sync product, not a toggle.

## External convergence patterns (vendors + OSS)

- **Headless engine serving many experiences**: one backend engine (APIs + business logic) serves admin, POS, and public storefront (plus future mobile/kiosk/partner portals).
- **Site/Channel/Location are first-class**: web presence (domains/branding), selling context (currency/tax/pricing/promos), and physical nodes (inventory/fulfillment) are distinct axes.
- **Edge/offline is explicit**: local persistence, sync correctness, conflict handling, observability, and an explicit capability matrix.

## Commerce identity & scope model (lock the nouns)

To avoid “store/channel/site/location” drift, standardize a canonical identity model used by every primitive:

- `tenant_id`: security/billing boundary.
- `site_id` (or `storefront_id`): web presence boundary (brand/base URLs/domains → routing to UISurfaces).
- `sales_channel_id`: selling context (currency, tax regime, price lists, assortment, payment/shipping methods).
- `stock_location_id`: physical node for inventory/fulfillment (store, warehouse, dark store).
- `store_site_id`: edge deployment unit (store cluster) that typically aligns to a store sales channel + stock location.

Rules of thumb:

- Storefront request resolution is `Host -> {tenant_id, site_id} -> sales_channel_id`.
- Most commerce “decision” APIs require `tenant_id + sales_channel_id` (and usually `currency` and `effective_at`).
- Inventory/fulfillment decisions also require `stock_location_id` (or return results keyed by location).

## Research map (D365 + SAP) → Ameide mapping

### D365 Commerce

Read order: architecture → CRT → CSU/CSU Core → headless integration → module library (only if adopting their site framework).

Mapping:

- CRT ≈ Commerce Domain runtime contract (headless business engine).
- CSU/Core ≈ deployment topology for Domain/Projection (cloud or edge), not a new primitive kind.
- Headless integration ≈ stable Domain APIs consumed by many UISurfaces.

### SAP Commerce Cloud

Read order: platform overview → OCC → Integration API module → composable storefront (Spartacus).

Mapping:

- OCC ≈ Domain public API surface (stable contract for storefront/POS/admin).
- Integration API module ≈ Integration + Projection seams for enterprise data exchange.
- Spartacus ≈ storefront implementation option, not a platform contract.

## Topology matrix (explicit)

- **Cloud mode**: commerce runs in cloud; POS/storefront require WAN.
- **Store edge mode (LAN availability)**: a store site runs a subset of primitives on-prem; WAN can be down; POS keeps operating over LAN.
- **Device offline mode (future)**: POS continues even if it cannot reach the edge cluster (POS-local DB/runtime/queue).

Edge is treated as topology + sync, not a new primitive kind.

## Decomposition (six primitives)

- `523-commerce-domain.md` — headless commerce bounded context + facts/intents + identity model (`site/channel/location`)
- `523-commerce-uisurface.md` — `commerce-admin`, `commerce-pos`, `commerce-storefront` (thin UIs)
- `523-commerce-projection.md` — search/availability/order-history + hostname resolution + edge caches/read models (read-only)
- `523-commerce-integration.md` — payments/EFT, taxes/shipping, hardware gateway, replication flows, BYOD domains plumbing
- `523-commerce-process.md` — retail value streams + workflows (BYOD, order-to-cash, inventory-to-deliver, store ops), plus rollouts/recovery
- `523-commerce-agent.md` — optional setup/support assistants (read-only diagnostics + guided steps)

## “Real-time required” operations (v1 list; keep small)

Default is async replication. These operations are explicitly not safe to rely on eventual consistency without a deliberate design:

- gift card balance/redeem (if external)
- loyalty redemption/earn (if external)
- customer create/update (if centrally governed)
- cross-store returns/exchanges
- inventory adjustments that affect other sites
- fraud checks / risk holds (provider-dependent)

## Phased delivery (suggested)

1. Storefront public domains (platform default + BYOD) with Shopify-grade onboarding UX.
2. Cloud-only commerce Domain + Projections (admin + storefront + POS).
3. Edge store-site support (LAN availability) with replication health UX.
4. Device-offline POS (separate phase; explicit reduced capability matrix).

## BYOD domains (product-level expectations)

Treat BYOD onboarding as a first-class workflow, not a support runbook:

- surface precise failure reasons (DNS mismatch, TXT mismatch, propagation pending, CAA blocks, proxy/CDN interference, ACME rate limits)
- implement retry/backoff; do not busy-loop DNS or ACME
- provide explicit revoke/transfer semantics (audit + optional cooldown)
