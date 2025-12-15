# 520 — Primitives Stack v2: Projection primitive (implementation guidance)

**Status:** Draft (normative once accepted)  
**Scope:** How to implement the **Projection** primitive kind under the `520` stack.

Projection primitives build derived read models and expose read-only query services. They are never canonical writers.

## Intent (ArchiMate)

- **Application Component:** Projection primitive runtime
- **Realizes:** read-only Application Services (queries)
- **Uses:** Technology Services (streams, DB/analytics store, K8s) per `520-primitives-stack-v2.md`

## Non-negotiables

- Projections are derived from facts/events (and/or CDC) and are safe under at-least-once delivery.
- Rebuild/replay must converge to the same read state (facts are source; projections are derived).
- UISurfaces/Agents query projections; they do not couple to domain internal tables.

## Processing semantics (required)

- Choose and document at-least-once vs exactly-once.
- Offsets/checkpoints + idempotency are non-negotiable.
- Exactly-once requires offset+side effects committed in the same transaction for transactional sinks.

## Scaffold stance (implementation approach)

This primitive follows the `520` default stance: **won’t compile until wired**.

Meaning (required):

- Generated ingestion must require explicit handler wiring (or explicit input bindings) so drift/missing dependencies fail at compile time (or CI), not at runtime.

## Storage options (pluggable)

- Postgres projection tables (default)
- Postgres materialized views (explicit refresh semantics)
- Columnstores (e.g., ClickHouse) when workload demands
- CDC from transactional DBs when events are unavailable

## Vector knowledge as a Projection (CNPG + pgvector)

Vector knowledge stores fit Ameide cleanly **only** as Projection-backed read models (derived representations optimized for semantic retrieval), never as canonical repositories.

Rules (required):

- **Domains remain canonical writers.** Domains emit facts; they do not compute embeddings and do not depend on vector storage.
- **Projections own indexing.** A Projection consumes facts, fetches canonical content by ID/version, chunks+embeds deterministically, and upserts idempotently.
- **Agents are read-only consumers.** Agents query vector knowledge via a read-only query service and treat results as evidence pointers back to canonical IDs/versions/baselines.

Idempotency/provenance (minimum):

- Store embeddings by provenance keys such as `(tenant_id, source_kind, source_id, source_version, chunk_hash, embedding_model_id)`.
- Retrieval results must carry canonical references (stable IDs and optional version pins).

Operational note:

- Example only (non-normative): CNPG-managed Postgres can host pgvector if the extension is present in the operand image and installed per database (e.g., `CREATE EXTENSION vector;`).

## Behavior plane (Proto → generation)

- **Proto defines**
  - read-only query services
  - projected schema (read models) and input bindings (facts/CDC)
- **Generation emits**
  - ingestion stubs + checkpoint store schema
  - query service stubs + harness/tests

Scope note: `520` standardizes envelope discipline (metadata, idempotency, trace/correlation) and ingestion guarantees; **projection schema and query semantics** are capability-defined.

## Conformance checklist (Projection)

- Schema/migrations are generator-owned under generated roots; rebuild/replay is supported and observable.
- Ingestion is idempotent and checkpointed; replay produces the same state.
