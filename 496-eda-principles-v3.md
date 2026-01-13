---
title: 496 – Event-Driven Architecture (EDA) Principles (v3: Kafka)
status: draft
owners:
  - platform
created: 2026-01-13
parent: 496-eda-principles-v2.md
supersedes:
  - 496-eda-principles-v2.md
---

# 496 – Event-Driven Architecture (EDA) Principles (v3: Kafka)

## Kubernetes Standard: Protobuf + CloudEvents + Kafka + Strimzi (no Knative Eventing)

**Status:** Active Standard (replaces `496-eda-principles-v2.md`)  
**Scope:** **All inter-primitive (microservice-to-microservice) messaging inside the Kubernetes cluster; not primitive-internal flows.**

This version standardizes the **runtime event plane** on **Kafka** and treats Kafka topics as a **first-class contract surface**. CloudEvents remains the required envelope; Protobuf remains the required payload schema.

## 0. Non-negotiable standard (no alternatives)

Ameide has exactly one inter-service messaging stack in Kubernetes:

* **Protobuf** for payload schema and codegen
* **CloudEvents v1.0** for metadata envelope
* **Kafka** as the transport and durability layer (topics + consumer groups)
* **Kafka operated by Strimzi** in-cluster

Services **publish to Kafka topics** and **consume from Kafka topics**. Knative Broker/Trigger delivery is not part of the v3 standard.

## 1. CloudEvents envelope (still mandatory)

All inter-primitive messages MUST be valid CloudEvents v1.0:

- required attributes: `specversion`, `id`, `source`, `type`
- required extensions: `tenantid`, `correlationid`, `causationid`, `traceparent`, `tracestate`
- required payload descriptors:
  - `datacontenttype = "application/protobuf"`
  - `dataschema` referencing the protobuf schema

CloudEvents `id` remains the global idempotency key; consumers must dedupe on `id`.

## 2. Kafka binding (contract)

### 2.1 Topic is part of the integration contract

Kafka topic names are a contract surface in v3. A service MUST NOT rely on “dynamic discovery” of topics by out-of-band agreements.

### 2.2 Topic naming (default mapping)

Default mapping rule:

- **Kafka topic name = CloudEvents `type`**

Rationale:
- a single stable identifier drives routing, documentation, and schema ownership.

If a bounded context needs topic-family aggregation (operational reasons), that exception must be explicitly documented and the mapping versioned.

### 2.3 Headers vs payload

Kafka message:

- key: MAY be set for partitioning (see §2.4); key is not a substitute for CloudEvents correlation/causation.
- headers: carry CloudEvents attributes/extensions (binding convention).
- value: raw protobuf bytes (no JSON transcoding).

### 2.4 Partitioning rule

Partitioning MUST be deterministic for correctness-sensitive streams:

- For facts/events: partition by `subject` (or aggregate id) when ordering matters per entity.
- For intents/commands: partition by the target entity id or command key.

## 3. Intent vs fact (ownership rules)

Unchanged from v2:

- **Intent**: request to change state, addressed to a single owning write surface.
- **Fact**: immutable statement emitted only by the owner after commit.

Domains remain single-writer; processes coordinate; projections derive read models from facts.

## 4. Outbox + dispatcher is mandatory for fact emission

Domain primitives MUST NOT publish to Kafka from request handlers/transactions directly.

Required flow:
- handler writes state + outbox row in one DB transaction
- dispatcher publishes outbox rows to Kafka
- dispatcher commits Kafka offsets only after durable outbox marking

## 5. Consumers: at-least-once is reality

Kafka delivery is at-least-once. Therefore:

- consumers MUST be idempotent (dedupe on CloudEvents `id`)
- consumers MUST treat replays as normal
- consumers MUST persist dedupe state in a durable inbox (per consumer group)

## 6. Relationship to Camunda/Zeebe processes

EDA remains between primitives. Zeebe is an orchestration runtime; Zeebe jobs/messages are not “the EDA spine”.

Kafka integration points:

- **Kafka → Zeebe**: an ingress bridge (or a Camunda inbound Kafka connector) consumes CloudEvents and publishes Zeebe messages to start/correlate BPMN instances.
- **Zeebe → Kafka**: process workers emit intents to domains and domains emit facts via outbox → Kafka.

## 7. Migration note

Documents and implementations that assume Knative Broker/Trigger delivery must be migrated to Kafka consumer groups and topic contracts.

