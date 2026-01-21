# 557 – Ameide CLI: As‑Is vs To‑Be Mindmaps (Runner Target)

**Status:** Draft  
**Audience:** CLI implementers, platform engineers, agent/tooling authors  
**Scope:** CLI internal structure + target posture (`doctor|scaffold|generate|verify`) with cross-references to existing backlogs.

> **Deprecation notice:** This backlog is superseded by `backlog/558-ameide-coding-helpers.md`, which moves deterministic tooling into a shared package and returns the CLI toward a proto-based platform client posture.

Implementation note: the extraction has started — `packages/ameide_coding_helpers/*` exists and `ameide doctor` is now implemented as a first-class CLI command.

## Target posture (what “runner-first” means)

Reframe: the CLI is **not** the delivery process — it is a deterministic tool invoked by the IT4IT-aligned Transformation Process.

- **Process primitive**: sequencing + gates + rework loops + evidence trail (process facts).
- **Deterministic tooling**: `doctor|scaffold|generate|verify` outputs structured evidence.
- **CLI**: human convenience surface that can call deterministic tooling locally or call platform APIs and render results; it must not “own” approvals/promotion/orchestration.

## AS‑IS mindmap (today)

```text
ameide (single binary)
├─ Platform client commands (gRPC)
├─ Repo tooling commands (scaffold/generate/verify)
├─ Guardrails + drift detection (mixed concerns)
└─ JSON outputs exist, but no single stable report contract
```

## TO‑BE mindmap (target)

```text
packages/ameide_coding_helpers (deterministic library)
├─ Actions: doctor|scaffold|generate|verify (repo gate)
└─ Report: ToolExecutionReport (proto), evidence-first

ameide (CLI)
├─ Calls coding helpers for local deterministic actions
└─ Trends toward proto-based platform client posture
```

## Target UX (opinionated)

- `ameide doctor` (preflight identity + required tool availability)
- `ameide scaffold` (repo-owned skeletons)
- `ameide generate` (wrapper for `buf generate` using repo templates)
- `ameide test` (policy gate definition in Phase 0; same checks locally and in CI)

## Cross-references

- Stack constitution: `backlog/520-primitives-stack-v6.md`
- CLI posture: `backlog/409-ameide-cli.md`
- Generation baselines: `backlog/521a-external-generation-baseline.md`, `backlog/521b-internal-generation-baseline.md`
- Verification baselines: `backlog/521e-internal-verification-baseline.md`, `backlog/521f-external-verification-baseline.md`
- Transformation evidence + facts: `backlog/527-transformation-process.md`, `backlog/527-transformation-proto.md`
- Coding helpers target: `backlog/558-ameide-coding-helpers.md`
