---
title: "621 – ameide inner-loop front doors (v2: 430-aligned)"
status: active
owners:
  - platform
created: 2026-01-13
updated: 2026-01-20
parent: 621-ameide-cli-inner-loop-test.md
supersedes:
  - 621-ameide-cli-inner-loop-test.md
---

# 621 – ameide inner-loop front doors (v2: 430-aligned)

This document exists to remove ambiguity created by older “Phase 3 via Telepresence” language.

## Normative contract (430v2)

Per `backlog/430-unified-test-infrastructure-v2-target.md`:

- `ameide test` runs **Phase 0/1/2 only** (local-only; no Kubernetes/Telepresence):
  - Phase 0: contract (vendor-driven discovery; fail fast)
  - Phase 1: unit (local-only, pure)
  - Phase 2: integration-local (local-only; mocked/stubbed only)
- `ameide test cluster` runs **Phase 3/4 only** (cluster-only verification; requires Kubernetes):
  - Phase 3: integration-cluster (cluster smokes / runtime semantics; `//go:build cluster`)
  - Phase 4: Playwright E2E against the deployed platform URL read from `AUTH_URL` in `ConfigMap/www-ameide-platform-config`.

## Why this split exists

- It keeps the “no-brainer” front doors stable (local-only, deterministic).
- It prevents Phase 2 “integration” from silently becoming “cluster integration”.
- It makes “diagram must not lie” runtime checks possible without breaking the local contract.

## Implementation status

This posture is now implemented as first-class CLI front doors:

- `ameideio/ameide#582`: Phase 0/1/2 (`ameide test`) front door
- `ameideio/ameide#589`: Playwright E2E runner (now executed as Phase 4 of `ameide test cluster`)
