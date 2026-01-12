# 527 Transformation - E2E Execution Sequence (Execution; Implementation)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

## Typical loop

1) Enter `execution` (emit `PhaseEntered(execution)` as a process fact)  
2) Request work (scaffold/codegen, agentic coding, verification) via Domain `RequestWork`  
3) Activity waits for completion (poll/long-poll + heartbeat), returns `{work_request_id, outcome, evidence_refs}`  
4) Workflow records `ToolRunRecorded`/`ActivityTransitioned` and decides next step

## WorkRequest classes (v1 examples; see `backlog/527-transformation-proto.md`)

- Tool runs: generate/verify queues (e.g., `transformation.work.queue.toolrun.generate.v1`, `transformation.work.queue.toolrun.verify.v1`)
- Agent work: coder queue (e.g., `transformation.work.queue.agentwork.coder.v1`)

## Codex auth (agentic coding)

- The AgentWork "coder" executor expects authenticated Codex CLI via an `auth.json` file.
- In-cluster, infra mounts `Secret/codex-auth` and sets `CODEX_HOME=/codex-home` (initContainer copies `auth.json` into place).
- If `codex-auth` is missing, the executor fails fast and records a failed WorkRequest outcome (Domain emits facts via outbox).
