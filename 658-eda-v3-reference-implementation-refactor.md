---
title: 658 – EDA v3 Reference Implementation Refactor (Kafka + CloudEvents)
status: draft
owners:
  - platform
created: 2026-01-13
supersedes:
  - 658-eda-v2-reference-implementation-refactor.md
---

# 658 – EDA v3 Reference Implementation Refactor (Kafka + CloudEvents)

**Standard:** `backlog/496-eda-principles-v3.md`  
**Related:** `backlog/509-proto-naming-conventions.md`, `backlog/657-transformation-domain-clean-target.md`

## Purpose

EDA standards only “exist” if:

- a reference implementation demonstrates them end-to-end, and
- CLI/verify gates can enforce them.

This backlog updates the “reference implementation” plan to match **EDA v3**:
- Kafka topics are the runtime contract surface,
- CloudEvents is the mandatory envelope,
- Protobuf is the mandatory payload.

## What must be true (v3 reference loops)

1) **Domain facts**
- Domain writes state + outbox row in one transaction.
- Dispatcher publishes outbox rows to Kafka topics (topic = CloudEvents `type` by default).

2) **Projection ingestion**
- Projection consumes Kafka topics via consumer groups.
- Projection dedupes on CloudEvents `id` (durable inbox).

3) **Execution intents (tool runs / agent work)**
- Intents are delivered via Kafka topics (not via HTTP Broker delivery).
- Executors are Kafka consumers (ACK/commit offsets only after durable acceptance/outcome recording).

4) **Failure loop**
- Poison messages are handled via retries, DLQs, and replay procedures that preserve causation/correlation.

5) **Verification loop**
- `ameide primitive verify` and static audits can enforce:
  - CloudEvents metadata presence,
  - topic/type naming rules,
  - outbox + dispatcher presence in domains,
  - inbox + dedupe presence in consumers.

## Relationship to Zeebe processes

Zeebe orchestration is not the EDA spine, but it must integrate with it:

- Kafka facts can start/correlate Zeebe processes via either:
  - a Kafka→Zeebe bridge (Kafka consumer → Zeebe publish message), or
  - Camunda inbound Kafka connectors (if adopted as a platform runtime).

The reference implementation must pick one and make it operable and testable.

