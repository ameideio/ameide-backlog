# 496 – Event-Driven Architecture (EDA) Principles (v2)

This document is the **canonical EDA v2 contract** for Ameide.

It is intentionally aligned to the repo’s static conformance gate:

- `scripts/eda_v2_contract_audit.sh` (heuristic audit; enforced “hard-no” rules)

## 0) Scope and goal

EDA v2 defines the **inter-primitive** communication contract:

- Domain ↔ Process ↔ Integration ↔ Agent ↔ Projection ↔ UISurface communicate across stable seams.
- EDA is used **between primitives**, not as hidden “internal control flow” inside a single primitive.

The goal is to make production behavior **deployable, observable, and drift-resistant**:

- Domains remain systems of record.
- Facts are durable and replayable.
- Consumers are idempotent (at-least-once delivery reality).

## 1) The core decision: facts on the broker, commands via gRPC (no broker intents)

### 1.1 Facts (events) are broker-delivered

- Facts MUST be published to the broker (Kafka) using a transactional outbox (single-writer domains).
- Facts MUST be modeled as **one semantic fact == one protobuf message** (no “aggregator envelope” messages).
- Facts MUST be treated as immutable audit statements; they are not requests.

### 1.2 Commands/intents are gRPC-only and do not travel over EDA

EDA v2 explicitly forbids broker-delivered domain intents:

- Do not publish commands/intents to `<domain>.domain.intents.vN` topics.
- Do not assign CloudEvents “intent” `type` identifiers.

Commands/intents MUST be submitted via gRPC to the owning write surface.

### 1.3 Decoupling via a deployable Command Bus (gRPC)

To keep callers decoupled from domain addresses while preserving the “commands are gRPC” rule, Ameide deploys a **Command Bus**:

- A cluster service with a stable gRPC endpoint.
- It authenticates/authorizes callers.
- It routes commands to the owning primitive’s gRPC write surface.
- It enforces idempotency on a caller-supplied idempotency key.

The Command Bus is a deployment/runtime concern; it does not change message semantics:
commands remain commands (requests), and facts remain facts (statements).

## 2) Vendor-aligned reality: at-least-once

EDA v2 assumes **at-least-once delivery** everywhere that matters:

- broker delivery is at-least-once
- consumers MUST be idempotent

Idempotency MUST be explicit:

- Commands MUST carry an idempotency key (e.g., `request_id`).
- Facts MUST carry an idempotency key via the CloudEvents envelope (`ce-id`) and must be dedupable by consumers.

## 3) Protobuf requirements (audit-aligned)

### 3.1 Package and type namespace

All public Ameide protos MUST:

- use `package io.ameide.<...>.vN`
- use `option java_package = "io.ameide.<...>.vN"`
- never use `com.ameide.*` namespaces

### 3.2 Stable type identifiers for facts

All fact messages that cross a primitive boundary MUST declare the standard eventing option:

```proto
option (io.ameide.common.v1.eventing) = {
  stable_type: "io.ameide.<capability>.<sub>.fact.<EventName>.v1"
  stream_ref: "<capability>.<sub>.domain.facts.v1"
};
```

Rules:

- `stable_type` MUST start with `io.ameide.`
- `stable_type` MUST NOT use `.intent.` (intents are not broker-delivered in EDA v2)
- `stream_ref` MUST be set (the logical broker stream/topic family)

### 3.3 “No aggregator envelopes”

Avoid protobuf aggregator wrappers like:

- `*DomainFact` / `*ProcessFact` / `*DomainIntent` messages
- `oneof fact { ... }` / `oneof intent { ... }`

In EDA v2, the envelope is CloudEvents; the payload is a single protobuf message.

## 4) CloudEvents binding (facts only)

Facts are published as Kafka messages using **CloudEvents binary mode**:

- CloudEvents attributes carry the cross-cutting metadata (id, type, source, time, trace context).
- The Kafka message payload is the serialized protobuf bytes.

Minimum required attributes for facts:

- `ce-id` (idempotency key)
- `ce-type` (maps to `eventing.stable_type`)
- `ce-source` (producer identity)
- `ce-specversion`
- `ce-time`
- W3C trace context propagation (`traceparent`, optional `tracestate`)

Content type:

- `ce-datacontenttype` SHOULD be set to the protobuf content type used by the platform.

## 5) Topic taxonomy (facts only)

EDA v2 uses stable **fact streams** (topic families). Examples:

- `<capability>.<sub>.domain.facts.v1`
- `<capability>.<sub>.process.facts.v1` (optional; never reintroduce hidden control flow)
- `<capability>.<sub>.integration.facts.v1` (optional)

The platform does not define `<domain>.domain.intents.vN` topics as a supported seam in v2.

## 6) What the audit script enforces

`scripts/eda_v2_contract_audit.sh` enforces these “hard-no” rules:

- no `com.ameide.*` namespaces
- no CloudEvents “intent” stable type patterns
- all `stable_type` values begin with `io.ameide.`
- all proto packages under `packages/ameide_core_proto/...` use `package io.ameide...`

It also reports (non-enforced, informational) counts for:

- legacy intent topic references (`.domain.intents.vN`)
- presence/shape of `eventing` option blocks
- legacy “envelope message” patterns

