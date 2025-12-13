# 523 Commerce — Storefront + POS + Edge (research-aligned)

This backlog defines “commerce” as a proto-first, primitive-first system aligned with:

- `520-primitives-stack-v2.md` (normative primitives stack)
- `509-proto-naming-conventions.md` (proto package + topic conventions)

It is also informed by convergent industry patterns (D365 / SAP Commerce / Shopify + OSS commerce):

- one headless engine serving many surfaces,
- `SalesChannel` / `Site` as a first-class axis (not just tenant),
- edge/offline as topology + sync product, not a toggle.

## External convergence patterns (vendors + OSS)

- **Headless engine serving many experiences**: one backend engine (APIs + business logic) serves admin, POS, and public storefront (plus future mobile/kiosk/partner portals).
- **Channel/Site is first-class**: catalog visibility, pricing, taxes, shipping rules, inventory sourcing, and payment methods are channel-scoped.
- **Edge/offline is explicit**: local persistence, sync correctness, conflict handling, observability, and an explicit capability matrix.

## SalesChannel (first-class axis)

`tenant_id` is insufficient for commerce. Introduce `sales_channel_id` (or `site_id`) as required context that flows through:

- Domain APIs + domain facts (pricing/promo eligibility, tax/shipping/payment eligibility, assortment),
- UISurface configuration (which channel(s) a UI serves),
- Projections (search/availability per channel),
- Integrations (payment/tax/shipping bindings per channel),
- Replication (downsync/upsync per store site channel).

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

- **Cloud-only**: all commerce primitives run in cloud; POS/storefront require WAN.
- **Edge (store LAN availability)**: a store site runs a subset of primitives on-prem; WAN can be down; POS keeps operating.
- **Device offline (future)**: POS continues even if it cannot reach the edge cluster (POS-local DB/runtime).

Edge is treated as topology + sync, not a new primitive kind.

## Decomposition (six primitives)

- `523-commerce-domain.md` — headless commerce bounded context + facts/intents + SalesChannel
- `523-commerce-uisurface.md` — `commerce-admin`, `commerce-pos`, `commerce-storefront` (thin UIs)
- `523-commerce-projection.md` — search/availability/order-history + edge “channel DB” + hostname resolution
- `523-commerce-integration.md` — payments/EFT, taxes/shipping, hardware gateway, replication flows, BYOD domains plumbing
- `523-commerce-process.md` — workflows: BYOD onboarding, replication control, rollouts/recovery; defines real-time required surface
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

