# 527 Transformation — E2E Sequence (v5: Zeebe Request→Wait→Resume for long work)

This updates `backlog/527-transformation-e2e-sequence-v4.md` with the vendor-aligned runtime semantics needed for long-running steps.

> **Superseded:** see `backlog/527-transformation-e2e-sequence-v6.md` (makes the repository substrate Git-backed and clarifies owner-issued `work_id`).

## The key runtime rule

Any step that can take more than “seconds” MUST be modeled as:

1) **Request** (serviceTask; short Zeebe job)  
2) **Wait** (message catch / receive task; explicit BPMN wait state)  
3) **Resume** (external completion publishes `work.completed.v1` with TTL + messageId)

This avoids holding Zeebe job leases across multi-minute work (build/tests/agent loops).

## work_id (correlation)

- `work_id` is persisted as a process variable before entering the wait state.
- The Wait step subscribes to `work.completed.v1` correlated by `work_id`.
- `work_id` is treated as immutable per wait (set before entering the wait state).

## Completion ingress (where user/UI and external systems connect)

The process solution owns the completion ingress:
- External execution backends (Coder, CI runners, etc.) call back to the worker service with `{work_id, status, outputs}`.
- The worker service publishes `work.completed.v1` (TTL + messageId) to Zeebe.

This keeps “EDA between primitives” intact while using Zeebe messages for BPMN wait-state resumption.
