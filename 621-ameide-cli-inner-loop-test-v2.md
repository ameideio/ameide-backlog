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

- `ameide dev inner-loop-test` runs **Phase 0/1/2 only** (local-only; no Kubernetes/Telepresence).
- `ameide ci test` runs **Phase 0/1/2 only** (local-only; no Kubernetes/Telepresence).
- Deployed-system verification is split:
  - `ameide ci e2e`: Playwright-only against preview ingress URLs.
  - `ameide ci smoke` (name TBD): cluster-only smokes for runtime semantics (e.g., Zeebe conformance), not part of Phase 2.

## Why this split exists

- It keeps the “no-brainer” front doors stable (local-only, deterministic).
- It prevents Phase 2 “integration” from silently becoming “cluster integration”.
- It makes “diagram must not lie” runtime checks possible without breaking the local contract.

