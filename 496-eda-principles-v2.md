# 496 – Event-Driven Architecture (EDA) Principles

## Kubernetes Standard: Protobuf + CloudEvents + Knative Kafka Broker + Strimzi

> **Superseded:** replaced by `backlog/496-eda-principles-v6.md` (`io.ameide.*` naming posture + hybrid messaging; owner-writes; Kafka remains the standard event plane).

**Status:** Active Standard (replaces `496-eda-principles.md`)  
**Audience:** Platform engineering, domain teams, AI agents, architects  
**Scope:** **All inter-primitive (microservice-to-microservice) messaging inside the Kubernetes cluster; not primitive-internal flows.**

**Out of scope:** flows that stay within a single primitive boundary (same deployable/service), including in-process events, internal queues, and workflow/job runtime internals.

## Cross-references (docs that depend on this standard)

The following backlogs reference EDA rules and should be treated as dependent on this document:

- Architecture foundations: `470-ameide-vision.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `476-ameide-security-trust.md`
- Proto conventions: `509-proto-naming-conventions.md` (naming + legacy envelope guidance)
- CLI enforcement: `484a-ameide-cli-primitive-workflows.md`, `484b-ameide-cli-proto-contract.md`
- Scaffolding patterns: `510-domain-primitive-scaffolding.md`, `511-process-primitive-scaffolding.md`
- Capability specs that cite EDA invariants: `523-commerce*.md`, `527-transformation*.md`, Kanban backlogs (`614-620`)

If any of the above documents contradict this spec (e.g., treating CloudEvents as optional, or embedding message metadata in Protobuf as the primary envelope), this spec wins for **inter-primitive Kubernetes traffic**.

## 0. Non-negotiable standard (no alternatives)

Ameide has exactly one inter-service messaging stack in Kubernetes:

* **Protobuf** for payload schema and codegen
* **CloudEvents v1.0** for metadata envelope
* **Knative Eventing (Kafka Broker)** for routing (Brokers + Triggers)
* **Kafka operated by Strimzi** as durability layer under Knative

Services **publish only to a Knative Broker** and **receive only via Knative Trigger delivery**. Direct Kafka topic coupling is forbidden as an integration contract.

This document defines what **MUST** be true for a service to be considered compliant.

---

## 1. Terminology

* **Primitive**: a deployable microservice (Domain, Process, Projection, Integration, Agent, UISurface).
* **Intent**: a command (request) addressed to exactly one owning write surface.
* **Fact**: an event (immutable statement) emitted only by the owner *after commit*.
* **Owner**: the primitive that is the single writer for an aggregate/process state.
* **Broker**: Knative resource that receives CloudEvents and routes them to subscribers via Triggers.
* **DLQ**: dead-letter sink receiving events that failed delivery after configured retries.

---

## 2. Message contract (normative)

### 2.1 CloudEvents envelope is mandatory

All inter-primitive messages **MUST** be valid **CloudEvents v1.0**. The required CloudEvents attributes are `id`, `source`, `specversion`, and `type`. ([GitHub][1])

**Required attributes (MUST):**

* `specversion = "1.0"`
* `id` (globally unique; see §2.4) ([GitHub][1])
* `source` (stable logical producer identity) ([GitHub][1])
* `type` (contract identifier; see §2.5) ([GitHub][1])

**Required extensions (MUST):**

* `tenantid`
* `correlationid`
* `causationid`
* `traceparent`
* `tracestate`

**Required payload descriptors (MUST):**

* `datacontenttype = "application/protobuf"`
* `dataschema` (see §3.3)

**Recommended (SHOULD):**

* `time` (UTC timestamp of occurrence or recording)
* `subject` (aggregate/process instance identity)

### 2.2 No metadata duplication inside Protobuf

Metadata **MUST NOT** be embedded in the Protobuf message (no `MessageMetadata` inside payload). CloudEvents is the single source of truth.

**Migration:** If legacy payload metadata exists, consumers MAY accept it temporarily, but MUST prefer the CloudEvents envelope.

### 2.3 Transport encoding: HTTP binary mode

All publishers and consumers **MUST use CloudEvents HTTP binary mode**:

* CloudEvents attributes/extensions are sent as `ce-` headers
* HTTP body is raw protobuf bytes
* HTTP `Content-Type` header represents `datacontenttype` (do **not** send `ce-datacontenttype`) ([CloudEvents HTTP Binding][2])

This is the simplest and most reliable way to carry protobuf bytes without JSON transcoding.

### 2.4 Idempotency key

* CloudEvents `id` **IS** the global idempotency key.
* Consumers **MUST** dedupe on `id`.
* `correlationid` and `causationid` **MUST NOT** be used for dedupe.

### 2.5 Type naming (one enforceable convention)

CloudEvents `type` is a stable API surface and MUST follow:

```
<org>.<bounded_context>.<kind>.<name>.v<major>
```

Examples:

* `ameide.scrum.fact.sprint.started.v1`
* `ameide.orders.intent.place-order.requested.v1`

Rules:

* `kind` MUST be `fact` or `intent`
* `v<major>` increments only for breaking payload changes
* Minor/patch changes are handled by Protobuf compatibility (field additions)

---

## 3. Protobuf schema standard (normative)

### 3.1 Schema-first (no ad-hoc JSON)

All payloads **MUST** be defined in `.proto` and published via your shared schema pipeline.

### 3.2 Versioning

* Protobuf packages MUST be versioned: `.../v1`, `.../v2`, etc.
* Breaking changes require a new package version and a new CloudEvents `type` major version.

### 3.3 `dataschema` convention (one standard)

`dataschema` MUST reference:

* the Buf module, and
* the fully qualified protobuf message name.

Recommended format:

```
buf://<org>/<module>/<package>.<Message>
```

Example:

```
buf://ameide/ameide-core/ameide_core_proto.orders.v1.OrderPlaced
```

### 3.4 Compatibility gates (Buf is mandatory)

All schema changes MUST pass:

* `buf lint`
* `buf breaking` against the mainline or BSR baseline ([Buf][3])

The platform CI MUST block merges that fail breaking-change checks.

---

## 4. Knative routing spine (normative)

### 4.1 Knative Kafka Broker persistence model (normative understanding)

Knative Kafka Broker stores incoming CloudEvents in Kafka using **binary content mode**:

* CloudEvents attributes/extensions → Kafka headers
* CloudEvents `data` → Kafka record value ([Knative][4])

### 4.2 Broker scoping (one rule)

**Standard rule:** one Broker per bounded context namespace.

* Each bounded context runs in its own Kubernetes namespace.
* Each namespace contains exactly one Broker named `ameide-broker`.

No other Broker topology is considered “standard”. Exceptions require platform governance and are documented elsewhere.

### 4.3 Trigger subscriptions

Consumers subscribe via **Triggers** that filter primarily on CloudEvents `type`.

* Triggers MUST filter by `type`.
* Triggers MAY also filter by `subject` for finer routing, but MUST NOT rely on Kafka topics/partitions as a contract.

### 4.4 Delivery policy (retry/backoff/DLQ)

Brokers and/or Triggers MUST specify a DeliverySpec with:

* `retry`
* `backoffPolicy` (`exponential` or `linear`)
* `backoffDelay` (ISO-8601 duration)
* `deadLetterSink` ([Knative][5])

Knative defines these delivery parameters for Brokers/Triggers via `spec.delivery`. ([Knative][6])

**Platform default (Broker-level):**

* `retry: 5`
* `backoffPolicy: exponential`
* `backoffDelay: PT0.5S`
* `deadLetterSink: <namespace>/ameide-dlq-sink`

(Triggers may override only with platform approval.)

### 4.5 Canonical Broker + Trigger YAML (example)

```yaml
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: ameide-broker
  namespace: orders
spec:
  delivery:
    retry: 5
    backoffPolicy: exponential
    backoffDelay: PT0.5S
    deadLetterSink:
      ref:
        apiVersion: v1
        kind: Service
        name: ameide-dlq-sink
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: orders-fact-orderplaced-to-projection
  namespace: orders
spec:
  broker: ameide-broker
  filter:
    attributes:
      type: ameide.orders.fact.order.placed.v1
  subscriber:
    ref:
      apiVersion: v1
      kind: Service
      name: orders-projection
```

---

## 5. Reliability invariants (non-optional)

### 5.1 Facts after commit: Transactional Outbox (MUST)

Any primitive that emits **facts** MUST use a **transactional outbox** (or strictly equivalent mechanism) to guarantee that:

* state commit and outbox record are in the same DB transaction
* publishing is performed by a dispatcher after commit

Publishing facts directly to the Broker from request-handling code is prohibited.

**Minimum outbox record (MUST store):**

* CloudEvents context (specversion, id, source, type, time, subject, extensions)
* payload bytes
* destination broker (logical) and routing hints (if any)
* publish status, attempt count, last error

### 5.2 Ingress intents: idempotent write surface (MUST)

All intent handlers (gRPC/HTTP write surfaces or broker-delivered intent consumers) MUST be idempotent on CloudEvents `id`.

### 5.3 Consumers: Inbox dedupe (MUST)

All consumers **MUST** be idempotent under at-least-once delivery (duplicates, retries, redelivery).

Consumers MUST implement one of the following idempotency strategies:

**Option A — Inbox dedupe (default; REQUIRED for side effects)**

Consumers that perform non-convergent side effects (external calls, emails, payments, writes that are not monotonic upserts) MUST implement **Inbox dedupe** backed by durable storage:

* Key: `(consumer_name, tenantid, ce_id)`
* Behavior: if duplicate, treat as success and ACK (return 2xx)

**Option B — Natural-key idempotency (allowed for convergent projections)**

Consumers that maintain convergent read models MAY implement idempotency by design, for example:

- upsert keyed by stable identity, and
- apply-only-if-newer using a monotonic aggregate/version field in the payload (or equivalent domain sequencing rule).

If Option B is used, duplicates MUST still be treated as success, and consumers MUST NOT rely on “best effort ordering” across different keys.

### 5.4 Partitioning and ordering

If ordering is required, producers MUST set the CloudEvents **partitioning extension** `partitionkey`. Knative Kafka Broker supports ordered delivery based on CloudEvents partitioning extension. ([Knative][4])

Rules:

* Ordering guarantees are only **within the same partitionkey**
* No cross-key ordering assumptions

---

## 6. Failure handling rules (Knative delivery semantics)

### 6.1 What counts as “failure”

A consumer indicates delivery failure by returning a non-2xx response. Knative will retry according to DeliverySpec, then send to DLQ. ([Knative][5])

### 6.2 Transient vs permanent failures

* **Transient (retryable):** return 5xx; rely on Knative retry/backoff.
* **Permanent (non-retryable):** return 2xx after recording the failure outcome and emitting a fact/alert (so Knative doesn’t hot-loop).

### 6.3 DLQ payload requirements

DLQ consumers/sinks MUST preserve:

* the original CloudEvents envelope
* the raw payload bytes
* delivery attempt metadata (attempt count, last status/error if available)

The platform MUST provide:

* dashboards/alerts on DLQ throughput and backlog
* a standard replay procedure (manual or automated) that re-injects events with a **new** CloudEvents `id` and links original via `causationid`.

---

## 7. Observability contract (MUST)

### 7.1 Trace propagation

`traceparent` and `tracestate` MUST be propagated unchanged across async boundaries (producer → broker → consumer). These fields are for tracing only and MUST NOT be used for dedupe.

### 7.2 Required logs (structured)

Every producer and consumer MUST log at INFO at least:

* `ce_id`, `ce_type`, `ce_source`
* `tenantid`, `correlationid`, `causationid`
* processing outcome (`success|duplicate|failed|dlq`)
* processing latency

### 7.3 Required metrics

Each primitive MUST emit Prometheus metrics:

* `events_published_total{type,source,tenantid}`
* `events_consumed_total{type,consumer,tenantid,outcome}`
* `event_processing_duration_seconds{type,consumer}`
* `event_delivery_failures_total{type,consumer,status}`
* `dlq_events_total{type,tenantid}`

Platform SHOULD also export Knative broker/trigger delivery metrics if available.

---

## 8. Multi-tenant isolation (MUST)

### 8.1 Tenant identity is mandatory

Every event MUST include `tenantid` and consumers MUST reject events without it.

### 8.2 Validation

Consumers MUST:

* establish tenant execution context from `tenantid` before any data access
* reject mismatched tenant processing in “single-tenant deployment mode” (if configured)

### 8.3 Storage isolation

Any consumer that writes to a database MUST ensure tenant isolation (e.g., RLS or explicit tenant predicates). The enforcement mechanism is implementation-specific, but the requirement is not.

### 8.4 Routing is not isolation

Filtering by Trigger is not sufficient for tenant isolation. It is an optimization, not a security boundary.

---

## 9. Security baseline (platform + service)

### 9.1 Kafka security (Strimzi)

Kafka clusters MUST be configured with authentication and authorization (ACLs). Strimzi supports configuring authorization/ACLs in the Kafka custom resource and managing users via KafkaUser resources. ([Strimzi][7])

### 9.2 Network boundaries

Broker ingress and subscribers MUST be cluster-internal only (NetworkPolicy / mesh policy). External publication is only via an explicitly approved gateway.

---

## 10. Platform responsibilities (MUST)

Platform engineering MUST provide:

1. Strimzi-managed Kafka clusters (HA, storage, upgrades)
2. Knative Eventing with Kafka Broker enabled
3. A standard Broker (`ameide-broker`) per bounded-context namespace
4. Default delivery policy + DLQ sink wired at Broker level ([Knative][8])
5. A standard “event SDK” for services to publish/consume CloudEvents binary-mode protobuf
6. CI templates enforcing Buf lint + breaking checks ([Buf][3])
7. Standard dashboards/alerts for delivery failures and DLQ

---

## 11. CLI / Operator enforcement (MUST)

The platform CLI (`ameide primitive verify`) MUST validate at minimum:

**CloudEvents conformance**

* required attributes present (`id/source/type/specversion`) ([GitHub][1])
* required extensions present (`tenantid/correlationid/causationid/traceparent/tracestate`)
* `datacontenttype=application/protobuf`
* `dataschema` present and matches Buf conventions

**Publishing guarantees**

* outbox table exists and is written in same transaction as state changes (static checks + runtime probe)
* no direct Broker publish in domain transaction path

**Consumer guarantees**

* consumer idempotency strategy is present (Inbox dedupe OR convergent upsert) and is used
* duplicate handling returns success (2xx)
* missing tenantid is rejected

**Knative config**

* Trigger filters include `type`
* Broker/Trigger delivery policy configured (retry/backoff/DLQ) ([Knative][5])

**Schema governance**

* Buf lint + breaking checks are configured in CI ([Buf][3])

---

## 12. End-to-end delivery model (canonical flow)

```text
Intent (client/agent/process) → (HTTP binary CloudEvent) → Broker
  → Trigger → Owning write surface (idempotent on ce_id)
  → DB transaction commits state + outbox row (same tx)
  → Outbox dispatcher → Broker (Fact)
  → Trigger → Consumers (each uses inbox dedupe on ce_id)
  → on failure: Knative retries → then DLQ sink
```

---

## 13. Delivery plateaus (one path forward)

## Plateau 0 — E2E MVP (works end-to-end in one namespace)

**Goal:** prove the standard works with one intent → one fact → one projection.

**Infrastructure (MUST):**

* Strimzi Kafka cluster (dev grade)
* Knative Eventing + Kafka Broker
* Namespace with `Broker/ameide-broker`
* DLQ sink Service (simple “store and log”)

**Service (MUST):**

* Producer sends CloudEvents HTTP binary mode (protobuf body)
* Consumer receives via Trigger and logs envelope + payload

**Tooling (MUST):**

* Buf lint runs in CI

**Exit criteria:**

* A single message path demonstrates at-least-once delivery and basic routing.

## Plateau 1 — Production Minimum (reliability + DLQ + compatibility)

**Goal:** no data loss and safe evolution.

Adds (MUST):

* Transactional outbox for all fact emitters
* Outbox dispatcher with retries and backoff
* Inbox dedupe for all consumers
* Broker/Trigger DeliverySpec (retry/backoff/DLQ) ([Knative][5])
* Buf breaking-change gate in CI ([Buf][3])

Exit criteria:

* Kill -9 during publish does not lose facts (outbox proves durability)
* Duplicate deliveries do not cause duplicate effects
* Breaking schema changes are blocked

## Plateau 2 — Multi-tenant Usable (isolation + auditability)

Adds (MUST):

* tenantid required + validated everywhere
* tenant isolation enforced in storage writes
* partitionkey set for ordered aggregates where required ([Knative][4])
* per-tenant metrics slices and DLQ alerts

Exit criteria:

* Cross-tenant leakage tests pass
* Per-tenant operational visibility exists (metrics/logs)

## Plateau 3 — Operational Excellence (scale + governance)

Adds (MUST):

* standard replay tooling from DLQ (new `id`, link with `causationid`)
* SLOs for delivery latency and DLQ rate
* Strimzi authz (Kafka ACLs) managed as code ([Strimzi][7])
* runbooks for broker failure, consumer poison messages, schema rollback

Exit criteria:

* On-call runbooks + automation for common failure modes
* Controlled schema rollout and replay procedures

---

## 14. Explicit retirements (from v1)

The following are retired and must be removed from 496:

* broker selection matrix (NATS/Watermill/etc.)
* “command queue” as a first-class alternative transport
* “direct Kafka topic contract between services” as the default pattern

---

## 15. Legacy v1 principle mapping (for older backlogs)

Some backlogs still cite “`496-eda-principles.md` Principle N”. Use this mapping when updating those references:

- **Principle 1 (facts vs intents)** → §1, §2.5, §5.1
- **Principle 2 (schema-first)** → §3
- **Principle 3 (state + events atomicity)** → §5.1
- **Principle 4 (idempotency)** → §2.4, §5.2, §5.3
- **Principle 5 (eventual consistency)** → §12 (end-to-end flow) + §6 (failure semantics)
- **Principle 6 (clean boundaries)** → §5.1 + §10–§11 (platform/CLI enforcement)
- **Principle 7 (delivery guarantees)** → §0 + §4.4 + §6
- **Principle 8 (observability)** → §7
- **Principle 9 (12-factor/cloud-native)** → out of scope here; see `backlog/520-primitives-stack-v2.md`
- **Principle 10 (multi-tenant isolation)** → §8

---

[1]: https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md "CloudEvents v1.0 specification"
[2]: https://github.com/cloudevents/spec/blob/main/cloudevents/bindings/http-protocol-binding.md "CloudEvents HTTP protocol binding"
[3]: https://buf.build/docs/breaking/ "Buf breaking change detection"
[4]: https://knative.dev/docs/eventing/brokers/broker-types/kafka-broker/ "Knative Broker for Apache Kafka"
[5]: https://knative.dev/docs/eventing/event-delivery/ "Knative event delivery failures"
[6]: https://knative.dev/docs/eventing/reference/eventing-api/ "Eventing API - Knative"
[7]: https://strimzi.io/docs/operators/latest/configuring.html "Configuring Strimzi"
[8]: https://knative.dev/docs/eventing/configuration/broker-configuration/ "Configure Knative Broker defaults"
