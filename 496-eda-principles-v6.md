---
title: 496 – Integration & EDA Principles (v6: `io.ameide.*`, owner-writes, contract-first, hybrid messaging)
status: draft
owners:
  - platform
created: 2026-01-18
parent: 496-eda-principles-v4.md
supersedes:
  - 496-eda-principles-v4.md
  - 496-eda-principles-v3.md
  - 496-eda-principles-v2.md
---

# 496 – Integration & EDA Principles (v6: `io.ameide.*`, owner-writes, contract-first, hybrid messaging)

**Status:** Proposed Standard (supersedes `backlog/496-eda-principles-v4.md`)  
**Scope:** Inter-primitive (microservice-to-microservice) communication. Not primitive-internal control flow.

This v6 standard keeps the v4 integration model, but makes two points explicit:

- **Naming posture:** inter-primitive semantic identities use `io.ameide.*` (not `com.ameide.*`).
- **Git-backed owners are first-class:** canonical state may live in Git (and GitLab is an internal storage surface).

## 0) Definitions (contract terms)

- **Primitive**: a deployable runtime unit (Domain / Process / Projection / Integration / Agent / UISurface). Polyglot runtimes are expected.
- **Owner**: the single authoritative write surface for some canonical state and its invariants.
  - In practice this is usually a Domain service.
  - Some non-domain primitives may own canonical state scoped to their runtime (e.g., process instance state), but that does not make them owners of business-domain truth.
- **Command**: request to an owner to change canonical state (often RPC; may be async).
- **Intent**: asynchronous request to an owner to change canonical state (message form).
- **Fact**: immutable statement emitted only by the owner after commit; used for propagation and derivations.
- **Derived read model**: rebuildable cache/index/view; never canonical truth.

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

## 3) Standard event plane (default in Kubernetes)

Kafka remains the standard event plane in Kubernetes:

- CloudEvents v1.0 envelope for metadata,
- Protobuf payloads for schema and codegen,
- Kafka (Strimzi) for transport/durability (topics + consumer groups).

This standard is intentionally not “Kafka only”: RPC is allowed where it is the right tool, as long as the semantics in §1 hold and interfaces remain proto-governed.

## 4) CloudEvents envelope (mandatory when using the event plane)

All facts/intents carried over the event plane MUST be valid CloudEvents v1.0. When using Kafka binary mode:

- CloudEvents attributes are carried as message headers,
- CloudEvents `data` is the Kafka record value (Protobuf bytes),
- payload metadata must not be embedded in the Protobuf payload.

Required attributes:

- `specversion = "1.0"`
- `id`
- `source`
- `type`

Required extensions (platform):

- `tenantid` (required for all inter-primitive traffic)
- `correlationid` (required where a workflow/process needs cross-event joins)
- `causationid` (required where event chains must be traceable)

## 5) Naming posture (`io.ameide.*`) (normative)

This v6 standard adopts `io.ameide.*` as the canonical semantic identity namespace.

- CloudEvents `type` MUST use `io.ameide.*` (not `com.ameide.*`).
- Zeebe message names (when mapping CloudEvents to Zeebe) SHOULD use `messageName == ce.type` to keep semantics consistent and verifiable.

Recommended type shapes:

- Facts: `io.ameide.<context>.fact.<name>.v<major>`
- Intents: `io.ameide.<context>.intent.<name>.v<major>`

This aligns with the process/runtime posture in `backlog/520-primitives-stack-v6.md`.

## 6) Publishing rules (outbox generalized)

### 6.1 DB-backed owners: outbox is mandatory for facts

- Owners MUST persist state and outbox rows in one transaction.
- A dispatcher publishes outbox rows to the event plane.

### 6.2 Git-backed owners: “outbox-equivalent” is mandatory for facts

Git-backed owners MUST provide a durable, replay-safe emission mechanism:

- Record audit pointers (commit SHA / MR id / pipeline id) durably.
- Emit facts only after the Git change is committed/merged/tagged.
- Ensure facts are replayable and idempotent (dedupe on `ce.id` and/or owner-issued idempotency keys).

See `backlog/694-elements-gitlab-v6.md` for the Git-backed Enterprise Repository posture.

## 7) Consumers (at-least-once reality; idempotency is mandatory)

- Delivery is at-least-once; consumers MUST treat replays as normal.
- Consumers MUST be idempotent; acceptable strategies include:
  - an inbox table keyed by `ce.id`,
  - upsert-by-natural-key with monotonic version checks where applicable,
  - replayable projections that can be rebuilt from facts (or from canonical Git + audit pointers where explicitly defined).

Partitioning SHOULD be deterministic when ordering matters (e.g., by aggregate id).

## 8) What these principles imply for each primitive type

This section exists to prevent “EDA principles” from being interpreted only through one primitive’s lens.

### 8.1 Domain primitives (owners of business truth)

- Domain primitives are the default owners for business-domain canonical state.
- All writes happen through Domain APIs/commands; Domains emit facts after commit (outbox or outbox-equivalent).
- Domains may be Git-backed owners; in that case, only Domains are permitted to execute canonical Git writes.

### 8.2 Process primitives (workflow/orchestration runtimes)

- Process primitives own orchestration state (process instance variables, timers, retries, human task progression).
- Process primitives MUST NOT write business-domain canonical state directly; they request changes from owners via commands (RPC) or intents.
- Process primitives may span multiple owners; this is normal (Saga / process manager).
- Workflow-engine messages used for wait-state resumption are orchestration mechanics; treat them as process-internal unless explicitly modeled as owner facts.

### 8.3 Projection primitives (derived read models)

- Projection primitives never own canonical business state.
- They consume owner facts to build derived views: search indexes, graphs, timelines, materialized query models.
- When canonical truth is Git-backed, projections may index Git content, but must treat it as derived (rebuildable) and remain anchored to owner audit pointers.

### 8.4 Integration primitives (external adapters)

- Integration primitives bridge external systems (webhooks, polling, connectors) into owner-facing commands/intents.
- They MUST be idempotent on inbound external events and MUST NOT mutate canonical business state except by calling the owner’s API.

### 8.5 Agent primitives (non-deterministic proposal runtimes)

- Agents produce proposals, drafts, and evidence; they do not become owners of business truth.
- Agents request changes from owners via commands/intents and attach evidence outputs (diffs, summaries, test results).
- Do not assume “an agent belongs to one domain”; agents may be reused across contexts.

### 8.6 UISurface primitives (interactive entry points)

- UISurfaces call owner APIs for commands and query services for reads.
- UISurfaces do not publish facts; facts come from owners after commit.

## 9) Relationship to workflow runtimes (Zeebe/Temporal)

- Zeebe/Temporal are orchestration runtimes for Process primitives; they are not the EDA backbone.
- Owners emit facts on the event plane; process workers consume those facts (or receive callbacks) to progress workflows.
- RPC between process workers and owners is allowed and often preferred for command issuance; use request→wait→resume for long-running work.

## 10) Cross-references and superseded guidance

### 10.1 Superseded documents (normative)

- `backlog/496-eda-principles-v2.md` (Knative Broker/Trigger posture)
- `backlog/496-eda-principles-v3.md` (Kafka-only posture; “topic == type” as a hard rule)
- `backlog/496-eda-principles-v4.md` (v6 replaces v4 with explicit `io.ameide.*` naming posture)

### 10.2 Related documents (interpret through v6)

- Runtime posture and naming: `backlog/520-primitives-stack-v6.md`
- Git-backed Enterprise Repository: `backlog/694-elements-gitlab-v6.md`
- Transformation v6 spine: `backlog/527-transformation-v6-index.md`

