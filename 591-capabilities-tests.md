# 591 — Capability Tests (vertical slices + repo-wide test front door)

**Status:** Draft  
**Audience:** Engineers, test infra, platform operators  
**Scope:** Define how **capability-owned tests** provide the repo-wide “front door” for validating end-to-end feature flows across primitives and services.

---

## Core posture

Capability tests validate **vertical slices** across primitives and services while preserving primitive boundaries:

- Assertions are on **durable facts and evidence**, not on ephemeral runtime artifacts (pods, logs).
- Tests are **dual-mode** per 430: `repo` (fast/in-memory) and `cluster` (real dependencies).
- Capability tests complement (not replace) primitive/service tests:
  - primitives prove invariants,
  - services prove service behavior,
  - capabilities prove the feature composition.

---

## Where capability tests live

```
capabilities/<capability>/__tests__/integration/
  run_integration_tests.sh
  ...
```

The `run_integration_tests.sh` script follows the runner contract in:

- `backlog/430-unified-test-infrastructure.md`

---

## The repo-wide “front door”

The default front door for integration packs is:

- `pnpm test:integration`

Expectations:

- In `repo` mode, all eligible packs run without requiring a cluster.
- In `cluster` mode, packs target the real deployed environment and fail fast if required env vars are missing.

Capability packs are discovered alongside services/packages and must be runnable via the same front door.

---

## Test ladder (from smallest to largest)

Capability tests should be written “RED→GREEN” and grown slice-by-slice:

1. **Repo-mode seam test** (fast, deterministic)
   - Uses in-memory stubs/testkits for the minimal cross-primitive seam.
2. **Repo-mode workflow slice** (still deterministic)
   - Adds Process orchestration determinism (Temporal test environment).
3. **Cluster substrate slice** (real wiring)
   - Validates broker + KEDA + executor Job + durable evidence recording.
4. **Headless end-to-end** (no UI)
   - Validates Process → Domain → executor → evidence → projection timeline via RPCs/queries.
5. **UI E2E** (optional, separate)
   - Proves UISurface correctness, not substrate correctness.

---

## Golden catalog for WorkRequest-based capabilities (527 seam)

For capabilities that are implemented via the 527 execution substrate, the normative seam is:

> **WorkRequested → (KEDA ScaledJob) → Job executes → Evidence recorded → Facts emitted → Process continues → Projection timeline**

Golden tests (minimum set):

- **E2E-0 WorkRequest lifecycle smoke (headless)**
  - request work, await terminal state, assert durable outcome + evidence descriptor.
- **E2E-1 Process orchestration slice**
  - start workflow, await WorkCompleted/Failed, assert `ToolRunRecorded` is emitted and correlated.
- **E2E-2 At-least-once duplicate tolerance**
  - replay the same WorkRequested and assert idempotent convergence (single canonical outcome).
- **E2E-3 Repo guardrails (optional/nightly)**
  - branch protections/rulesets enforce “what is possible” (physics plane), independent of agents.

The key rule: tests must assert correctness on **Domain/Process/Projection facts + evidence references**, not on “did a pod exist”.

---

## Cross-references

- Integration packs contract + dual-mode requirements: `backlog/430-unified-test-infrastructure.md`
- Transformation execution substrate plan: `backlog/527-transformation-implementation-migration.md`
- Kafka/topic inventory for WorkRequest queues: `backlog/587-kafka-topics-and-queues-inventory.md`
- WorkRequest execution queue inventory: `backlog/586-workrequests-execution-queues-inventory.md`

