# 523 Commerce — UISurface component

**Status:** Scaffolded (minimal Node placeholders + tests; real UX pending)

Implementation (repo): `primitives/uisurface/commerce-admin`, `primitives/uisurface/commerce-pos`, `primitives/uisurface/commerce-storefront`

## Layer header (Application: experiences)

- **Primary ArchiMate layer(s):** Application (UISurfaces are Application Components that present experiences).
- **Primary element types used:** Application Component (UISurface primitives), Application Services (commands/queries), Application Events (facts that drive correctness), Data Objects (view models).
- **Out-of-scope layers:** Strategy/Business definitions (see `523-commerce.md`); Technology routing/TLS implementation (Integration/Process responsibility + `473-ameide-technology.md`).

## Intent

Define the starting set of commerce experiences and keep them thin.

UISurfaces:

1. `commerce-admin` (internal HQ): setup/config + operational visibility (sync/health/status).
2. `commerce-pos` (store staff): selling UX; must tolerate WAN issues via edge topology.
3. `commerce-storefront` (public): multi-tenant hostname routing and BYOD domains.

This document covers UISurface runtime/operator boundaries. Payments/hardware/replication and BYOD plumbing live in `523-commerce-integration.md` and `523-commerce-process.md`.

## Implementation progress (current)

Implemented (scaffold):

- [x] Minimal Node servers for `commerce-admin`, `commerce-pos`, and `commerce-storefront`.
- [x] Guardrails in place (tests, non-root containers, `ameide primitive verify` passes).

Not yet implemented:

- [ ] Real UX (auth, navigation, domain onboarding flows, health/sync dashboards).
- [ ] Storefront host resolution integration (call Projection `ResolveHostname` and fail closed).
- [ ] POS degraded-mode UX contract enforcement (if store-edge is in v1 scope).

Build-out checklist (UISurfaces v1):

- [ ] Define the auth/session model and “current context” selector UX (tenant/site/channel/location).
- [ ] Implement `commerce-admin` BYOD domain onboarding flow (claim, verify instructions, status, revoke).
- [ ] Implement `commerce-storefront` strict host allowlist + “unmapped host” behavior (404) and caching rules.
- [ ] If store-edge is in scope: implement `commerce-pos` degraded-mode capability matrix and WAN-down gating UX.

## Clarification requests (next steps)

Confirm/decide:

- [ ] Auth/identity approach for admin/POS (Keycloak? session model?) and tenant/site/channel context selection UX.
- [ ] Whether `commerce-storefront` is truly headless (API-only) in v1 or also serves rendered content.
- [ ] Minimal operational dashboards required in `commerce-admin` (domain onboarding, cert status, replication health).

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

## EDA responsibilities (496-native)

UISurfaces initiate change via commands/intents and render from read models:

- Initiate change via command RPCs and/or `commerce.domain.intents.v1`; do not mutate state directly.
- Render state via query APIs and projections; correctness is driven by domain/process facts, not UI push.
- UI “real-time” streams (if any) are best-effort UX signals, not replayable sources of truth (see `496-eda-principles.md` Principle 7).

See `523-commerce-proto.md` for message families and envelopes.

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
