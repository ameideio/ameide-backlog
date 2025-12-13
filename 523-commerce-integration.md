# 523 Commerce — Integration component

## Intent

Define the external seams that make commerce work across channels and topologies:

- storefront public domain routing/TLS/DNS (multi-tenant, BYOD safe),
- payments/EFT provider connectivity,
- taxes/shipping carriers (channel-scoped),
- retail hardware/peripheral control (local gateway),
- cloud ↔ edge replication flows (async default + small real-time surface).

Per v2 rules, proto declares ports/contracts; endpoints and secrets are runtime-bound (see `520-primitives-stack-v2.md` and `520-primitives-stack-v2-research-integration.md`).

## Stack alignment (proto / generation / operator)

- Proto: stable external seams (`*IntegrationService` ports) with schemas; no URLs/credentials in proto.
- Generation: flow artifacts + parameter templates + schema descriptors + harness/tests (NiFi-default).
- Operator: reconciles Integration CRDs into NiFiKop/NiFi resources, manages Parameter Contexts, and surfaces conditions (backpressure/bulletins/ready).

## Integration buckets (keep them separate)

Two common “retail integration classes” have very different constraints:

- **Cloud enterprise integrations**: ERP/finance, PIM/MDM, WMS/3PL, CRM, EDI.
- **Store/edge integrations**: payments/EFT terminals, hardware peripherals, replication/sync.

Keep both as Integration primitives, but expect different operator/runtime contracts and different failure/latency assumptions.

### A) Storefront public domains (routing/TLS/DNS)

Goal: safely expose `commerce-storefront` on:

1. platform default domain (`*.storefront.platform.tld`)
2. tenant BYOD domains (`shop.tenant.com`) after ownership verification

Key integrations:

- Gateway API (`Gateway`, `HTTPRoute`, `ReferenceGrant`)
- cert-manager (ACME issuance into `Secret`s; renewals)
- DNS (manual CNAME + TXT verification; ExternalDNS only for zones you control)

Control plane resources (platform-owned; names placeholders):

- `UISurfaceDomainClaim`: reserves hostname + tracks verification token/status (exact hostname only in v1)
- `UISurfaceDomainMapping`: binds verified hostname → UISurface (unique hostname invariant)

Recommended routing start:

- one shared, platform-owned public `Gateway` (stable public address)
- wildcard listener + wildcard cert for default domain
- per-custom-domain listeners for BYOD hostnames with per-tenant cert `Secret`s + `ReferenceGrant`

Operational requirements (Shopify-grade UX):

- Claim/Mapping workflow MUST surface precise failure reasons (DNS mismatch, propagation delay, CAA blocks, proxy/CDN interference, multiple/incorrect A records, etc.).
- Retry/backoff semantics MUST be explicit; do not busy-loop DNS or ACME.
- Revocation and transfer rules MUST be explicit (cooldown before a hostname can move between tenants).

### B) Payments + EFT (providers and terminals)

Principles:

- Domain owns the payment intent/state machine + idempotency.
- Integrations own provider connectors, terminal SDKs, and fraud hooks.

Ports (sketch):

- authorize/capture/refund/void
- webhooks ingestion (validated + idempotent)
- tokenization (where applicable)

Channel sensitivity:

- Payment methods are channel-scoped (available tenders vary by `sales_channel_id`); bind explicitly per channel.
- Taxes and shipping carrier integrations are also commonly channel-scoped; bind per channel rather than globally per tenant.
- Offline payment acceptance is capability-limited and policy-driven; do not assume “queue and reconcile later” works for every provider/terminal.

### C) Hardware gateway (“Hardware Station” analogue)

Treat peripherals as a local service:

- LAN-local gateway drives printers/scanners/cash drawers and may host payment SDKs where required
- POS talks to it via a stable local API (pairing/auth explicit)

### D) Replication (cloud ↔ edge) (CDX analogue)

Default: async downsync + upsync.

Downsync candidates:

- catalog/pricing/promos (channel-scoped)
- tender configuration (channel-scoped)
- users/roles relevant to store
- device/register configuration

Upsync candidates:

- orders/payments
- shifts/receipts
- inventory adjustments (where allowed)

Implementation default:

- NiFi-backed Integration primitive with Parameter Contexts for secrets and conditions for backpressure/health.

Channel sensitivity:

- replication bindings and payload selection are channel-scoped (store site channel vs eCommerce channel can have different datasets and frequencies).
