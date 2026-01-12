# 527 Transformation - E2E Execution Sequence (Triage; Functional)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

## Intent

Turn intake into a promotable slice: gather context, draft artifacts, and converge on a DoR decision.

## Allowed behavior

- Time-bounded projection reads (declared and treated as hints, not truth)
- Domain writes to draft/update artifacts and links
- Optional delegated analysis via WorkRequest (still treated as a tool/agent run whose completion is observed via Activity result, not broker facts)

## Outputs

- Draft artifacts ready for DoR gate
- Optional: start a durable run and emit `PhaseEntered(phase_key=triage)` as a process fact
