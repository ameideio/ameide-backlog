---
title: 658 – EDA v6 Reference Implementation Refactor (Kafka + CloudEvents + hybrid)
status: draft
owners:
  - platform
created: 2026-01-18
supersedes:
  - 658-eda-v3-reference-implementation-refactor.md
  - 658-eda-v2-reference-implementation-refactor.md
related:
  - 496-eda-principles-v6.md
  - 509-proto-naming-conventions-v6.md
---

# 658 – EDA v6 Reference Implementation Refactor (Kafka + CloudEvents + hybrid)

**Standard:** `backlog/496-eda-principles-v6.md`  
**Naming:** `backlog/509-proto-naming-conventions-v6.md` (`io.ameide.*`)

## Purpose

EDA standards only “exist” if:

- a reference implementation demonstrates them end-to-end, and
- CLI/verify gates can enforce them.

This backlog updates the “reference implementation” plan to match v6:

- Kafka remains the standard event plane,
- CloudEvents is the mandatory envelope when using the event plane,
- Protobuf is the mandatory payload,
- RPC is allowed for commands (hybrid model),
- semantic identities use `io.ameide.*`.

## What must be true (v6 reference loops)

1) **Domain facts (after commit)**
- Owner performs canonical write (DB-backed or Git-backed).
- Owner emits facts only after commit (outbox / outbox-equivalent).
- Dispatcher publishes facts to Kafka with CloudEvents metadata.

2) **Projection ingestion**
- Projection consumes facts via Kafka consumer groups.
- Projection dedupes on CloudEvents `id` (durable inbox or equivalent).
- Projection is rebuildable from facts (and/or from canonical Git content where explicitly defined).

3) **Execution intents (tool runs / agent work)**
- Intents are delivered via Kafka topics (execution queues are allowed as an operational pattern).
- Executors are Kafka consumers (ACK/commit offsets only after durable acceptance and/or outcome recording back to the owner).

4) **Failure loop**
- Poison messages are handled via retries and DLQs (transport-specific), plus replay procedures that preserve causation/correlation.

5) **Verification loop**
- `ameide primitive verify` and static audits enforce:
  - CloudEvents metadata presence where required,
  - `io.ameide.*` naming posture,
  - outbox/outbox-equivalent in owners,
  - inbox/idempotency in consumers.

## Relationship to Zeebe processes

Zeebe orchestration is not the EDA spine, but it must integrate with it:

- Owner facts can start/correlate Zeebe processes via a Kafka→Zeebe bridge (Kafka consumer → Zeebe publish message), with `messageName == ce.type` as the default mapping.

