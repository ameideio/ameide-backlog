# 591 — Capability Tests: Integration Packs at the Capability Boundary

**Status:** Draft (target-state direction)  
**Audience:** Capability owners, platform engineering, test-infra  
**Scope:** Define how to write and run **capability-owned tests** (vertical slice tests spanning primitives), aligned with `backlog/430-unified-test-infrastructure-v2-target.md`.

> **Update (2026-01): 430v2 contract**
>
> 430v2 removes `INTEGRATION_MODE` and per-component `run_integration_tests.sh` packs as the canonical execution path. This document keeps the old “pack” language for historical context but should be read through the v2 lens:
> - Phase 2 Integration is local-only and mocked/stubbed.
> - Phase 3 E2E is cluster-only and Playwright-only.

---

## Use with

- Repo-wide test target state: `backlog/430-unified-test-infrastructure-v2-target.md`
- Repo-wide test entrypoint: `backlog/468-testing-front-door.md`
- Primitive testing discipline (mandatory invariants): `backlog/537-primitive-testing-discipline.md`
- Capability repo hierarchy (where these packs live): `backlog/590-capabilities.md`
- WorkRequests substrate + execution queues (when applicable): `backlog/527-transformation-capability.md`, `backlog/585-keda.md`, `backlog/586-workrequests-execution-queues-inventory.md`, `backlog/587-kafka-topics-and-queues-inventory.md`

---

## 1) Why capability tests exist (and what they are not)

### 1.1 Capability tests (vertical slice)

Capability tests validate **end-to-end behavior** across primitives (Domain/Process/Projection/Integration/UISurface/Agent) for a single capability.

They answer questions like:

- “Does the capability’s workflow actually complete?”
- “Do we produce durable evidence and an auditable timeline?”
- “Do the contracts (proto + topics + RPC surfaces) work together?”

### 1.2 What capability tests do NOT replace

- **Primitive unit/invariant tests** remain mandatory (`backlog/537-primitive-testing-discipline.md`).
- **Operator correctness and control-plane acceptance** remains a separate surface:
  - envtest (Layer A)
  - kind + Helm + KUTTL (Layer B)
  - see `make test-operators-envtest` and `make test-acceptance` (`tests/acceptance/README.md`).

Capability tests are the missing middle: “feature works” across primitives.

---

## 2) Where capability tests live (repo structure)

Each capability owns its **integration tests** under its capability folder:

```text
capabilities/<capability_name>/__tests__/integration/
  (native tests executed by the repo orchestrator)
```

This mirrors the v2 “integration classification” convention described in `backlog/430-unified-test-infrastructure-v2-target.md`.

---

## 3) Runner contract (inherits 430)

Capability integration tests must follow the **same v2 contract** as the rest of the repo:

- Phase 2 (Integration) is **local-only** and must be mocked/stubbed (no cluster).
- Phase 3 (E2E) is **cluster-only** and Playwright-only (if applicable).
- Output JUnit evidence via the orchestrator contract.

Reference: `backlog/430-unified-test-infrastructure-v2-target.md`.

---

## 3.1 No ad-hoc “test scripts”

Capability tests must not introduce ad-hoc runner scripts. They must run under `ameide test` so they are:

- discoverable by CI,
- artifact-producing (JUnit + logs),
- enforceable as a quality gate.

If a capability needs a convenience wrapper for developers, it must call `ameide test` (or a narrower repo-sanctioned wrapper), not create a parallel contract.

---

## 4) What to assert (527-native “WorkRequest seam”)

For capabilities that use the WorkRequests substrate (notably agentic coding / Transformation), the **single testable seam** is:

> WorkRequest → evidence → facts → projection timeline

Assertions must be on **durable state and facts**, not on “did a Pod exist”:

- Domain:
  - WorkRequest idempotency (duplicate deliveries converge to one canonical outcome)
  - facts emitted after persistence (outbox discipline)
  - valid status transitions
- Executor/runner:
  - “ack/commit only after durable outcome + evidence recorded”
  - evidence descriptor schema is well-formed (and redaction rules hold)
- Process:
  - requests work (domain intent), waits via Activity completion results (poll/heartbeat), emits process facts (`ToolRunRecorded`, `GateDecisionRecorded`, etc.)
  - deterministic progression under retries (engine-specific tests: Zeebe/Flowable runtime smokes for BPMN processes; Temporal replay tests only for platform Temporal workflows)
- Projection:
  - joins domain + process facts into a capability run timeline with citations

This matches the normative direction already stated in the Transformation docs and the Kafka/KEDA posture: at-least-once delivery is expected; idempotency + durable evidence is the correctness mechanism.

---

## 5) Test ladder (capability-owned)

Capability tests should be built **small → large** (TDD ladder), so we can validate behavior before adding cluster complexity.

### 5.1 Repo mode (fast, deterministic)

In Phase 2 (Integration), capability tests should:

- run the capability flow against in-process fakes or repo-local dependencies
- prove the “seam invariants” (idempotency, evidence schema, process determinism, projection materialization)

This is how we keep capability tests fast and CI-friendly for pre-merge.

### 5.2 E2E (Phase 3; cluster-only, Playwright-only)

Cluster validation for capabilities is owned by Phase 3 E2E under the 430v2 contract:

- Cluster-only
- Playwright-only
- Runs against a real base URL (remote-first AKS per `backlog/435-remote-first-development.md`)

For WorkRequests-based capabilities, cluster mode also validates:

- Kafka topic wiring (work queue topics; see `backlog/586-workrequests-execution-queues-inventory.md` and `backlog/587-kafka-topics-and-queues-inventory.md`)
- KEDA ScaledJob scheduling (execution backend)
- executor images and credentials are correct (for agentic coding: executor image is devcontainer-derived for toolchain parity)
- durable evidence is recorded and queryable even after job cleanup

---

---

## 5.4 Golden catalog (WorkRequests-based capabilities)

Capability owners should start with a small standard set of end-to-end tests (headless; no UI required) and then extend as needed:

- **E2E‑0: WorkRequest lifecycle smoke (no Process)**  
  Domain creates a deterministic WorkRequest → executor runs → Domain records outcome/evidence → Projection shows completion. Assert idempotency + evidence descriptor present + citations/timeline.
- **E2E‑1: Process orchestration slice (BPMN runtime)**  
  Start the workflow that requests work and awaits completion. Assert it emits `ToolRunRecorded` (and/or `GateDecisionRecorded`) referencing the WorkRequest/evidence.
- **E2E‑2: At-least-once + retry tolerance**  
  Simulate duplicate deliveries / retries. Assert one canonical outcome per idempotency key and stable timeline materialization.
- **E2E‑3: Guardrails slice (optional / nightly)**  
  Validate repo “physics plane” policies (PR-only, branch namespace permissions, path restrictions) where feasible; keep this isolated because it depends on external systems.

This catalog keeps E2E stable under KEDA retries and Kubernetes Job cleanup, because correctness is asserted on durable facts/evidence rather than transient Pods.

---

## 6) How to run “headless E2E” without a UI

Capability packs must not require a UI for core behavior validation.

The canonical headless approach is:

1. Create/trigger a capability action (direct domain command, or start a process workflow that creates the WorkRequest).
2. Wait for completion by querying Domain/Projection (not by inspecting Pods).
3. Assert evidence + facts + projection timeline.

UI e2e (Playwright) is separate and optional, and should only validate UISurface/editor UX—not the WorkRequest substrate.

---

## 7) Relationship to the repo-wide “front door”

- Repo-wide entrypoint stays `backlog/468-testing-front-door.md`.
- This backlog defines where capability packs live and what they must prove.

Follow-up (implementation): update the integration runner discovery (or the front door doc) so `capabilities/*/__tests__/integration/run_integration_tests.sh` can be executed alongside service/package packs under the same `INTEGRATION_MODE` contract.
