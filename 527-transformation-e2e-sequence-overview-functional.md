# 527 Transformation - E2E Execution Sequence (Overview; Functional)

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`  
**Related:** `backlog/520-primitives-stack-v2.md`, `backlog/496-eda-principles.md`, `backlog/527-transformation-process.md`

## Separation (what this document set is asserting)

- **Process** decides what happens next (orchestration only; deterministic workflow code).
- **Events** carry **intents** (requests) and **facts** (audit/outcomes) with strict 496 semantics.
- **Infra** decides *how* work runs (KEDA/Jobs/worker pools/external executors), without leaking into process semantics.
- **UI** shows a projection-driven timeline/kanban (domain facts + process facts + evidence refs), not infra internals.

## Terminology

- **Activity-inline execution:** work runs inside the Process primitive’s Temporal Activity worker.
- **Activity-delegated execution:** an Activity initiates work elsewhere (via WorkRequest) and returns a correlation/result handle; the Workflow waits for completion (Signal/Update; timers once supported) and continues.

## Kanban phases (recommended)

1. `intake` - question/need captured; no durable run required yet.
2. `triage` - context gathering + drafting of promotable artifacts.
3. `awaiting_approval` - a human decision is required (user task).
4. `execution` - work is running (tool-run, agent-work, verification loops).
5. `acceptance` - optional acceptance gate (preview/validation; user task).
6. `release` - publish + GitOps promotion + rollout verification.
7. `terminal` - `done|failed`.

## Transformation R2R profile (capability-specific; normative)

Once a run exists, keep human gating to **two** user tasks only:

1) **DoR gate** (“ready to execute” / baseline decision)  
2) **Acceptance gate** (“accept for release” decision)

Everything else MUST be automated via `serviceTask`/Activities/WorkRequests.

## Contract invariants (must not regress)

1. **Process is orchestration only**: workflows never encode infra mechanics (KEDA, Jobs, consumer lag, image tags).
2. **Activities are the side-effect boundary**: workflows initiate side effects only via Activities.
3. **Facts are not requests**: facts are pub/sub audit statements; execution queues are intent-only (496).
4. **Owner emits facts**: only the owning Domain emits domain facts, and only after persistence (outbox).
5. **Infra is swappable**: the executor can change without changing process semantics.
6. **At-least-once everywhere**: Kafka delivery is at-least-once; Activities may run more than once; all writes MUST be idempotent.
7. **No internal EDA control flow**: workflows MUST NOT advance step progression by consuming broker facts.
8. **No “activities-only orchestration”**: long-lived state + gate decisions live in Workflows; Activities remain retryable side effects only (and may heartbeat while waiting).
