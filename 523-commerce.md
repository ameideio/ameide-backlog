# 523 Commerce — Storefront + POS + Edge (research-aligned)

This backlog defines Commerce using the `470`/`529` ArchiMate language anchor, while staying precise about Ameide’s primitives + EDA + operators.

**Status:** Implemented (v1 primitives scaffolded: BYOD storefront domains + hostname resolution; other subdomains TBD)

**Implementation (repo):**
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/commerce/`
- Domain: `primitives/domain/commerce`
- Projection: `primitives/projection/commerce`
- Process: `primitives/process/commerce`
- Integration: `primitives/integration/commerce`
- UISurfaces: `primitives/uisurface/commerce-admin`, `primitives/uisurface/commerce-pos`, `primitives/uisurface/commerce-storefront`
- Agent: `primitives/agent/commerce-assistant`

## Implementation progress (current)

Delivered (v1 BYOD storefront domains slice):

- [x] Proto contracts exist for Domain/Process/Projection/Integration/Agent and are buf-lint/breaking clean.
- [x] Domain supports BYOD hostname claim → verify/fail → mapping request → activate/fail → revoke (with inbox/outbox tables + Flyway migration images).
- [x] Projection supports hostname resolution + claim/mapping reads (currently bridge mode: reads Domain tables).
- [x] Process has worker/ingress scaffold + required shape files; placeholder BYOD workflow scaffold exists.
- [x] Integration has hardware “ping” + payments port stub; tests/guardrails are in place.
- [x] UISurfaces (`commerce-admin`, `commerce-pos`, `commerce-storefront`) and the Agent (`commerce-assistant`) are scaffolded with tests + non-root containers.

Repo guardrails:

- [x] `ameide primitive verify --mode repo` passes for the commerce primitives when GitOps manifests are present.

Build-out checklist (next implementation slices):

- [ ] Decide/lock v1 scope profile (go-live foundation only vs web O2C MVP vs omnichannel MVP with store-edge + replication).
- [ ] Switch hostname projection from bridge reads → facts ingestion + materialized read model.
- [ ] Implement BYOD onboarding workflow end-to-end (Process orchestrates DNS/cert/gateway steps via Integration adapters).
- [ ] Add the “ownership table” for Domain vs Process workflows and enforce it in contracts.
- [ ] Expand Domain subdomains beyond BYOD (SalesChannel/Site, Orders/OMS, Inventory/ATP, StoreOps) based on the v1 scope decision.

## Layer header (Strategy + Business, with Application realization)

- **Primary ArchiMate layer(s):** Strategy + Business (Capability, Value Streams, Business Processes).
- **Primary element types used:** Capability, Value Stream, Business Process (plus Application Services/Interfaces/Events realized by Application Components).
- **Out-of-scope layers:** Technology details (see `473-ameide-technology.md`), except for topology constraints called out explicitly.

Language note: “facts = Application Events” and “intents/commands = requests to invoke Application Services” per `470-ameide-vision.md` §0.2 and `529-archimate-alignment-470plus.md`.

This backlog defines Commerce as a proto-first, primitive-first system aligned with:

- `520-primitives-stack-v2.md` (normative primitives stack)
- `509-proto-naming-conventions.md` (proto package + topic conventions)
- `496-eda-principles.md` (commands vs facts, outbox, idempotent consumers, tenant isolation)
- `524-transformation-capability-decomposition.md` (repeatable method: capability/value streams → application contracts → primitives)
- `530-ameide-capability-design-worksheet.md` (layered worksheet: Motivation → Strategy → Business → Application → Technology → Implementation & Migration)

It is also informed by convergent industry patterns (D365 / SAP Commerce / Shopify + OSS commerce):

- one headless engine serving many surfaces,
- `Site` / `SalesChannel` / `StockLocation` as first-class axes (not just tenant),
- edge/offline as topology + sync product, not a toggle.

## Motivation: constraints (non-negotiables)

- Tenant isolation and traceability metadata on all intents/facts/queries (see `470-ameide-vision.md` §0.2 and `496-eda-principles.md`).
- Single-writer Domain primitives emit domain facts (Application Events) via transactional outbox; reads come from projections/read models.
- BYOD hostnames are globally unique and must be explicitly claimed/verified/revoked (fail closed on uncertainty).
- Edge/offline is an explicit topology mode with explicit replication + degraded-mode UX (not a toggle).

## Strategy: Commerce capability and value streams

**Commerce** is a Strategy-layer **Capability**: the ability for a tenant to sell across channels (storefront + POS) and operate reliably across topologies (cloud/edge/offline).

Value streams (Level 0/1, non-exhaustive):

- **Storefront go-live**: configure site/channel → connect domains → verify → activate → monitor/revoke.
- **Order-to-cash (O2C)**: browse → cart → checkout → pay → fulfill → returns/refunds.
- **Inventory-to-deliver**: publish availability → allocate/sourcing → fulfill → adjust.
- **Store operations**: register/shift → sell → reconcile → close.
- **Edge operations**: downsync masterdata → operate → upsync transactions → recover/backfill.

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

## Technology: required services (sketch)

Commerce Application Components (primitives) rely on Technology Services such as:

- Kubernetes + GitOps + operators
- Gateway API (public routing), plus cert-manager (ACME issuance) for BYOD domains
- Postgres/CNPG for Domain state + outbox
- Broker (NATS/Kafka) for facts topic families
- Temporal for Process primitive orchestration

## Application realization (six primitives)

- `523-commerce-domain.md` — headless commerce bounded context + facts/intents + identity model (`site/channel/location`)
- `523-commerce-uisurface.md` — `commerce-admin`, `commerce-pos`, `commerce-storefront` (thin UIs)
- `523-commerce-projection.md` — search/availability/order-history + hostname resolution + edge caches/read models (read-only)
- `523-commerce-integration.md` — payments/EFT, taxes/shipping, hardware gateway, replication flows, BYOD domains plumbing
- `523-commerce-process.md` — application-layer orchestration workflows that implement business processes/value streams under eventual consistency
- `523-commerce-agent.md` — optional setup/support assistants (read-only diagnostics + guided steps)
- `523-commerce-proto.md` — proto proposal for primitive-to-primitive communication

## Application Interfaces for Agents (MCP)

Commerce should declare an explicit MCP surface for agentic access (external devtools + portal chat), implemented as a capability-owned **Integration primitive** (protocol adapter), not as an Agent feature.

References:
- MCP adapter pattern: `backlog/534-mcp-protocol-adapter.md`
- Read optimizations: `backlog/535-mcp-read-optimizations.md`
- Write optimizations: `backlog/536-mcp-write-optimizations.md`

**Published interface (target)**
- MCP tools: yes
- MCP resources: yes (read-only)
- MCP prompts: optional; only if versioned/promotable

**Integration primitive**
- `commerce-mcp-adapter` translates MCP (stdio + Streamable HTTP) into proto-first Application Services.
- Tool schemas/manifests are generated from proto annotations (default deny; explicit allowlist).

Starter tool map (illustrative; authoritative source is generated from proto once defined):

| Tool | Kind | Application Service (canonical) | Primitive | Approval | Edge constraints / notes |
|------|------|----------------------------------|----------|----------|--------------------------|
| `commerce.listProducts` | query | `CatalogQueryService.ListProducts` | Projection | no | interactive-safe; paginated |
| `commerce.getOrder` | query | `OrderQueryService.GetOrder` | Projection | no | may be eventually consistent |
| `commerce.searchInventory` | query | `InventoryQueryService.SearchAvailability` | Projection | no | topology-aware (cloud/edge) |
| `commerce.placeOrder` | command | `OrderWriteService.PlaceOrder` | Domain | yes | must emit domain facts; idempotent |
| `commerce.cancelOrder` | command | `OrderWriteService.CancelOrder` | Domain | yes | compensation semantics explicit |
| `commerce.claimDomain` | command | `SiteWriteService.ClaimDomain` | Domain/Process | yes | BYOD workflow evidence required |

Starter resources (illustrative):

| Resource URI pattern | Backing query service | Primitive |
|----------------------|-----------------------|----------|
| `commerce://product/{id}` | `CatalogQueryService.GetProduct` | Projection |
| `commerce://order/{id}` | `OrderQueryService.GetOrder` | Projection |
| `commerce://site/{id}` | `SiteQueryService.GetSite` | Projection |

Edge constraints (Commerce):

- Writes require human approval by default when invoked by agents; exceptions must be explicit and auditable.
- Evidence/audit is produced as domain facts (after persistence) and surfaced via projection-backed timelines; BYOD domain operations require observable evidence (DNS checks, verification, ACME issuance, revoke/transfer).
- Offline/edge modes must be reflected in failure modes and responses (stale reads, degraded writes, queued intents).

## “Real-time required” operations (v1 list; keep small)

Default is async replication. These operations are explicitly not safe to rely on eventual consistency without a deliberate design:

- gift card balance/redeem (if external)
- loyalty redemption/earn (if external)
- customer create/update (if centrally governed)
- cross-store returns/exchanges
- inventory adjustments that affect other sites
- fraud checks / risk holds (provider-dependent)

## Implementation & Migration: phased delivery (suggested)

1. Storefront public domains (platform default + BYOD) with Shopify-grade onboarding UX.
2. Cloud-only commerce Domain + Projections (admin + storefront + POS).
3. Edge store-site support (LAN availability) with replication health UX.
4. Device-offline POS (separate phase; explicit reduced capability matrix).

## BYOD domains (product-level expectations)

Treat BYOD onboarding as a first-class workflow, not a support runbook:

- surface precise failure reasons (DNS mismatch, TXT mismatch, propagation pending, CAA blocks, proxy/CDN interference, ACME rate limits)
- implement retry/backoff; do not busy-loop DNS or ACME
- provide explicit revoke/transfer semantics (audit + optional cooldown)

## EDA articulation (496)

Commerce should be explainable as an event-driven platform, not just a set of primitives:

- **Commands/intents** initiate change (imperative business verbs).
- **Domain facts** are immutable sources of truth (past tense) emitted by single-writer domains via transactional outbox.
- **Process facts** describe orchestration progress; Temporal implements sagas (no distributed transactions).
- **Projections** are idempotent consumers building read models (UPSERT/inbox), and lag/health are observable SLOs.
- **Integrations** are boundary adapters; inbound is at-least-once + idempotent; outcomes are surfaced as facts/conditions.
- **Tenant isolation + traceability metadata** are required on every message.

The concrete message families, envelopes, and topic naming are proposed in `523-commerce-proto.md`, including CloudEvents/W3C Trace Context alignment for cross-broker/HTTP interoperability.

## Clarification requests (next steps)

To expand Commerce beyond BYOD domains without redesign churn, confirm these v1 choices:

- [ ] v1 profile: go-live foundation only vs web O2C MVP vs omnichannel MVP (POS + store-edge + replication).
- [ ] Topology: cloud-only vs cloud + store-edge (LAN mode) in v1 (device-offline deferred unless explicitly in scope).
- [ ] Payments posture: provider choice vs mock; whether offline tenders are in scope.
- [ ] Broker target: Kafka/Strimzi now vs “outbox-only” until broker wiring is standardized.
- [ ] PIM/CMS posture: integration-only vs minimal embedded product content for v1.
- [ ] Scope owners: which subdomains are first-class v1 (Orders/OMS, Inventory/ATP, StoreOps, Customer/Loyalty, GiftCards).

Also clarify the platform posture for generated SDK stubs:

- [ ] `packages/ameide_sdk_*` generated proto trees are build artifacts; confirm whether CI/devcontainer generation is an expected precondition for `ameide primitive verify` gates.
