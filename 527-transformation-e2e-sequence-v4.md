# 527 Transformation — E2E Sequence (v4: Zeebe runtime for BPMN-authored processes)

This document replaces the earlier “Temporal-based” E2E sequence narratives under `backlog/527-transformation-e2e-sequence*.md`.

## Intent

Describe the real-world end-to-end sequence for Transformation governance when:

- The governance process is authored in BPMN.
- The BPMN executes on **Camunda 8 / Zeebe**.
- Side effects are executed by **Ameide primitives** as Zeebe workers (agents/integrations), not inside the process engine.
- Domains remain canonical truth; orchestration history is observable via Zeebe/Operate.

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

- Service tasks enqueue work via worker job types (coding agent, tests, PR creation, preview labeling).

User actions (Ameide UI):

- Observe PR + preview URL.
- Mark UAT outcome per requirement (pass/fail) using a user task or an explicit UI action bridged to Tasklist semantics.

### Stage 3: Release package — batched promotion + verify

Trigger:

- Operator selects N “Approved for Release” requirements into a release package and starts release.

Zeebe execution:

- Promotion and verification steps run as worker jobs (GitOps PR, rollout verify, release verify).

User actions:

- Optional single acceptance gate for the release package (if desired).

## Verification posture (non-negotiable)

- BPMN must be verifiable as:
  - deployable to Zeebe,
  - conformant to Ameide profile constraints,
  - fully covered by worker implementations for every side-effect task type.

If any of the above is not true, promotion/deployment must fail (“diagram must not lie”).

