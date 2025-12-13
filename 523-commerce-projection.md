# 523 Commerce — Projection component

## Intent

Provide read-optimized views for commerce across channels and topologies.

Projection themes:

- cloud projections for storefront/admin reads (search, history, availability),
- edge “channel DB” projection for store LAN availability,
- hostname resolution projection for public routing.

## Key projections (v1 set)

### 1) Hostname resolution (storefront)

- `hostname -> {tenant_id, sales_channel_id, uisurface_ref, canonical_host, status}`

### 2) Catalog discovery/search (cloud)

Read model optimized for browse/search, partitioned/scoped by `sales_channel_id`.

### 3) Order history (cloud)

Customer and CSR views; read-optimized with pagination and filters.

### 4) Inventory availability (cloud and/or edge)

Expose availability by site/region with explicit staleness semantics; callers provide `sales_channel_id` and the projection defines sourcing rules.

### 5) Store site projection (“channel DB” analogue) (edge)

Minimal dataset required for POS selling in edge mode:

- product/pricing/promo snapshots
- tender configuration
- device/register/shift configuration
- user/role mappings relevant to store

This projection is inherently channel-scoped (store site channel).

## Inputs

Prefer facts/events ingestion per v2 Projection guidance (`520-primitives-stack-v2-research-projection.md`):

- consume `commerce.domain.facts.v1` (and `commerce.process.facts.v1` where needed)

Fallback/bridge:

- watch CRs for hostname mapping if those are the only source of truth in v1.

## Query API sketch (examples)

- `ResolveHostname(hostname)`
- `SearchProducts(sales_channel_id, query, filters)`
- `GetOrderHistory(customer_id, cursor)`
- `GetAvailability(sales_channel_id, sku, site_id)`
- `GetStoreSiteSnapshot(site_id, version)`

## Stack alignment (proto / generation / operator)

- Proto: query APIs + projected schema declarations (read-only); input bindings declared as facts/events or CDC.
- Generation: SDKs + ingestion stubs + schema/migrations + structural tests (determinism/idempotency).
- Operator: owns projection workloads, stores/references, migrations/backfills, HTTPRoute exposure, and health/lag conditions.

## Consistency requirements

- Hostname resolution must converge quickly on revoke (fail closed; deny by default on uncertainty).
- Edge store-site projection must expose `last_synced_at` and `checkpoint` markers for UX and diagnostics.

