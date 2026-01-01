# Protobuf Contracts for Event-Driven Architectures

**Ameide baseline standard (schema-first, Buf-enforced, CloudEvents/TraceContext-alignable)**

## Status

**Draft baseline** intended to unify and operationalize the rules currently spread across:

* `496-eda-principles.md` 
* `508-scrum-protos.md` 
* `509-proto-naming-conventions.md` 
* `523-commerce-proto.md` 
* `525-it4it-value-stream-mapping.md`

This document is written in a “contract standard” style, with **MUST/SHOULD/MAY** language.

---

## 1. Normative language

* **MUST** = required for conformance.
* **SHOULD** = recommended best practice; deviations need a documented rationale.
* **MAY** = optional.

---

## 2. Authoritative sources and workflow

### 2.1 Source of truth

1. **`.proto` files are the source of truth.** If any backlog prose contradicts `.proto`, the `.proto` wins. 
2. Backlog documents **MUST NOT** embed full copies of contracts (to prevent drift). Prefer “authoritative file list + invariants” (as done in 508). 

### 2.2 Schema-first requirement

* **All events and commands MUST be defined in Protobuf.** No ad‑hoc JSON event payloads. 

### 2.3 Buf as the enforcement tool

* **Buf MUST be used** for generation and compatibility enforcement (your docs already assume “enforced by Buf”). 
* **Breaking-change detection MUST run in CI** on every change to public protos. Buf’s breaking checker provides multiple strictness categories; **WIRE_JSON is the recommended minimum** because it checks both wire (binary) and JSON encoding compatibility. ([Buf][1])
* **Buf modules are the unit of publication.** A module is a versioned set of `.proto` files pushed together (BSR). A workspace can group multiple modules, but the module remains the compatibility boundary.
* **Layout MUST be Buf-friendly:**
  * directories SHOULD mirror package names;
  * the last package component MUST be a major version (`...v1`);
  * filenames MUST be `lower_snake_case.proto` (Buf STANDARD style expectations).

### 2.4 BSR / Confluent Schema Registry (CSR) compatibility (optional, but supported)

Buf/BSR does not define Ameide’s domain semantics (intent vs fact vs process), but it *does* provide authoritative guidance for schema hygiene and (on Enterprise) a CSR-compatible API.

When using BSR as a CSR-compatible registry:

* The CSR API is **read-only**; producers/serializers MUST NOT rely on auto-registration.
* Schema publication MUST happen via CI (e.g., `buf push`) and compatibility must be enforced there (`buf breaking`).
* Topic naming MUST be paired with a clear **CSR subject strategy**:
  * With Confluent’s default TopicNameStrategy, the subject is typically `<topic>-value`.
  * Ameide’s “one topic family ↔ one aggregator message” pattern maps cleanly to this.
* For schema-managed topics, the aggregator message SHOULD declare a CSR subject association via Buf’s Confluent extension option:
  * `option (buf.confluent.v1.subject) = { instance_name: "...", name: "<topic>-value" };`

---

## 3. Message taxonomy: what exists and why

Ameide’s EDA seam uses three primary classes:

1. **Domain intents** (commands over the bus): “please change state”
2. **Domain facts** (immutable state-change facts): “state changed”
3. **Process facts** (immutable orchestration/governance/timebox facts) 

The Scrum seam is explicitly defined this way (intents vs facts vs process facts) and mapped to exactly three topic families. 

### 3.1 Producer/consumer responsibility model

* **Domains** SHOULD accept commands by RPC and publish facts via outbox→broker. 
* **Process primitives** orchestrate the lifecycle and emit **process facts**; they MUST NOT “duplicate” domain facts on process topics. 

### 3.2 Classification rule: classify by ownership, not producer

A message MUST be classified by **whose state it is trying to change or describe**, not by who produced it (UI vs process vs agent).

* **Domain intent (command):** requests a change to domain-owned aggregate state (origin can be UI, agent, integration, or process).
* **Domain fact (event):** emitted only by the owning domain, after commit (outbox→broker).
* **Process fact (event):** emitted by the process runtime to describe orchestration state (timers/SLAs/work-available); UISurfaces observe these rather than being “commanded”.

Concrete examples (naming only):

* UI “post invoice” → domain intent `PostInvoiceRequested` (or RPC `PostInvoice`) → domain fact `InvoicePosted`.
* Process “needs human review” → process fact `HumanReviewRequested` / `ApprovalStepActivated` → human responds with a domain/process intent (e.g. `ApproveInvoiceRequested`, `CompleteHumanReviewRequested`).

---

## 4. Package and directory conventions

### 4.1 Package naming

All Ameide protos MUST use the single root namespace and major-versioned package suffix:

```proto
package ameide_core_proto.<bounded_context>[.<subcontext>].v<major>;
```



### 4.2 Directory layout

Protos MUST live under `packages/ameide_core_proto/src/ameide_core_proto/...` and be organized by bounded context + version. 

Scrum’s canonical source list is authoritative and non-negotiable. 

---

## 5. Topic naming and “runtime seam” mapping

### 5.0 Topic vs subject (Kafka/CSR precision)

In Kafka + CSR environments, distinguish:

* **Kafka topic**: the broker destination, e.g. `scrum.domain.facts.v1`
* **CSR subject**: the registry key for schema history, commonly `scrum.domain.facts.v1-value` under the default TopicNameStrategy

### 5.1 Topic naming pattern

Topic names (or stream/subject names) MUST encode:

* context
* class (`domain` or `process`)
* kind (`intents` or `facts`)
* major version

Pattern:

```text
<context>.<class>.<kind>.v<major>
```



### 5.2 1:1 mapping to aggregator message types

Topic families MUST map **1:1** to the “aggregator messages” (see §6). The Scrum mapping is canonical:

* `scrum.domain.intents.v1` → `...ScrumDomainIntent`
* `scrum.domain.facts.v1` → `...ScrumDomainFact`
* `scrum.process.facts.v1` → `...ScrumProcessFact` 

Commerce mirrors the same seam shape and mapping. 

### 5.3 Anti-duplication rule

* **Do not invent synonyms** (e.g., `scrum.events.v1`) for the same payload/topic family. 
* If you need multiple consumer groups, use broker constructs (consumer groups/partitions), not additional semantic topics. 

---

## 6. Aggregator messages: stable topic contracts

### 6.1 Required pattern

Each topic family MUST carry **one aggregator message** that contains:

* a shared envelope/meta field
* a `oneof` payload for the intent/fact variants

Example pattern (Scrum):

* `ScrumDomainIntent { meta; oneof intent { ... } }`
* `ScrumDomainFact { meta; aggregate; oneof fact { ... } }`
* `ScrumProcessFact { meta; oneof fact { ... } }` 

Commerce adopts the same pattern. 

### 6.3 Buf/BSR CSR subject association (if applicable)

If the platform treats a topic family as “schema-managed” via a CSR-compatible API, the corresponding aggregator message SHOULD declare its subject explicitly using the Buf Confluent extension option so the topic/subject↔message contract is unambiguous.

### 6.2 Naming suffix rules

* **Domain intents:** `…Requested` 
* **Domain facts:** past tense / `Defined|Created|Recorded|Updated` 
* **Process facts:** explicit governance/timebox vocabulary 
* Avoid generic CRUD verbs (“UpdateX”, “SetStatus”) for new contracts. 

---

## 7. Envelope and identity invariants

### 7.1 The “same semantics everywhere” rule

All contexts MUST carry a consistent envelope shape and semantics (even if each context defines its own `*MessageMeta` type). 

Required envelope concepts across contexts:

* **Idempotency key:** `message_id` (globally unique) 
* **Tenant isolation:** `tenant_id` (required) 
* **Correlation/causation:** `correlation_id`, `causation_id` 
* **Producer timestamp:** `occurred_at` 
* **Producer identity:** `producer` 
* **Schema marker:** `schema_version` 
* Optional attribution and routing:

  * `subject` (routing keys) 
  * `actor` (who caused the change) 

`message_id` is the canonical idempotency key name across Ameide contracts.

### 7.2 Aggregate identity and ordering

For **domain facts**, the envelope MUST carry an aggregate reference with a monotonic per-aggregate `version` for idempotent downstream consumption. 

Commerce’s proposal uses:

```proto
message CommerceAggregateRef {
  string type = 1;
  string id = 2;
  int64 version = 3; // monotonic per aggregate
}
```



### 7.3 Partitioning guidance

* For ordering/partitioning in brokers, **prefer `{tenant_id, aggregate_id}` keys** (domain-specific). 

---

## 8. CloudEvents and Trace Context alignment

Ameide does not have to “emit CloudEvents everywhere”, but **envelopes SHOULD be mappable losslessly** to CloudEvents and W3C trace context to simplify cross-broker/HTTP bridging (as already proposed for Commerce). 

### 8.1 CloudEvents baseline

CloudEvents is a CNCF-hosted interoperability spec with multiple formats and protocol bindings, including a Protobuf event format. ([GitHub][2])

CloudEvents Protobuf representations (as used by Google Eventarc, for example) include required context attributes like `id`, `source`, `spec_version`, and `type`. ([Google Cloud][3])

**Recommended mapping (internal → CloudEvents):**

* `meta.message_id` → `ce-id`
* `meta.producer` (or derived producer URI) → `ce-source`
* `ce-type` derived from the concrete payload variant (e.g., oneof case / fully-qualified message name)
* `meta.subject.*` (or aggregate identity) → `ce-subject`
* `meta.occurred_at` → `ce-time`

Commerce already proposes these mapping targets explicitly. 

### 8.2 W3C Trace Context

If you propagate trace context in the message envelope (or headers), use W3C `traceparent` / `tracestate` semantics. The W3C Trace Context spec standardizes these headers and their allowed mutations. ([W3C][4])

**Privacy/security requirement:** `traceparent`/`tracestate` MUST NOT contain personally identifiable or otherwise sensitive information. ([W3C][4])

Commerce includes `traceparent` and `tracestate` fields in the meta for cross-transport propagation. 

### 8.3 OpenTelemetry guidance for async boundaries

OpenTelemetry’s messaging conventions emphasize that context must be propagated so consumer traces can be correlated with producer traces, and recommend attaching a “message creation context”. ([OpenTelemetry][5])

Your `496` already reflects this operationally by showing span linking patterns. 

---

## 9. Schema evolution and compatibility

### 9.1 Protobuf evolution rules (wire safety)

At minimum, contracts MUST follow Protobuf compatibility rules:

* Field numbers **cannot be changed** once in use (changing a field number is equivalent to deleting and re-adding). ([Protocol Buffers][6])
* Deleted fields’ **numbers and names MUST be reserved** to prevent reuse. ([Protocol Buffers][6])
* Enum deletions SHOULD reserve removed numeric values/names (to avoid corruption/privacy bugs). ([Protocol Buffers][6])

These match the rules you already state in `496`/`509` (never remove/rename fields, never change field numbers, add new fields with new numbers, use `reserved`). 

### 9.2 Versioning: package/topic major is the breaking boundary

Breaking changes MUST be handled by:

* New major proto package (`…v1` → `…v2`)
* New logical topics (`…v1` → `…v2`)
* A migration plan/backlog describing coexistence and consumer upgrade 

### 9.3 Buf breaking policy

* CI SHOULD run `buf breaking` with at least `WIRE_JSON`. Buf documents that `WIRE_JSON` checks breakage in both wire and JSON encoding and is “recommended minimum”. ([Buf][1])

### 9.4 Version vocabulary (avoid ambiguity)

To avoid “version” meaning three different things, use this vocabulary:

* **Package version**: the `v1` suffix in `package ...v1;` (semantic breaking boundary).
* **Module version**: the BSR module commit/label for the pushed module (distribution boundary).
* **Subject version**: the CSR subject’s version number/history (registry boundary).

**Example (`buf.yaml`)** (illustrative — adopt to your repo conventions):

```yaml
version: v2
breaking:
  use:
    - WIRE_JSON
lint:
  use:
    - DEFAULT
```

(Choose stricter categories like `FILE` if you want generated-code compatibility enforcement at the file level.) ([Buf][7])

---

## 10. Reliability: delivery guarantees, outbox, idempotency

### 10.1 Transactional outbox is mandatory

Aggregate updates and event publishing MUST happen in a single DB transaction: **write domain state + outbox record atomically**, then relay to the broker. 

This is a widely documented industry pattern (“Transactional outbox”) to avoid dual-write inconsistencies without 2PC. ([microservices.io][8])

### 10.2 At-least-once is the default assumption

Outbox relays can publish more than once (e.g., crash between publish and marking sent), so **consumers MUST be idempotent** and deduplicate using message IDs. ([microservices.io][8])

### 10.3 Ordering expectations

Where ordering matters:

* Preserve per-aggregate ordering using `{tenant_id, aggregate_id}` partition keys (recommended) 
* Require monotonic `aggregate.version` in facts so consumers can:

  * ignore duplicates
  * detect gaps
  * apply in order 

---

## 11. Operational requirements: logging, metrics, DLQ

### 11.1 Structured logs

Event handlers SHOULD log:

* event/message id
* event type
* tenant id
* correlation id / trace id
* outcome and latency

`496` already shows this pattern. 

### 11.2 DLQ + observability

DLQ depth metrics and structured logs are part of the baseline operational posture. 

---

## 12. Documentation: AsyncAPI as the event catalog surface

### 12.1 Use AsyncAPI for event-driven API documentation

If you publish an event catalog, **AsyncAPI is a common industry standard** for event-driven APIs.

Important constraint: AsyncAPI specifies that **non‑JSON-based schemas (like Protobuf) MUST be inlined as a string** in the schema field. ([AsyncAPI][9])

### 12.2 Minimal AsyncAPI payload example (Protobuf)

```yaml
payload:
  schemaFormat: application/vnd.google.protobuf;version=3
  schema: |
    // paste the full .proto schema (or the relevant subset) as a string
    syntax = "proto3";
    package ameide_core_proto.commerce.v1;
    message CommerceDomainFact { /* ... */ }
```

(AsyncAPI references are strict about schemaFormat matching and inlining rules.) ([AsyncAPI][9])

---

## 13. 12‑Factor and cloud-native basics (applies to EDA handlers)

Event-driven services SHOULD follow 12-factor methodology (config via env, stateless processes, logs as event streams, etc.). 
This aligns with the canonical Twelve-Factor App guidance. ([Twelve-Factor App][10])

---

## 14. Concrete “Ameide seam” examples

### 14.1 Scrum seam (baseline)

The canonical Scrum seam is exactly:

* `scrum.domain.intents.v1`
* `scrum.domain.facts.v1`
* `scrum.process.facts.v1` 

And process primitives must follow the intent/fact split and avoid re-emitting domain facts on process topics. 

### 14.2 Commerce seam (CloudEvents + TraceContext alignable)

Commerce proposes:

* explicit topic families
* scope (tenant + site/channel/location)
* meta fields that map to CloudEvents and W3C trace context 

---

## 15. Contract review checklist (copy/paste)

**Naming & topology**

* [ ] Message is correctly classified: domain intent / domain fact / process fact. 
* [ ] Topic name matches `<context>.<class>.<kind>.v<major>` and maps 1:1 to aggregator type. 
* [ ] New events follow suffix rules (`…Requested`, past tense, governance vocabulary). 

**Envelope**

* [ ] `tenant_id` present and required. 
* [ ] `message_id` is globally unique and used for idempotency. 
* [ ] `correlation_id` / `causation_id` semantics are correctly set. 
* [ ] Facts include aggregate ref with monotonic version. 
* [ ] If trace context is propagated, it follows W3C and contains no PII. ([W3C][4])

**Schema evolution**

* [ ] No field numbers changed; no removed fields without reserving names/numbers. ([Protocol Buffers][6])
* [ ] Buf breaking check passes (WIRE_JSON minimum). ([Buf][1])
* [ ] Breaking changes handled by new package/topic major and migration plan. 

**Reliability**

* [ ] Events are published via transactional outbox (no dual write). 
* [ ] Consumers are idempotent (dedupe by message id). ([microservices.io][8])

**Docs**

* [ ] AsyncAPI (or equivalent catalog) updated; Protobuf schema inlined as string if using AsyncAPI. ([AsyncAPI][9])

---

## Industry standards and authoritative references (external)

* Protocol Buffers field numbering + reserving deleted fields/names; reserved enum values guidance. ([Protocol Buffers][6])
* Buf breaking-change detection categories and the recommended minimum `WIRE_JSON`. ([Buf][1])
* CloudEvents specification project (CNCF), including Protobuf event format availability. ([GitHub][2])
* CloudEvents Protobuf shape (example reference docs showing required context attributes). ([Google Cloud][3])
* W3C Trace Context recommendation + privacy requirements. ([W3C][4])
* OpenTelemetry messaging semantic conventions (context propagation across async messaging). ([OpenTelemetry][5])
* Transactional outbox pattern (industry pattern reference). ([microservices.io][8])
* Twelve-Factor App methodology. ([Twelve-Factor App][10])
* AsyncAPI specification: non‑JSON schemas (Protobuf) must be inlined as strings. ([AsyncAPI][9])

---

If you want, I can also produce a **“drop-in” `buf.yaml` + CI checklist** aligned to this standard (WIRE_JSON breaking, lint defaults, and a suggested module structure), but the standard above is already fully written with the industry references preserved.

[1]: https://buf.build/docs/breaking/rules/?utm_source=chatgpt.com "Rules and categories - Buf Docs"
[2]: https://github.com/cloudevents/spec "GitHub - cloudevents/spec: CloudEvents Specification"
[3]: https://cloud.google.com/eventarc/docs/reference/publishing/rpc/io.cloudevents.v1?utm_source=chatgpt.com "Package io.cloudevents.v1  |  Eventarc  |  Google Cloud"
[4]: https://www.w3.org/TR/trace-context/?utm_source=chatgpt.com "Trace Context"
[5]: https://opentelemetry.io/docs/specs/semconv/messaging/messaging-spans/?utm_source=chatgpt.com "Semantic conventions for messaging spans | OpenTelemetry"
[6]: https://protobuf.dev/programming-guides/editions/?utm_source=chatgpt.com "Language Guide (editions) | Protocol Buffers Documentation"
[7]: https://buf.build/docs/breaking/?utm_source=chatgpt.com "Breaking change detection - Buf Docs"
[8]: https://microservices.io/patterns/data/transactional-outbox?utm_source=chatgpt.com "Pattern: Transactional outbox"
[9]: https://www.asyncapi.com/docs/reference/specification/v3.0.0?utm_source=chatgpt.com "3.0.0 | AsyncAPI Initiative for event-driven APIs"
[10]: https://12factor.net/?utm_source=chatgpt.com "The Twelve-Factor App"
