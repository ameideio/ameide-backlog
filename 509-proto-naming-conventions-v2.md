---
title: 509 – Proto Naming & Package Conventions (v2)
status: superseded
owners:
  - platform
created: 2026-01-13
superseded_by:
  - 509-proto-naming-conventions-v6.md
---

# 509 – Proto Naming & Package Conventions (v2)

> **Superseded:** replaced by `backlog/509-proto-naming-conventions-v6.md` (`io.ameide.*` semantic identity posture).

This document is the current naming baseline for proto contracts and event naming **under EDA v3 (Kafka + CloudEvents)**.
It supersedes `backlog/509-proto-naming-conventions.md` for anything that touches inter-primitive messaging.

## Canonical references

- EDA standard: `backlog/496-eda-principles-v3.md`
- v1 historical baseline: `backlog/509-proto-naming-conventions.md`

## Goals

- Keep proto package naming stable and predictable across primitives.
- Make event contracts explicit and vendor-honest for Kafka transport.
- Ensure “what the message means” (CloudEvents `type`) is not confused with “how it is carried” (Kafka topic mechanics).

## 1) Proto package naming (unchanged)

All Ameide protos use:

```proto
package ameide_core_proto.<bounded_context>[.<subcontext>].v<major>;
```

Use v1 for detailed examples; v2 focuses on the EDA-facing seams.

## 2) CloudEvents naming (contract surface)

CloudEvents `type` is the canonical semantic name of an inter-primitive message.

Naming rules:

- Use a stable reverse-DNS prefix: `com.ameide.<context>...`
- Encode kind and version in the type:
  - facts: `com.ameide.<context>.fact.<name>.v<major>`
  - intents/commands: `com.ameide.<context>.intent.<name>.v<major>`

## 3) Kafka topics (delivery plane)

Kafka is the delivery plane for CloudEvents.

Default topic rule (EDA v3):

- topic name == CloudEvents `type` (one topic per semantic message type), unless an explicit exception is documented in `496-eda-principles-v3.md`.

## 4) Protobuf payload naming

Payload messages should be readable without relying on transport metadata:

- Facts use `*Fact` suffix (or a `DomainFact`/`ProcessFact` envelope with a `oneof`).
- Intents use `*Intent` suffix (or a `DomainIntent` envelope with a `oneof`).
- Do not embed CloudEvents-like metadata inside the Protobuf payload for inter-primitive traffic; it lives in the CloudEvents envelope per `496-eda-principles-v3.md`.

## 5) Versioning rules

- The major version is expressed in:
  - the proto package suffix (`...v1`, `...v2`, …),
  - the CloudEvents `type` (`...v1`, `...v2`, …),
  - and therefore the default topic name.
- Do not add a second “schema version” field inside messages for normal evolution; treat changes as new major versions.
