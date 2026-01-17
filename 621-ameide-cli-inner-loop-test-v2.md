---
title: "621 – ameide inner-loop front doors (v2: 430-aligned)"
status: draft
owners:
  - platform
created: 2026-01-13
parent: 621-ameide-cli-inner-loop-test.md
supersedes:
  - 621-ameide-cli-inner-loop-test.md
---

# 621 – ameide inner-loop front doors (v2: 430-aligned)

This document exists to remove ambiguity created by older “Phase 3 via Telepresence” language.

## Normative contract (430v2)

Per `backlog/430-unified-test-infrastructure-v2-target.md`:

- `ameide test` runs **Phase 0/1/2/3** (contract → unit → integration → cluster E2E) and is the only “no-brainer” front door for agents.
- `ameide test ci` runs **Phase 0/1/2 only** (skip Phase 3), for environments where cluster access/secrets are intentionally unavailable.

## Why this split exists

- It keeps a single “no-brainer” front door for agents (`ameide test`) while still supporting CI without cluster access (`ameide test ci`).
- It prevents Phase 2 “integration” from silently becoming “cluster integration” by keeping Phase 3 explicit in the phase ordering and evidence.
