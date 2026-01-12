# 511 – Process Primitive BPMN Conformance Suite (v2: Zeebe runtime)

This backlog defines the **new conformance gate** for BPMN-authored processes in Ameide:

- BPMN is authored as “Flowable-shaped” (standards-aligned constructs where possible).
- BPMN is executed by **Camunda 8 / Zeebe**.
- Ameide enforces “diagram must not lie” by verifying **deployability, worker coverage, and runtime semantics** against the Zeebe engine.

This replaces the v1 “BPMN compiled/executed via Temporal testsuite” conformance posture (see `backlog/511-process-primitive-bpmn-conformance-suite.md`).

## Non-negotiables (vendor-aligned)

1. **Use Camunda 8 Orchestration Cluster REST API as the primary test surface**.
   - Do not build the conformance gate on Operate/Tasklist REST APIs.
2. **Do not rely on “await completion” semantics for processes that contain waits** (messages, timers, user tasks).
   - Tests must drive wait states explicitly (publish/correlate message, complete user task, wait for timer).
3. **Timers are asserted as “not earlier”**.
   - Never assert exact firing times; allow “later” within a bounded timeout.
4. **Incidents are asserted via engine APIs**.
   - Create an incident deterministically by exhausting retries and verify it exists for the process instance.

## What this conformance suite must prove

### A) Deployability (“executable BPMN”)

- The BPMN resource deploys successfully to Zeebe (atomic deployment).

### B) Worker coverage (“side effects are implementable”)

- Every `zeebe:taskDefinition type="..."` used by the BPMN is:
  - activatable by a worker (jobs can be activated), and
  - completable (jobs can be completed and the instance progresses).

### C) Runtime semantics (Camunda/Zeebe truth, not “our interpretation”)

- Exclusive gateway branching is correct and observable.
- Message correlation behaves as the engine defines (subscription + correlation key).
- Message buffering/uniqueness is tested using TTL + `messageId` semantics.
- Timers eventually drive the expected path (never earlier, may be later).
- Incidents are produced when jobs fail with no retries remaining.

## Conformance fixture: a single “kitchen sink” BPMN, tested by segments

We keep a single fixture BPMN but we do **not** run it as one giant happy-path test.

Instead, tests run **small, deterministic segments** using Camunda’s “run process segment” feature (runtime/start instructions), which is recommended for testing-only usage.

### Required segments (v2)

- Segment A: `serviceTask` job success + variable output
- Segment B: `exclusiveGateway` branches by variable (two runs)
- Segment C: message catch (subscription open → correlate message → progress)
- Segment D: message buffering (publish with TTL before subscription; later opens subscription and consumes buffer)
- Segment E: message uniqueness (duplicate `messageId` while buffered is rejected / not accepted as a second buffered message)
- Segment F: timer catch/boundary timer (assert “not earlier”, allow “later”)
- Segment G: incident (job failure with retries exhausted)

## Test runner: cluster-first (no local runtime required)

The cluster is the execution truth. The conformance gate runs directly against the dev namespace Camunda cluster.

### Execution model

- A test runner deploys the fixture BPMN to Zeebe via Orchestration Cluster REST API.
- Tests start instances using segment instructions.
- Tests run “in-test workers” by calling:
  - activate job(s)
  - complete job(s)
  - fail job(s)
- Tests assert engine outcomes by reading:
  - taken sequence flows for a process instance
  - incidents for a process instance

### Orchestration Cluster REST API surface (expected)

The runner should be implemented against the Orchestration Cluster REST API endpoints for:

- Deploy resources (BPMN): `POST /v2/deployments`
- Start process instance (segment testing): `POST /v2/process-instances` with `startInstructions` / runtime instructions
- Activate jobs: `POST /v2/jobs/activation`
- Complete job: `POST /v2/jobs/{jobKey}/completion`
- Fail job: `POST /v2/jobs/{jobKey}/failure`
- Publish message (buffering): `POST /v2/messages/publication`
- Correlate message (immediate feedback): `POST /v2/messages/correlation`
- Read taken sequence flows: `GET /process-instances/{processInstanceKey}/sequence-flows`
- Search incidents for instance: `POST /process-instances/{processInstanceKey}/incidents/search`

### Backpressure and determinism

The runner must implement retry/backoff on temporary engine responses (including backpressure) and must apply strict, scenario-specific timeouts. On timeout, dump diagnostics (sequence flows, incidents, last job activation errors).

## Message semantics: what we test (vendor semantics)

Do **not** test “message dedupe” as “one delivery can’t satisfy two waits” (that’s not how Zeebe message correlation works).

Instead, test:

1. **No subscription → discard** (TTL=0).
2. **Buffering** works (TTL>0 publish before subscription opens → later correlates).
3. **Uniqueness while buffered**:
   - publishing the same `(name, correlationKey, messageId)` while a buffered copy exists must be rejected / not create a second buffered copy.
4. **Cardinality**:
   - a correlated message is applied once (do not design tests assuming one message can satisfy multiple open subscriptions).
5. Prefer **correlate** when the test needs immediate feedback; prefer **publish** when the test needs buffering.

## Timers: what we test

- Assert “timer-driven path eventually taken”.
- Never assert exact time; allow timer to trigger later than due date.

## Incidents: what we test

- Fail a job until retries are 0 (or fail it with retries already set to 0).
- Assert an incident exists for the process instance via Orchestration Cluster API.

## Implementation plan (repo work)

### 0) Decide the fixture + runner locations (one-time)

- Add a new fixture primitive `primitives/process/bpmn_conformance_v2/`:
  - `bpmn/positive/bpmn_conformance_v2.bpmn` (segment-enabled)
  - `bpmn/negative/*.bpmn` (unsupported constructs, engine/profile rejects)
  - `README.md` (how to run in-cluster)

### 1) Implement a small Orchestration REST client (Go)

- Implement a minimal HTTP client in `packages/ameide_core_cli/internal/zeebe_orchestration_api/`:
  - deploy BPMN
  - start instance with segment instructions
  - activate/complete/fail jobs
  - publish/correlate messages
  - query sequence flows + incidents for a process instance

### 2) Implement scenario tests (integration, in-cluster)

- Add Go integration tests under `tests/integration/process_bpmn_conformance_v2/` (or equivalent) that:
  - deploy once per test run (or per test file),
  - execute segments A–G independently,
  - assert the engine state per the semantics above.

### 3) Wire into the repo front door

- Run the conformance suite as a Kubernetes Job in the dev namespace (same network + auth as the platform) and make it part of the repo test front door (`./ameide dev inner-loop-test`).
- Make the conformance gate fail fast with actionable output (including URLs/ids for Operate troubleshooting when available).

### 4) Decommission v1 runtime semantics gate

- Keep v1 fixtures for historical context only, but remove them from “required for merge” gates once v2 is green in CI.
