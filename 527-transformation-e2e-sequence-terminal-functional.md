# 527 Transformation - E2E Execution Sequence (Terminal; Functional)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

## Intent

Close the run with a terminal status and durable evidence references.

## Rules

- Terminal state is expressed as a process fact (`RunCompleted` or `RunFailed`) with evidence links and correlation metadata.
