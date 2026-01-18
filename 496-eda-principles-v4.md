---
title: 496 – Integration & EDA Principles (v4: owner-writes, contract-first, hybrid messaging)
status: superseded
owners:
  - platform
created: 2026-01-18
parent: 496-eda-principles-v3.md
supersedes:
  - 496-eda-principles-v3.md
  - 496-eda-principles-v2.md
superseded_by:
  - 496-eda-principles-v6.md
---

# 496 – Integration & EDA Principles (v4: owner-writes, contract-first, hybrid messaging)

> **Superseded:** replaced by `backlog/496-eda-principles-v6.md` (`io.ameide.*` naming posture + Git-backed owners made explicit).

**Status:** Proposed Standard (supersedes `backlog/496-eda-principles-v3.md`)  
**Scope:** Inter-primitive (microservice-to-microservice) communication. Not primitive-internal control flow.

This v4 reframes “EDA” as an **industry-standard hybrid integration model**:

- **Commands** (typically RPC) request changes from an owner.
- **Facts** (typically events) propagate committed outcomes for replay, fan-out, and derived read models.
- **Proto contracts** are the governance surface for both.

Kafka remains the **standard event plane** in Kubernetes, but v4 avoids the “everything is Kafka” posture by explicitly allowing RPC where appropriate.

---

## 0) Definitions (contract terms)

- **Primitive**: a deployable runtime unit (Domain / Process / Projection / Integration / Agent / UISurface). Polyglot runtimes are expected.
- **Owner**: the single authoritative write surface for some canonical state and its invariants.
  - In practice this is usually a Domain service.
  - Some non-domain primitives may own canonical state scoped to their runtime (e.g., process instance state), but that does not make them owners of business-domain truth.
- **Command**: request to an owner to change canonical state (often RPC; may be async).
- **Intent**: asynchronous request to an owner to change canonical state (event/message form).
- **Fact**: immutable statement emitted only by the owner after commit; used for propagation and derivations.
- **Derived read model**: rebuildable cache/index/view; never canonical truth.

---

## 1) Non‑negotiables (the principles that must hold)

### 1.1 Owner-only writes (the core invariant)

- No primitive writes another primitive’s database.
- No primitive mutates another owner’s canonical external store (e.g., a Git repository) except through the owner’s API/command surface.
- Process / Agent / Integration / UISurface may request changes, but the owner performs the write and emits the fact.

### 1.2 Facts only from owners, only after commit

Owners emit facts only after the outcome is durable and auditable.

What counts as “commit” depends on the owner’s persistence:

- **DB-backed owner:** state write + outbox row in one DB transaction.
- **Git-backed owner:** Git operation succeeds (commit / merge / tag) and the owner durably records the audit pointer(s) it will rely on (e.g., project id, branch, MR id, commit SHA, pipeline id) before fact emission.

### 1.3 Contract-first communication

- Public APIs and event payloads are defined in Protobuf.
- Compatibility is governed at the schema boundary (Buf + CI gates), not by convention.

### 1.4 Idempotency is mandatory

- Every command/intent has an idempotency key.
- Every fact has a stable identity; consumers must tolerate duplicates and replays.

### 1.5 Observability and traceability are mandatory

- Correlation and causation metadata must be propagated across inter-primitive calls (RPC and events).
- Trace context must be forwarded where possible.

---

## 2) Hybrid messaging model (industry-aligned guidance)

### 2.1 When to use RPC (commands / queries)

Prefer RPC when:

- the caller needs an immediate acknowledgement (validation, authorization, created IDs, current status),
- the operation is naturally request/response,
- the interaction is within a tight latency budget.

RPC is not an excuse to bypass owner-only writes: RPC calls must target the owner’s API.

### 2.2 When to use events (facts / propagation)

Prefer events when:

- multiple consumers need the outcome,
- you need replay/backfill,
- you’re updating projections/search/indexes,
- you want loose coupling across runtimes/teams.

Events are not request/response; do not use facts as a control-plane substitute for “did my write succeed”.

### 2.3 When to use intents (async commands)

Prefer intents when:

- you want decoupled async requests to an owner,
- you can tolerate eventual acknowledgement,
- you want fan-in from multiple producers without tight RPC coupling.

---

## 3) Standard event plane (default in Kubernetes)

Kafka remains the standard event plane in Kubernetes:

- **CloudEvents v1.0** envelope for metadata,
- **Protobuf** payloads for schema and codegen,
- **Kafka (Strimzi)** for transport/durability (topics + consumer groups).

v4 allows additional transport bindings (e.g., RPC callbacks, workflow-engine connectors) when justified, as long as the semantics in §1 are preserved and the interface remains proto-governed.

---

## 4) CloudEvents envelope (mandatory when using the event plane)

All facts/intents carried over the event plane MUST be valid CloudEvents v1.0:

- required attributes: `specversion`, `id`, `source`, `type`
- required extensions (minimum): `tenantid`, `correlationid`, `causationid`, `traceparent`, `tracestate`
- payload descriptors:
  - `datacontenttype = "application/protobuf"`
  - `dataschema` referencing the protobuf schema

`ce.id` is the global idempotency key for event consumption.

---

## 5) Topics and naming (operable, not dogmatic)

- `ce.type` is the semantic contract identifier and must be stable and versioned.
- Kafka topic naming is an operational choice; allowed strategies include:
  - `topic == ce.type` (simple, explicit),
  - topic families (e.g., `io.ameide.<owner>.<stream>.v1`) with `ce.type` still carrying the semantic identifier.

If you use topic families, document the mapping and version it.

---

## 6) Publishing rules (outbox generalized)

### 6.1 DB-backed owners: outbox is mandatory for facts

Owners MUST NOT publish facts directly from request handlers/transactions.

Required flow:

1) handler writes state + outbox row in one DB transaction
2) dispatcher publishes outbox rows
3) dispatcher marks outbox rows as published idempotently

### 6.2 Git-backed owners: “outbox-equivalent” is mandatory for facts

When canonical state includes Git mutations, owners MUST:

1) perform the Git operation (commit/merge/tag) idempotently,
2) persist audit pointers (MR id, commit SHA, pipeline ids, tag refs) in owner-controlled storage,
3) emit facts based on those pointers.

This prevents “Git succeeded but we emitted no fact” and “we emitted a fact but Git didn’t commit” split-brain outcomes.

---

## 7) Consumers (at-least-once reality; multiple idempotency strategies)

- Delivery is at-least-once; consumers MUST treat replays as normal.
- Consumers MUST be idempotent; acceptable strategies include:
  - an inbox table keyed by `ce.id`,
  - upsert-by-natural-key with monotonic version checks where applicable,
  - replayable projections that can be rebuilt from facts.

Partitioning SHOULD be deterministic when ordering matters (e.g., by aggregate id).

---

## 8) What these principles imply for each primitive type

This section exists to prevent “EDA principles” from being interpreted only through one primitive’s lens.

### 8.1 Domain primitives (owners of business truth)

- Domain primitives are the default **owners** for business-domain canonical state.
- All writes happen through Domain APIs/commands; Domains emit facts after commit (outbox or outbox-equivalent).
- Domains are allowed to use internal adapters (e.g., GitLab) as implementation details, but only Domains are permitted to execute those canonical writes.

### 8.2 Process primitives (workflow/orchestration runtimes)

- Process primitives own orchestration state (process instance variables, timers, retries, human task progression).
- Process primitives MUST NOT write business-domain canonical state directly; they request changes from owners via commands (RPC) or intents.
- Process primitives may span multiple owners/modules; this is normal (Saga / process manager pattern).
- Workflow-engine messages used for wait-state resumption (e.g., `work.completed.v1`) are orchestration mechanics; treat them as internal to the process runtime unless explicitly modeled as owner-emitted facts.

### 8.3 Projection primitives (derived read models)

- Projection primitives never own canonical business state.
- They consume owner facts to build derived views: search indexes, graphs, timelines, materialized query models.
- They must be rebuildable from facts (or from canonical Git + audit pointers where explicitly defined).

### 8.4 Integration primitives (external adapters)

- Integration primitives bridge external systems (webhooks, polling, connectors) into owner-facing commands/intents.
- They MUST be idempotent on inbound external events and MUST NOT mutate canonical business state except by calling the owner’s API.
- They may own connector configuration state (secrets refs, cursor checkpoints), but not the business domain’s canonical truth.

### 8.5 Agent primitives (non-deterministic proposal runtimes)

- Agents produce proposals, drafts, and evidence; they do not become owners of business truth.
- Agents request changes from owners via commands/intents and attach evidence outputs (diffs, summaries, test results).
- Agents may be reused across domains/modules; do not assume “an agent belongs to one domain”.

### 8.6 UISurface primitives (interactive entry points)

- UISurfaces call owner APIs for commands and query services for reads.
- UISurfaces do not publish facts; facts come from owners after commit.
- UISurfaces may emit telemetry, but telemetry is not business-domain facts.

---

## 9) Relationship to workflow runtimes (Zeebe/Temporal)

- Zeebe/Temporal are orchestration runtimes for Process primitives; they are not the EDA backbone.
- Owners emit facts on the event plane; process workers consume those facts (or receive callbacks) to progress workflows.
- RPC between process workers and owners is allowed and often preferred for command issuance; use request→wait→resume for long-running work.

---

## 10) Migration notes from v3

Compared to v3 (`backlog/496-eda-principles-v3.md`), v4:

- Keeps Kafka/CloudEvents/Protobuf as the standard event plane, but explicitly supports a hybrid RPC+events integration model.
- Reframes “topic == type” as an allowed default, not the only mapping strategy.
- Replaces “durable inbox mandatory for every consumer” with “idempotency strategy mandatory”.
- Adds explicit primitive-by-primitive implications to avoid one-angle interpretations.

---

## 11) Cross-references and superseded guidance

### 11.1 Superseded documents (normative)

The following documents are superseded by this v4 spec and should not be treated as the current standard:

- `backlog/496-eda-principles-v2.md` (Knative Broker/Trigger posture)
- `backlog/496-eda-principles-v3.md` (Kafka-only posture; “topic == type” as a hard rule)

### 11.2 Related documents (still valid; interpret through v4)

These documents depend on EDA semantics and should be interpreted using the owner-write + hybrid messaging model defined here:

- Runtime posture and naming: `backlog/520-primitives-stack-v6.md`
- Zeebe process semantics (request→wait→resume): `backlog/527-transformation-process-v4.md`, `backlog/527-transformation-e2e-sequence-v5.md`
- Git-backed canonical writes (Git as SoR; owners emit facts after Git commit/merge): `backlog/694-elements-gitlab-v6.md`

### 11.3 How to read older guidance under v4 (common conflicts)

If an older document asserts any of the following, treat it as guidance that is either relaxed or re-scoped by v4:

- “EDA is the only way primitives communicate” → v4 allows RPC for commands/queries while keeping facts for propagation (§2).
- “Kafka topic name must equal `ce.type`” → v4 allows that mapping but also allows topic families (§5).
- “Every consumer must implement a durable inbox table” → v4 requires idempotency, but allows multiple strategies (§7).
- “Outbox applies only to DB-backed state” → v4 generalizes outbox semantics for Git-backed owners (§6.2).
