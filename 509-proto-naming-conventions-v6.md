---
title: 509 – Proto Naming & Semantic Identity Conventions (v6: `io.ameide.*`)
status: draft
owners:
  - platform
created: 2026-01-18
supersedes:
  - 509-proto-naming-conventions-v2.md
  - 509-proto-naming-conventions.md
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
---

# 509 – Proto Naming & Semantic Identity Conventions (v6: `io.ameide.*`)

This v6 document is the current naming baseline for:

- proto package naming conventions (human and tool readability), and
- inter-primitive semantic identity conventions (CloudEvents `type`, Zeebe message names when mapped from facts),

under the integration posture in `backlog/496-eda-principles-v6.md`.

## 0) Principles

1) **`io.ameide.*` is the canonical semantic identity namespace.**  
Do not introduce `com.ameide.*` semantic identities in new work.

2) **Contracts are proto-governed.**  
RPC services, message payloads, and (where applicable) stable semantic identities are defined in protobuf and enforced by tooling.

3) **Delivery plane is not the contract.**  
Kafka topics and partitions are operational; semantic identity is the contract. Defaults may map `topic == ce.type`, but v6 does not require it as a hard rule.

## 1) Proto packages (convention)

Proto packages SHOULD follow one stable pattern across the codebase:

```proto
package io.ameide.<context>[.<subcontext>].v<major>;
```

Notes:

- `<context>` is a bounded context (e.g., `transformation`, `sales`).
- `<subcontext>` is optional and used when the context surface would otherwise become too broad.
- `<major>` is the compatibility major for the contract.

## 2) CloudEvents `type` (semantic identity)

CloudEvents `type` is the canonical semantic identity of an inter-primitive message.

v6 naming rules:

- Use a stable reverse-DNS prefix: `io.ameide.<context>...`
- Encode kind and major version in the type:
  - facts: `io.ameide.<context>.fact.<name>.v<major>`
  - intents: `io.ameide.<context>.intent.<name>.v<major>`

This aligns with Zeebe message mapping used by process primitives: when bridging facts into Zeebe, prefer `messageName == ce.type` so the diagram and the event contracts cannot drift.

## 3) Kafka topics (delivery plane)

Kafka is the standard event plane (per `backlog/496-eda-principles-v6.md`).

Allowed topic strategies:

- **Topic == CloudEvents `type`** (simple, high-clarity default; easy to reason about replay/backfill per message type).
- **Topic families** (operational grouping) where explicitly justified, with `ce.type` remaining the semantic identity.

In either case, consumers must treat `ce.id` as the idempotency key and tolerate replays.

