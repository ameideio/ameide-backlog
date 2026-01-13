# 527 Transformation — E2E Sequence (v5: Zeebe request → wait → resume)

This document supersedes `backlog/527-transformation-e2e-sequence-v4.md` by making the Zeebe execution model explicit for long-running steps.

## Intent

Describe the real-world end-to-end sequence for Transformation governance when:

- Governance processes are authored in BPMN and execute on **Camunda 8 / Zeebe**.
- Side effects are executed outside the engine by **job workers** (owned by the Process primitive).
- Long work is modeled as **Request → Wait → Resume** (messages with buffering).
- Domains remain canonical truth; process execution state is observable via Zeebe/Operate.

## The default execution template for “big steps”

For any step that can exceed “seconds” (coding, tests, builds, rollout verify):

1. **Request** (service task): worker schedules/requests work quickly and persists `work_id` + handles.
2. **Wait** (message catch / receive task): process instance waits for completion correlated by `work_id`.
3. **Resume** (publish message): completion publishes a buffered message (TTL>0) with `messageId=work_id` and outputs.

This avoids holding a Zeebe job lease for minutes/hours and aligns the diagram with Zeebe runtime behavior.

## High-level stages (Requirement → Development workpackage → Release package)

### Stage 1: Requirement (triage) — human-driven authoring + publish

User actions (Ameide UI):

- Draft requirement (document element + metadata).
- Publish / DoR decision (explicit human gate).

System outcomes:

- A pinned requirement slice/version exists (immutable reference for downstream work).

### Stage 2: Development workpackage — automated until UAT outcome

Trigger:

- Workpackage created and started (select N requirement slices).

Zeebe execution:

- Request long-running work (agentic coding, CI/tests, PR creation, preview) via service-task workers.
- Wait for completion via message catch events (buffered publish to avoid races).

User actions (Ameide UI):

- Observe deliverables (PR link, preview URL, evidence summary).
- Mark UAT outcome per requirement (pass/fail) via a user task or a UI action bridged to Tasklist semantics.

### Stage 3: Release package — batched promotion + verify

Trigger:

- Operator selects N “Approved for Release” requirements into a release package and starts release.

Zeebe execution:

- Request build/publish/promotion/rollout verify as long-running work and resume by messages.

User actions:

- Optional single acceptance gate for the release package (if desired).

## Verification posture (non-negotiable)

Promotion/deployment must fail unless:

- BPMN is deployable to Zeebe,
- BPMN conforms to Ameide profile constraints,
- the Process primitive worker implements every job type used by the BPMN,
- long-running steps are modeled as Request → Wait → Resume (messages with buffering), not “hold the lease and do minutes of work”.

