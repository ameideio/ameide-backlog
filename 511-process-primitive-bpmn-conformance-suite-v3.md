---
title: "511 – Process Primitive BPMN Conformance Suite (v3): Zeebe semantics + 430-aligned execution"
status: draft
owners:
  - platform
created: 2026-01-13
parent: 511-process-primitive-bpmn-conformance-suite-v2.md
supersedes:
  - 511-process-primitive-bpmn-conformance-suite-v2.md
---

# 511 – Process Primitive BPMN Conformance Suite (v3): Zeebe semantics + 430-aligned execution

This document resolves the remaining contradiction between:
- “cluster-first Zeebe semantics conformance”, and
- the repo-wide test contract in `backlog/430-unified-test-infrastructure-v2-target.md` (Phase 0/1/2 local-only).

## What stays the same (vendor truth)

- The conformance suite asserts **engine semantics** (deployability, jobs, messages, timers, incidents) against **Camunda 8 / Zeebe**.
- The runner uses the **Orchestration Cluster REST API** (not Operate/Tasklist APIs).
- Message tests are vendor-shaped:
  - buffering via TTL on publish,
  - idempotency via `messageId` while buffered,
  - avoid “correlate-message” when buffering is required.

## What changes in v3 (430 alignment)

### Conformance is not Phase 2 “integration”

Per `backlog/430-unified-test-infrastructure-v2-target.md`:
- Phase 2 is local-only and must not touch Kubernetes.

Therefore:
- Zeebe runtime conformance is executed under a **cluster-only test tag** (e.g. `//go:build cluster`) and a **separate CLI entrypoint** (not `inner-loop-test`, not `ci test`).

### What Phase 0/1/2 prove for processes

Phase 0/1/2 (local-only) must still provide strong guarantees:
- BPMN profile conformance (design-time).
- Worker coverage (job type extraction + handler registry coverage).
- Deterministic scaffolding/generation (no drift).

But they do **not** attempt to prove cluster runtime semantics.

### What cluster conformance proves (separate gate)

The cluster suite proves the engine truth:
- deploy resource,
- activate/complete/fail jobs,
- publish/correlate messages (buffering/uniqueness),
- timer semantics (“not earlier”),
- incident semantics (retries exhausted → incident observable).

## Entry points (proposed contract)

- Local contract/unit/integration:
  - `ameide test` (Phase 0/1/2)
- Cluster semantics smoke (separate):
  - `ameide test smoke` runs `go test -tags=cluster ...` and produces JUnit.
- Preview environment E2E (separate):
  - `ameide test e2e` runs Playwright against preview ingress URLs.

## Definition of Done (v3 conformance)

Done when:
- No cluster-dependent tests are selectable by the Phase 2 `integration` build tag.
- Cluster semantics can be executed deterministically via the dedicated smoke entrypoint and produces JUnit evidence.
- The conformance fixture covers message buffering/uniqueness and supports the Request → Wait → Resume template (see `backlog/511-process-primitive-scaffolding-v3.md`).
