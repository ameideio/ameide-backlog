Note: this document is supporting background; the consolidated, normative spec is `backlog/520-primitives-stack-v2.md`. Examples use placeholder namespaces (e.g., `acme.*`)—the canonical repo proto namespace is `ameide_core_proto.*` per `backlog/509-proto-naming-conventions.md`.

## What the “Projection” primitive is

A **Projection** is a deployed **read projection / materialized view / analytical columnstore** built from transactional data and/or domain facts/events, exposing **query APIs** optimized for fast analytical reads.

Your principles imply a very specific shape:

* **Proto is the single source of truth** for:

  * query APIs (services + RPCs),
  * projected schema (tables/columns/keys),
  * and optionally the fact/event inputs that drive the projection.
* **Buf plugins** generate:

  * SDKs,
  * projection runtime skeletons (query handlers + ingestion stubs),
  * store schema + migrations,
  * tests + harness.
* **ProjectionOperator** (K8s controller) owns:

  * CRDs, Deployments/Jobs, Services/HTTPRoutes, secrets/config,
  * DB/columnstore resources or references,
  * migrations/backfills/HPAs,
  * status conditions.
* **CI guardrails** only (no bespoke CLI):

  * Buf lint/breaking, regen-diff, import-policy checks,
  * generated structural tests, human behavior tests.

The rest of this answer is the “smartest implementation” that respects those constraints.

---

## Findings from official docs you can build on

### Buf `buf generate`, remote plugins, pinning, determinism

* Remote plugins are referenced from the BSR in `buf.gen.yaml`, and Buf explicitly recommends **pinning plugin versions** for reproducible generation. ([Buf][1])
* In **v2 `buf.gen.yaml`**, you can also pin an *exact build* of a plugin with `revision` (Buf’s rebuild/repackage sequence number). ([Buf][2])
* `clean: true` in `buf.gen.yaml` makes `buf generate` delete plugin output directories before generating, which is crucial for rename/deletion correctness and deterministic “regen-diff” gates. ([Buf][2])
* Buf advertises that its internal Protobuf compiler uses parallelism “without compromising deterministic output.” ([GitHub][3])
* Plugins are invoked in parallel, but outputs are written in the order you specify in `buf.gen.yaml` (so you can keep stable output layering). ([Buf][2])

**Implication for Projection:** pin **every** remote plugin + use `revision` for “byte-for-byte reproducible,” always set `clean: true`, and make your custom generator produce deterministic file ordering and content (no timestamps, random IDs, etc.).
Also: implement the generator on a standard plugin harness (e.g., Buf’s `bufbuild/protoplugin`) and add golden tests that assert byte-for-byte deterministic output from fixed descriptor inputs.

---

### Protobuf custom options / annotations best practices (for your Projection markers)

* Custom options are defined by **extending** descriptor option messages (`FileOptions`, `ServiceOptions`, `MessageOptions`, `FieldOptions`, etc.). ([Protocol Buffers][4])
* Since custom options are extensions, they need field numbers; Protobuf guidance notes the **50000–99999** range is reserved for internal use within an organization. ([Protocol Buffers][4])
* Proto3 includes **Option Retention** and **Option Targets**. Targets let you constrain which entity types fields in an options message can apply to; retention lets you keep option metadata out of runtime descriptors if you only need it at codegen time. ([Protocol Buffers][5])

**Implication for Projection:**
Define one canonical `projection/annotations.proto` file that contains all your custom options, use reserved internal option numbers (50000+), and (optionally) mark heavy schema metadata as **source-retained** when you don’t need it in runtime descriptors.
Also: write generators to read SOURCE-retained options from `CodeGeneratorRequest.source_file_descriptors` (not only `proto_file`) so options are not “mysteriously” missed.

---

### Kubernetes operator/controller-runtime + Gateway API HTTPRoute

* Operator reconciliation loops MUST be **idempotent**—re-running reconcile converges on the same desired state. ([Kubebuilder Book][6])
* Controllers are **event-driven** and MUST watch resources; controllers MUST NOT rely on polling via `RequeueAfter`. ([Kubebuilder Book][7])
* controller-runtime’s reconcile contract documents requeue behavior (including exponential backoff in some cases). ([Go Packages][8])
* Gateway API’s **HTTPRoute** spec centers on `parentRefs`, `hostnames`, and `rules` composed of `matches`, `filters`, and `backendRefs`. ([Kubernetes Gateway API][9])
* Kubernetes documents Gateway API as the role-oriented, protocol-aware routing mechanism (so treating HTTPRoute as your “front door” primitive is aligned). ([Kubernetes][10])

**Implication for Projection:**
Have the operator reconcile **HTTPRoute → Service → Deployment** (plus migrations/HPAs) and expose status via standard `metav1.Condition` types.

---

## Storage and ingestion options (2+ compared, with operational interfaces)

### Storage option A: “in-DB” materialized view (Postgres)

* `CREATE MATERIALIZED VIEW` creates a persisted, table-like result of a query. ([PostgreSQL][11])
* `REFRESH MATERIALIZED VIEW` **replaces the contents** of the MV (i.e., full refresh semantics by default). ([PostgreSQL][12])

**Operational interface**

* SQL DDL (`CREATE MATERIALIZED VIEW`, indexes on MV)
* Refresh scheduling (cron/job) or manual refresh
* Migration = SQL migration files

**Strengths**

* Lowest infra overhead (just Postgres)
* Great for “cached query results” where freshness can be coarse

**Weaknesses**

* Refresh is usually heavyweight; incremental maintenance is not native in core Postgres MVs
* Concurrency + refresh mechanics complicate “always-fresh” analytics at scale

---

### Storage option B: “external analytical store” (ClickHouse columnstore)

* ClickHouse’s MergeTree family uses an **ORDER BY** key that determines sort order within parts, and supports TTL operations. ([ClickHouse][13])
* ClickHouse `CREATE TABLE` is standard SQL DDL with engine selection and column definitions. ([ClickHouse][14])
* TTL can remove rows after time windows at the table level (good for “rolling analytics”). ([ClickHouse][15])

**Operational interface**

* ClickHouse DDL (tables, materialized views, TTL, partitions)
* Migrations via DDL scripts
* Query via native protocol/HTTP interfaces (implementation choice)

**Strengths**

* Excellent for high-cardinality analytics and large scans/aggregations
* TTL/partitioning patterns are first-class for analytics retention

**Weaknesses**

* Extra operational surface area (cluster, backups, tuning, replication)
* Need careful schema design (ORDER BY / partition keys) to avoid pain

---

### Storage option C: event-driven read model tables in Postgres (v2 default)

This is “plain tables” updated incrementally from facts/events (or CDC). It’s not “materialized view,” it’s a projection table.

**Operational interface**

* Standard Postgres tables + indexes
* Migrations as SQL
* Backfill as batch job (replay events) or CDC snapshot

**Strengths**

* Near-real-time reads (incremental updates)
* Simple infra (still Postgres)
* Fits event-driven architecture

**Weaknesses**

* You must build idempotent ingestion & replay/backfill
* Complex analytical queries can outgrow Postgres at scale

---

### Ingestion option 1: Pull from transactional DB (CDC / logical decoding / queries)

* Postgres logical replication uses publisher/subscriber concepts; a subscription defines the downstream side. ([PostgreSQL][16])
* Logical decoding extracts persistent changes into a coherent format. ([PostgreSQL][17])

**Operational interface**

* replication slot/publication/subscription (or a CDC connector)
* checkpointing + schema evolution
* backfill via snapshot + stream

**Pros**

* Works even if domain events aren’t available
* Can be applied incrementally during adoption

**Cons**

* Couples projection correctness to DB schema and CDC semantics
* Harder to express “domain intent” than facts/events

---

### Ingestion option 2: Consume domain facts/events (v2 default)

* Debezium’s Outbox Event Router docs describe the outbox pattern as a way to avoid inconsistencies between DB state and events, capturing changes from an outbox table for downstream consumers. ([Debezium][18])

**Operational interface**

* subscribe to topics/streams
* idempotent handler (dedupe key + checkpoint)
* replay/backfill = re-consume from offset/time

**Pros**

* Clean domain contracts and looser coupling
* Natural support for replay/backfill
* Projection schema can evolve independently from OLTP schema

**Cons**

* Requires event discipline + schema governance
* Needs idempotency + ordering strategy

**Interoperability requirement (cross-boundary):** treat consumed/emitted facts/events as CloudEvents-compatible on the wire and propagate W3C Trace Context (e.g., `traceparent`), mapping shared `event_fact.type` to CloudEvents `type`.

---

## Processing semantics are explicit (offsets + idempotency)

Projection ingestion must choose and document its processing semantics; “offsets + idempotency” is the universal requirement across projection frameworks:

### Option A: at-least-once (default)

* Ingestion reprocesses events after restarts/rebalances.
* **Requirement:** projection handlers must be idempotent (use a stable idempotency key derived from the event/fact envelope and/or `(…event_fact).type` + key fields).
* Offset/checkpoint commits happen independently of sink writes, so duplicates must be safe.

### Option B: exactly-once (transactional sink)

* Achieve “effectively exactly-once” by committing **offset/checkpoint + sink side effects in the same transaction** (e.g., relational DB transaction).
* **Requirement:** a transactional sink and a checkpoint schema that can be updated in the same DB transaction as projection writes.
* Tradeoff: throughput/latency depends on checkpoint frequency and transaction overhead; expose checkpoint cadence as config and surface lag/health.

### Generated contracts (required)

* **Checkpoint/offset store schema** (e.g., a table keyed by `{projection_name, binding_ref, partition}` with last-processed offset + updated_at).
* **Handler skeletons that enforce idempotency**:
  * require an idempotency key per consumed binding (`ConsumesFact.idempotency_key` or a default derived from event metadata)
  * provide a standard “seen key” check/update path (at-least-once) or “offset+write” transaction wrapper (exactly-once).
* **Operational endpoints/metrics** that report lag and checkpoint position (feeds `IngestionHealthy` / `/admin/ingestion`).

---

## Recommendation + decision matrix (Postgres MV vs external columnstore vs event-driven tables)

### Recommendation

1. **Default**: **event-driven projection tables in Postgres** (incremental updates).
2. **Scale-up path**: add **ClickHouse** for projections that truly need columnstore performance / retention / large scans.
3. **Use Postgres MVs selectively** for “periodic cached reports” where full refresh cost and freshness constraints are acceptable.

This keeps the primitive aligned with EDA principles without forcing every team onto a columnstore.

### Decision matrix

| Criterion              | Postgres Materialized View                                                 | External Columnstore (ClickHouse)                                        | Event-driven Postgres Tables           |
| ---------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------ | -------------------------------------- |
| Freshness              | Low–Medium (refresh cadence; refresh replaces contents) ([PostgreSQL][12]) | Medium–High (stream ingestion; append/merge patterns) ([ClickHouse][13]) | High (incremental per event)           |
| Operational complexity | Low                                                                        | Medium–High                                                              | Low–Medium                             |
| Query workload fit     | Simple aggregates, cached queries                                          | Large scans, high-cardinality aggregation                                | Fast read models, moderate analytics   |
| Migrations             | SQL                                                                        | DDL scripts                                                              | SQL                                    |
| Backfill story         | Refresh or rebuild MV                                                      | Replay into CH                                                           | Replay events / rebuild tables         |
| Best when              | “Good enough analytics” on OLTP DB                                         | True analytics platform needs                                            | EDA read models + near-real-time views |

---

## The “smartest implementation” of the trio

## 1) Proto shape

### Goals

* Proto is authoritative for:

  * query API surface,
  * projected schema,
  * ingestion bindings (required).
* Annotations must be:

  * strongly typed,
  * codegen-friendly,
  * stable and evolvable.

### Annotation schema design

Create a dedicated annotations file, e.g.:

* `acme/platform/projection/v1/annotations.proto`

  * defines options extending:

    * `google.protobuf.ServiceOptions` → ProjectionQueryService marker
    * `google.protobuf.MessageOptions` → ProjectionSchema marker (table)
    * `google.protobuf.FieldOptions` → ProjectionColumn marker (column)
    * `google.protobuf.ServiceOptions` only → ConsumesFact bindings (method-level bindings are not supported in v2)

This uses the Protobuf-approved mechanism: extending descriptor option messages. ([Protocol Buffers][4])
Use internal option numbers in 50000–99999. ([Protocol Buffers][4])
Option Targets/Retention are available if you want to constrain option usage and/or keep metadata source-only. ([Protocol Buffers][5])

### Concrete annotations.proto (proposed)

```proto
// Keep custom option extensions in proto2 syntax.
syntax = "proto2";

package acme.platform.projection.v1;

import "google/protobuf/descriptor.proto";

// =======================
// Service-level options
// =======================

message ProjectionQueryService {
  // Stable projection identity (used in generated paths, k8s labels, etc.)
  optional string projection_name = 1;

  // Default HTTP base path. v2 generators ignore this.
  optional string http_path_prefix = 2;

  // Optional: declare which store kinds this projection supports.
  repeated string supported_store_kinds = 3;
}

message ConsumesFact {
  // Cross-primitive requirement: reuse the shared "event/fact metadata spine" for stable identifiers.
  // If you must reference a message by string, treat this as an external/public identifier, not an internal type name.
  // **Hard alignment rule:** this value must match the shared `(…event_fact).type` identifier on the consumed fact/event message
  // (i.e., do not invent a second naming scheme for projections).
  optional string fact_type = 1;

  // Do not place per-environment transport bindings (topic names, broker URLs, etc.) in proto.
  // Bind these via the Projection CRD/runtime config and reference them here by logical name.
  optional string binding_ref = 2;

  // Idempotency key expression hint (store-neutral).
  // Example: "source + ':' + id" or "order_id"
  optional string idempotency_key = 3;
}

// Extend ServiceOptions (custom options are done by extending descriptor options)
extend google.protobuf.ServiceOptions {
  optional ProjectionQueryService projection_query_service = 50001;
  repeated ConsumesFact consumes_fact = 50002;
}

// =======================
// Message-level options (schema/table)
// =======================

message ProjectionSchema {
  // Store-neutral logical table name.
  optional string table = 1;

  // One or more primary key columns (by proto field name, unless overridden by column option).
  repeated string primary_key = 2;

  // Reserved logical partitioning hint. Generators ignore this in v2.
  repeated string partition_by = 3;
}

extend google.protobuf.MessageOptions {
  optional ProjectionSchema projection_schema = 50011;
}

// =======================
// Field-level options (schema/column)
// =======================

message ProjectionColumn {
  // Physical column name override (defaults to snake_case field name).
  optional string name = 1;

  // Whether null is allowed (defaults derived from scalar vs message).
  optional bool nullable = 2;

  // Reserved indexing/sort hints. Generators ignore these in v2.
  optional bool indexed = 3;
  optional uint32 sort_key_order = 4;
}

extend google.protobuf.FieldOptions {
  optional ProjectionColumn projection_column = 50021;
}
```

*(The key is: store-neutral metadata + “hinting,” not store-specific DDL embedded in proto.)*

---

## 2) Example .proto snippet (query RPCs + schema messages + consumes bindings)

```proto
syntax = "proto3";

package acme.orders.analytics.v1;

import "acme/platform/projection/v1/annotations.proto";

// Query API surface (source-of-truth)
service OrdersAnalyticsQuery {
  option (acme.platform.projection.v1.projection_query_service) = {
    projection_name: "orders-analytics"
    http_path_prefix: "/projections/orders-analytics"
    supported_store_kinds: "postgres_table"
    supported_store_kinds: "clickhouse"
  };

  // If you declare logical ingestion bindings in proto (transport binding remains CRD/runtime config):
  option (acme.platform.projection.v1.consumes_fact) = {
    fact_type: "orders.order_placed.v1"
    binding_ref: "orders-facts"
    idempotency_key: "order_id"
  };
  option (acme.platform.projection.v1.consumes_fact) = {
    fact_type: "orders.order_refunded.v1"
    binding_ref: "orders-facts"
    idempotency_key: "refund_id"
  };

  rpc GetDailySales(GetDailySalesRequest) returns (GetDailySalesResponse);
}

message GetDailySalesRequest {
  string tenant_id = 1;
  string day_yyyy_mm_dd = 2;
}

message GetDailySalesResponse {
  repeated DailySalesRow rows = 1;
}

// Projected table row schema
message DailySalesRow {
  option (acme.platform.projection.v1.projection_schema) = {
    table: "daily_sales"
    primary_key: "tenant_id"
    primary_key: "day_yyyy_mm_dd"
    partition_by: "tenant_id"
  };

  string tenant_id = 1 [(acme.platform.projection.v1.projection_column) = {
    name: "tenant_id"
    nullable: false
    indexed: true
    sort_key_order: 1
  }];

  string day_yyyy_mm_dd = 2 [(acme.platform.projection.v1.projection_column) = {
    name: "day"
    nullable: false
    indexed: true
    sort_key_order: 2
  }];

  int64 order_count = 3 [(acme.platform.projection.v1.projection_column) = {
    name: "order_count"
    nullable: false
  }];

  int64 gross_cents = 4 [(acme.platform.projection.v1.projection_column) = {
    name: "gross_cents"
    nullable: false
  }];
}
```

This uses “extend descriptor options” per Protobuf’s documented mechanism. ([Protocol Buffers][4])

---

## 3) Buf plugins

### a) `buf.gen.yaml` (pinned remote plugins + deterministic outputs)

Key doc-grounded knobs:

* `clean: true` to remove stale generated outputs. ([Buf][2])
* pin remote plugins by version, and optionally `revision` for exact packaging reproducibility. ([Buf][2])

Example:

```yaml
# buf.gen.yaml
version: v2
clean: true

managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/acme/platform/gen/go

plugins:
  # SDK generation (Go example)
  - remote: buf.build/protocolbuffers/go:v1.36.10
    revision: 1
    out: packages/ameide_sdk_go/internal/proto
    opt: paths=source_relative

  - remote: buf.build/grpc/go:v1.5.1
    revision: 1
    out: packages/ameide_sdk_go/internal/proto
    opt: paths=source_relative

  # Projection primitive generator (your custom remote plugin published to BSR)
  - remote: buf.build/acme/projection-runtime-go:v0.1.0
    revision: 3
    out: build/generated/projection
    opt:
      - module=orders-analytics
      - default_store=postgres_table
      - also_emit=clickhouse
```

Remote plugins and version pinning are standard in Buf’s docs. ([Buf][1])

### b) Exact generated file tree (one Projection)

Assume the projection is named `orders-analytics` and you generate under `build/generated/projection/orders-analytics/`.

**Implementation-owned**

* `primitives/projection/orders-analytics/app/` (implementation)
* `primitives/projection/orders-analytics/Dockerfile`
* `primitives/projection/orders-analytics/README.md`

**Generated (always overwritten)**

* `*.generated.*` OR under a dedicated `gen/` subtree

Proposed tree:

```text
packages/ameide_core_proto/
  src/ameide_core_proto/…                  # proto sources

packages/ameide_sdk_go/
  gen/go/…                                 # SDK outputs

build/generated/
  projection/
    orders-analytics/                      # GENERATED-ONLY (safe to delete)
      runtime-go/
        cmd/
          projection-api/main.generated.go
          projection-ingest/main.generated.go
          projection-migrate/main.generated.go
        internal/
          query/{handlers,server}.generated.go
          ingest/{consumer,dbsync,checkpoints}.generated.go
          store/{store,schema}.generated.go
        migrations/{postgres,clickhouse}/*.generated.sql
        tests/**/*
      manifests/projection.generated.yaml  # generated CR example (NOT applied by a CLI)

primitives/
  projection/
    orders-analytics/                      # IMPLEMENTATION-OWNED (never cleaned)
      app/
        query/handlers.go
        ingest/{reducer,mapping}.go
        store/custom_indexes.sql
      behavior_test.go
```

### c) Generated vs implementation-owned split rules

**Rules**

* Generator writes only to `build/generated/projection/**` and SDK output roots.
* Every generated file ends with `.generated.<ext>` (or lives under a `gen/` folder) and contains a header like: “DO NOT EDIT”.
* Implementation-owned code lives outside `gen/` and never gets rewritten.

**Deletion/rename strategy**

* `clean: true` in Buf generation deletes plugin output directories before generation, preventing stale orphaned outputs. ([Buf][2])
* Within a projection, the custom generator:

  * names files deterministically from fully-qualified proto names,
  * sorts emitted tables/columns/rpcs lexicographically,
  * never includes timestamps.

**CI regen-diff gate**

* Run:

  * `buf lint` + `buf breaking` (see CI section below)
  * `buf generate`
  * `git diff --exit-code` (fails if regeneration changes tracked files)

(Determinism is a first-class goal in Buf tooling. ([GitHub][3]))

---

## 4) Operator internals

### a) CRD spec fields (Projection)

**Group/Kind**

* `ameide.io/v1`, Kind `Projection`

**Spec (core fields)**

```yaml
spec:
  image: "ghcr.io/acme/orders-analytics:1.2.3"

  query:
    port: 8080
    protocol: "h2c"             # or http1; depends on your gateway/proxy
    service:
      annotations: {}
    route:
      parentRefs:
        - name: "shared-gateway"
          namespace: "gateway-system"
      hostnames: ["analytics.ameide.internal"]
      pathPrefix: "/projections/orders-analytics"

  store:
    kind: "postgres_table"       # enum: postgres_mv | postgres_table | clickhouse
    # Either reference an external managed cluster, or let operator provision via another CR.
    connectionSecretRef:
      name: "orders-analytics-store-conn"
    database: "analytics"
    schema: "orders_analytics"

  migrations:
    strategy: "Job"              # Job | InitContainer
    dir: "/migrations/postgres"
    timeoutSeconds: 600

  ingestion:
    mode: "events"               # events | cdc | disabled
    events:
      broker: "kafka"            # or nats, etc.
      topics:
        - "orders.facts"
      consumerGroup: "orders-analytics"
    cdc:
      # alternative is not supported in v2
      sourceConnectionSecretRef:
        name: "orders-oltp-conn"
      publication: "orders_pub"
      slot: "orders_analytics_slot"

  workload:
    query:
      replicas: 2
      resources:
        requests: { cpu: "250m", memory: "256Mi" }
        limits:   { cpu: "2",    memory: "2Gi" }
    ingest:
      replicas: 1
      resources:
        requests: { cpu: "250m", memory: "256Mi" }
        limits:   { cpu: "2",    memory: "2Gi" }

  autoscaling:
    query:
      enabled: true
      minReplicas: 2
      maxReplicas: 20
      targetCPUUtilizationPercentage: 70
    ingest:
      enabled: true
      minReplicas: 1
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70
```

### b) Status conditions (and why these map cleanly to runtime)

Use `status.conditions[]` as `metav1.Condition` with the standard `status` values True/False/Unknown. ([Go Packages][19])

Recommended condition types for Projection:

* `SecretsReady` — all referenced secrets exist (store DSN, broker creds, etc.)
* `DependenciesReady` — external deps reachable (store, broker, etc.)
* `WorkloadReady` — Deployments Available (query and, if enabled, ingest)
* `StoreReady` — store reachable and expected DB objects exist
* `MigrationSucceeded` — last migration job succeeded
* `BackfillComplete` — backfill checkpoint reached “done”
* `IngestionHealthy` — ingestion heartbeat fresh + lag under threshold
* `RouteReady` — HTTPRoute attached/accepted by Gateway implementation
* `Ready` — umbrella condition: query ready AND (if enabled) ingest healthy AND migrations ok

(Use `metav1.Condition` everywhere and set `observedGeneration` to the Projection CR’s `.metadata.generation`; keep condition types positive-polarity and use `reason`/`message` for detail. For routing, map `RouteReady` from HTTPRoute `Accepted`/`ResolvedRefs`/`Programmed` where available.)

### c) Mapping conditions → runtime contracts (env vars, mounts, health endpoints)

**Operator-owned runtime contract** (language-agnostic):

* Env vars:

  * `PROJECTION_NAME`
  * `PROJECTION_STORE_KIND`
  * `PROJECTION_STORE_DSN` (from secret)
  * `PROJECTION_MIGRATIONS_DIR`
  * `PROJECTION_INGEST_MODE`
  * `PORT_HTTP` (query API + health/metrics bind; standard operator↔runtime waist)
  * `PROJECTION_HTTP_PORT` is forbidden; any existing alias MUST map to `PORT_HTTP`
* Mounts:

  * migrations dir packaged in image OR mounted via ConfigMap (I recommend packaged in image for immutability)
* Health endpoints:

  * `GET /healthz` (liveness)
  * `GET /readyz` (readiness)
  * `GET /admin/ingestion` (lag/checkpoint/backfill progress)
  * `GET /metrics` (Prometheus)

### d) Reconcile outline (controller-runtime style)

Idempotent, event-driven reconcile is a first principle. ([Kubebuilder Book][6])

**Reconcile steps**

1. Fetch Projection CR.
2. Ensure finalizer; on delete, clean owned resources (or rely on ownerRefs).
3. Reconcile secrets/config wiring (DSN, ingestion bindings, etc.).
4. Reconcile store:

   * If `storeRef` is external: check connectivity and set `StoreReady`.
   * If operator provisions store: reconcile subordinate CRs/StatefulSets.
5. Reconcile migration execution:

   * If `strategy=Job`: ensure Job exists and completed successfully; set `MigrationSucceeded`.

---

## Reconcile discipline note (keep reconcile short)

Projections often involve long-running operations (migrations, backfills, rebuilds, large store provisioning). Keep reconcile responsive:

* Reconcile MUST be **idempotent** and **short**: create/patch Jobs and observe their status; do not run migrations/backfills inside the controller process.
* Surface progress via conditions (`MigrationSucceeded`, `BackfillComplete`, `IngestionHealthy`) with clear `reason`/`message`.
* Prefer watching Jobs/Deployments/HTTPRoutes to drive requeues instead of polling with long sleeps.
6. Reconcile Deployments:

   * `projection-api` Deployment
   * `projection-ingest` Deployment (if enabled)
7. Reconcile Services for each Deployment.
8. Reconcile HTTPRoute (Gateway API):

   * Set `parentRefs`, `hostnames`, and a path match to route to the Service backend. HTTPRoute’s structure is standardized around these fields. ([Kubernetes Gateway API][9])
9. Reconcile HPAs.
10. Update status conditions (ObservedGeneration, reasons/messages).

---

## CI guardrail plan

### 1) Schema/API hygiene

* `buf lint` in CI to enforce consistent proto conventions. ([Buf][20])
* `buf breaking` against your main branch (or last release tag) to prevent accidental breaking changes. ([Buf][21])

### 2) Deterministic regen gate

* Run `buf generate` with pinned remote plugins (and `clean: true`) ([Buf][2])
* Fail CI if `git diff` is non-empty (regen-diff check)

### 3) Import policy checks (no “core proto packages” in runtime)

You want: **runtime code imports SDK outputs only**.

Two complementary checks:

1. **Proto import policy** (schema layer):

   * enforce allowed import roots (`google/*`, `acme/platform/projection/*`, domain facts, etc.)
2. **Runtime dependency policy** (code layer):

   * forbid imports from proto source directories in runtime packages; allow only `gen/<lang>/**` SDK packages.

Implementation options:

* A small CI script (fastest to ship), OR
* A **Buf check plugin** (better long-term governance): Buf lint supports custom rules via plugins. ([Buf Docs][22])

### 4) Generated structural tests

Generated by the Projection plugin:

* compilation tests that ensure:

  * server wiring compiles with unimplemented stubs,
  * schema/migration files exist for configured store kinds,
  * handlers cover every RPC with a stub.

### 5) Human-written behavior tests

Per-projection test suite (owned by engineers):

* “Given these facts/events, when I query X, I get Y”
* “Backfill from empty store produces expected rows”
* “Migration from N-1 to N preserves correctness”

---

## How this satisfies your success criteria

### Operators remain language-agnostic

* Operator manipulates only K8s resources + store references + Jobs/HPAs/HTTPRoutes, and relies on a **runtime contract** (env vars + endpoints), not language-specific internals.

### Runtime code imports SDK outputs only

* SDKs are generated into `gen/<lang>/...`.
* Runtime skeleton imports only those generated packages (and never imports proto source trees).

### Projections regenerate deterministically and validate in CI

* Remote plugin pinning + `revision` + `clean: true` gives stable generation inputs/outputs. ([Buf][2])
* Regen-diff gate + structural tests enforce determinism continuously.

### Operational behavior visible via conditions

* Migrations/backfill/ingestion health are represented as standard conditions (`StoreReady`, `MigrationSucceeded`, `IngestionHealthy`, `BackfillComplete`, `Ready`) using Kubernetes conventions.

---

## Practical next steps (what I would implement first)

1. Ship `projection/v1/annotations.proto` + generator plugin that emits:

   * Postgres-table projection schema + migrations
   * Query server bootstrap + handler interfaces
   * Event-consumer ingestion skeleton + checkpoint table schema
   * Integration harness (docker compose with Postgres)
2. Add ClickHouse generator backend later (same annotations).
3. Implement ProjectionOperator CRD + reconcile:

   * migration Job + API Deployment + ingest Deployment + HTTPRoute + HPA
   * conditions wired to Job completion + readiness + heartbeat table.

If you want, I can also provide a concrete HTTPRoute YAML template and a reconcile pseudocode skeleton (controller-runtime) that matches the condition model above—using only the contracts outlined here.

[1]: https://buf.build/docs/bsr/remote-plugins/ "Remote plugins - Buf Docs"
[2]: https://buf.build/docs/configuration/v2/buf-gen-yaml/ "buf.gen.yaml - Buf Docs"
[3]: https://github.com/bufbuild/buf "bufbuild/buf: The best way of working with Protocol Buffers."
[4]: https://protobuf.dev/programming-guides/proto2/ "Language Guide (proto 2) | Protocol Buffers Documentation"
[5]: https://protobuf.dev/programming-guides/proto3/ "Language Guide (proto 3) | Protocol Buffers Documentation"
[6]: https://book.kubebuilder.io/reference/good-practices "Good Practices"
[7]: https://book.kubebuilder.io/reference/watching-resources "Watching Resources"
[8]: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile "reconcile package - sigs.k8s.io/controller-runtime"
[9]: https://gateway-api.sigs.k8s.io/api-types/httproute/ "HTTPRoute"
[10]: https://kubernetes.io/docs/concepts/services-networking/gateway/ "Gateway API"
[11]: https://www.postgresql.org/docs/current/sql-creatematerializedview.html "CREATE MATERIALIZED VIEW"
[12]: https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html "REFRESH MATERIALIZED VIEW"
[13]: https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree "MergeTree table engine | ClickHouse Docs"
[14]: https://clickhouse.com/docs/sql-reference/statements/create/table "CREATE TABLE | ClickHouse Docs"
[15]: https://clickhouse.com/docs/guides/developer/ttl "Manage Data with TTL (Time-to-live)"
[16]: https://www.postgresql.org/docs/current/logical-replication-subscription.html "Subscription (logical replication)"
[17]: https://www.postgresql.org/docs/current/logicaldecoding-explanation.html "Logical Decoding Concepts"
[18]: https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html "Outbox Event Router"
[19]: https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1 "v1 package - k8s.io/apimachinery/pkg/apis/meta/v1"
[20]: https://buf.build/docs/lint/ "Linting and code standards - Buf Docs"
[21]: https://buf.build/docs/breaking/ "Breaking change detection - Buf Docs"
[22]: https://buf.build/docs/lint/rules/ "Rules and categories - Buf Docs"
