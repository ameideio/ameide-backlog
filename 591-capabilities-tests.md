# 591 — Capability Tests: Integration Packs at the Capability Boundary

**Status:** Draft (target-state direction)  
**Audience:** Capability owners, platform engineering, test-infra  
**Scope:** Define how to write and run **capability-owned tests** (vertical slice tests spanning primitives), aligned with `backlog/430-unified-test-infrastructure.md`.

---

## Use with

- Repo-wide test target state: `backlog/430-unified-test-infrastructure.md`
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

Each capability owns its integration pack under its capability folder:

```text
capabilities/<capability_name>/__tests__/integration/
  run_integration_tests.sh
  (implementation of the pack)
```

This mirrors the existing pack pattern in `services/*` and `packages/*` described in `backlog/430-unified-test-infrastructure.md` and indexed by `backlog/468-testing-front-door.md`.

---

## 3) Runner contract (inherits 430)

Capability integration packs must follow the **same runner contract** as service integration packs:

- `INTEGRATION_MODE=repo|cluster` (accept `local` as non-cluster; do not introduce new modes)
- One implementation, dual execution (no “cluster-only” test logic branches)
- Fail fast on missing env vars in `cluster` mode
- Output JUnit + artifacts in the standard locations used by CI

Reference: `backlog/430-unified-test-infrastructure.md`.

---

## 3.1 No ad-hoc “test scripts”

Capability tests must be runnable as an integration pack (`run_integration_tests.sh`) so they are:

- discoverable by CI,
- mode-consistent (`INTEGRATION_MODE`),
- artifact-producing (JUnit + logs),
- enforceable as a quality gate.

If a capability needs a convenience wrapper for developers, it must call the pack entrypoint rather than creating a parallel test path.

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
  - requests work (domain intent), awaits domain facts, emits process facts (`ToolRunRecorded`, `GateDecisionRecorded`, etc.)
  - deterministic replay (Temporal workflow tests)
- Projection:
  - joins domain + process facts into a capability run timeline with citations

This matches the normative direction already stated in the Transformation docs and the Kafka/KEDA posture: at-least-once delivery is expected; idempotency + durable evidence is the correctness mechanism.

---

## 5) Test ladder (capability-owned)

Capability tests should be built **small → large** (TDD ladder), so we can validate behavior before adding cluster complexity.

### 5.1 Repo mode (fast, deterministic)

In `INTEGRATION_MODE=repo`, capability packs should:

- run the capability flow against in-process fakes or repo-local dependencies
- prove the “seam invariants” (idempotency, evidence schema, process determinism, projection materialization)

This is how we keep capability tests fast and CI-friendly for pre-merge.

### 5.2 Cluster mode (real dependencies)

In `INTEGRATION_MODE=cluster`, capability packs run against the real deployed system (remote-first AKS per `backlog/435-remote-first-development.md`, and GitOps authority per `backlog/430-unified-test-infrastructure.md`).

For WorkRequests-based capabilities, cluster mode also validates:

- Kafka topic wiring (work queue topics; see `backlog/586-workrequests-execution-queues-inventory.md` and `backlog/587-kafka-topics-and-queues-inventory.md`)
- KEDA ScaledJob scheduling (execution backend)
- executor images and credentials are correct (for agentic coding: executor image is devcontainer-derived for toolchain parity)
- durable evidence is recorded and queryable even after job cleanup

---

## 5.3 Where cluster-mode tests run

Cluster-mode capability packs may be executed:

- **from outside the cluster** (CI runner or DevContainer) using kubeconfig/Telepresence, or
- **as a headless Kubernetes Job** (when you want the test driver to run inside the same network boundary).

Regardless of where the pack runs, the assertions remain the same: verify correctness via Domain/Process/Projection state and evidence, not via “pods existed” heuristics.

**Default guidance:**

- **Developer inner loop:** run cluster-mode packs from a DevContainer using Telepresence (fast iteration).
- **CI / remote execution:** prefer running the pack as a headless Kubernetes Job when you want network parity and fewer port-forward assumptions.

---

## 5.4 Golden catalog (WorkRequests-based capabilities)

Capability owners should start with a small standard set of end-to-end tests (headless; no UI required) and then extend as needed:

- **E2E‑0: WorkRequest lifecycle smoke (no Process)**  
  Domain creates a deterministic WorkRequest → executor runs → Domain records outcome/evidence → Projection shows completion. Assert idempotency + evidence descriptor present + citations/timeline.
- **E2E‑1: Process orchestration slice (Temporal)**  
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
