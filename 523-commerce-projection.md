# 523 Commerce — Projection component

## Intent

Provide read-optimized views for commerce across channels and topologies.

Projection themes:

- cloud projections for storefront/admin reads (search, history, availability),
- hostname resolution projection for public routing,
- edge caches/read models that support store-edge operation.

Important: the store-edge “operational store” (transactional writes for POS selling) should be treated as a Domain-owned write model deployed to edge (D365 “channel DB” analogue). Projections remain read-only/derived.

## Key projections (v1 set)

### 1) Hostname resolution (storefront)

- `hostname -> {tenant_id, site_id, default_sales_channel_id, uisurface_ref, canonical_host, status}`

### 2) Catalog discovery/search (cloud)

Read model optimized for browse/search, partitioned/scoped by `sales_channel_id`.

### 3) Order history (cloud)

Customer and CSR views; read-optimized with pagination and filters.

### 4) Inventory availability (cloud and/or edge)

Expose availability with explicit staleness semantics. Callers provide `sales_channel_id` (demand context) and either:

- request availability for a specific `stock_location_id`, or
- request a result set keyed by `stock_location_id` (multi-location ATP).

### 5) Edge caches/read models (store-edge support)

In edge mode, keep Projections as derived/read-only datasets that support an edge-deployed transactional Domain. Typical edge caches:

- product subset snapshot (for a store channel)
- price book + promotion ruleset snapshots (effective-dated)
- tender configuration snapshot
- device/register configuration snapshot
- “last sync / lag” health read model per `store_site_id`

## Inputs

Prefer facts/events ingestion per v2 Projection guidance (`520-primitives-stack-v2-research-projection.md`):

- consume `commerce.domain.facts.v1` (and `commerce.process.facts.v1` where needed)

Fallback/bridge:

- watch CRs for hostname mapping if those are the only source of truth in v1.

## Query API sketch (examples)

- `ResolveHostname(hostname)`
- `SearchProducts(sales_channel_id, query, filters)`
- `GetOrderHistory(customer_id, cursor)`
- `GetAvailability(sales_channel_id, sku, stock_location_id)`
- `GetEdgeSnapshot(store_site_id, version)`

## Stack alignment (proto / generation / operator)

- Proto: query APIs + projected schema declarations (read-only); input bindings declared as facts/events or CDC.
- Generation: SDKs + ingestion stubs + schema/migrations + structural tests (determinism/idempotency).
- Operator: owns projection workloads, stores/references, migrations/backfills, HTTPRoute exposure, and health/lag conditions.

## Consistency requirements

- Hostname resolution must converge quickly on revoke (fail closed; deny by default on uncertainty).
- Edge caches/read models must expose `last_synced_at` and `checkpoint` markers per `store_site_id` for UX and diagnostics.
