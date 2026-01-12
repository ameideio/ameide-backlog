# 527 Transformation - E2E Execution Sequence (Execution; Functional)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

## Intent

Run deterministic machine steps (generate/implement/verify/publish preparation) until gate rules are satisfied.

## Rules

- `serviceTask` compiles to a single Activity invocation.
- Delegated work uses WorkRequests, but **workflow progression is via Activity completion results** (no internal EDA control flow).
- Retries rely on explicit idempotency keys and must be visible as process facts.
