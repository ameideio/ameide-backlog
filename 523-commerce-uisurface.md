# 523 Commerce — UISurface component

## Intent

Define the starting set of commerce experiences and keep them thin.

UISurfaces:

1. `commerce-admin` (internal HQ): setup/config + operational visibility (sync/health/status).
2. `commerce-pos` (store staff): selling UX; must tolerate WAN issues via edge topology.
3. `commerce-storefront` (public): multi-tenant hostname routing and BYOD domains.

This document covers UISurface runtime/operator boundaries. Payments/hardware/replication and BYOD plumbing live in `523-commerce-integration.md` and `523-commerce-process.md`.

For the value-stream view (Plan-to-Merchandise, Order-to-Cash, Inventory-to-Deliver, Store Ops), see `523-commerce-process.md`.

## Headless engine, many experiences

The architecture assumes additional experiences over time (mobile, kiosks, partner portals) without changing the core domain model. Experiences differ in UX and topology constraints, not in business rules.

## Commerce context (required): tenant/site/channel/location

All commerce UISurfaces MUST operate within a consistent identity/scope model:

- `tenant_id`: security boundary
- `site_id`: web presence boundary (domains/base URLs)
- `sales_channel_id`: selling context (currency/tax/pricing/promos/eligible tenders/shipping)
- `stock_location_id`: physical node for inventory/fulfillment
- `store_site_id`: edge deployment unit (store cluster), typically aligned to a store channel + stock location

Practical expectations:

- `commerce-storefront`: resolves `Host -> {tenant_id, site_id}` then derives `sales_channel_id` from site configuration (and/or request locale/currency); inventory queries may be per `stock_location_id` or “best sourced for channel”.
- `commerce-pos`: pinned to `store_site_id` and operates in a store `sales_channel_id` + `stock_location_id` context.
- `commerce-admin`: configures sites, channels, and stock locations and can switch context explicitly.

## Topology matrix (explicit)

- Cloud mode: POS/storefront require WAN to reach cloud domains.
- Store edge mode (LAN): POS prefers local edge endpoints; WAN may be down.
- Device offline mode (future): POS continues even if edge is unreachable.

## Runtime responsibilities (all commerce UISurfaces)

- MUST be presentation + orchestration only (no core commerce business rules).
- MUST call Domain APIs and/or Projection query APIs via generated SDKs (per `520-primitives-stack-v2.md`).
- SHOULD degrade gracefully under partial connectivity (especially `commerce-pos`).

## `commerce-storefront` specifics (public domains)

- MUST resolve tenant/site context from `Host` header and enforce a strict host allowlist (unmapped hosts 404).
- MUST support platform default domains and BYOD custom domains, but it MUST NOT self-claim domains.
- SHOULD implement safe caching (static aggressively; dynamic carefully).

## `commerce-pos` specifics (edge-first)

- MUST treat store site as the primary availability boundary in store edge mode.
- SHOULD prefer local edge endpoints for pricing/cart/checkout and store-ops.
- MUST expose clear UX for real-time-required operations when WAN is unavailable.

### Offline/edge capability matrix (v1; explicit)

In store edge mode (WAN down, LAN up), `commerce-pos` SHOULD support:

- cart/build basket
- price/promo evaluation from local ruleset snapshots
- tender acceptance where allowed by policy (offline payments are capability-limited)
- receipt issuance
- shift/register operations

In store edge mode, these operations are typically real-time-required and MUST provide UX that requires WAN (or a local authoritative system):

- cross-store returns/exchanges
- loyalty/gift card operations (if external)
- fraud/risk checks (provider-dependent)

In device-offline mode (future), define an explicit reduced capability set and sync behavior; do not assume “queue everything”.

## `commerce-admin` specifics (HQ)

- SHOULD be the primary UI for catalog/pricing/promo setup and store/register/device provisioning.
- MUST surface replication health/status per store site (lag, last sync, failures).

## Operator responsibilities (UISurface operator)

- MUST reconcile `Deployment` + `Service` + `HTTPRoute`.
- MUST keep hostnames constrained by platform policy; do not allow tenants to create arbitrary hostnames on shared gateways.
- SHOULD remain dumb about domain verification/certs (Integration/Process responsibility).

## Suggested proto shape (behavior plane)

Keep UISurface protos minimal and UI-facing:

- package: `ameide_core_proto.commerce.uisurface.v1` (or fold into `...commerce.v1` if small).
- typed request/response models only where needed for interoperability; avoid encoding UI policy/routing logic in proto.

## Stack alignment (proto / generation / operator)

- Proto: UI-facing contracts and view models only; no per-environment bindings.
- Generation: `buf generate` produces SDKs + UI skeleton wiring where applicable.
- Operator: reconciles web workloads and HTTPRoutes; BYOD domain verification/certs are not owned by the UISurface operator.
