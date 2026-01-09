---
title: 430v2 – Unified Test Infrastructure (Index)
status: active
owners:
  - platform
  - test-infra
created: 2026-01-09
updated: 2026-01-09
supersedes:
  - 430-unified-test-infrastructure.md
---

## Summary

430v2 defines the **target-state** testing contract for the Ameide monorepo:

- One front door: `ameide dev inner-loop-test`
- Strict phases: **Unit → Integration → E2E**
- Native tooling per language (Go/Jest/Pytest/Playwright)
- **No test “modes”** (`INTEGRATION_MODE` et al.) and **no per-component runner scripts** as the canonical execution path
- **JUnit XML is mandatory evidence** for every phase (including synthetic JUnit on early failures)

This file is an index so older backlogs can reference “430v2” without duplicating the contract.

## Canonical docs

- Target contract: `backlog/430-unified-test-infrastructure-v2-target.md`
- Refactoring inventory (docs that must be aligned): `backlog/430-unified-test-infrastructure-v2-refactoring-backlogs.md`
- Refactoring plan (repo implementation changes): `backlog/430-unified-test-infrastructure-v2-refactoring-test.md`

## Historical context

- Previous contract (kept for history only): `backlog/430-unified-test-infrastructure.md`

