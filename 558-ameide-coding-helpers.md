# 558 — Ameide Coding Helpers (Deterministic Tooling Package)

**Status:** In progress (partial implementation landed)  
**Audience:** CLI implementers, agent/tool implementers, Transformation Process teams  
**Scope:** Add a shared deterministic tooling package (consumable by humans, agents, and process activities) and move toward a proto-first report contract.

## Target statement

The delivery process is the IT4IT-aligned Transformation Process. This package exists to provide deterministic “coding helper” actions that can be invoked by:

- humans in the repo (inner-loop),
- agents (tool calls in the Transformation Domain),
- deterministic process activities (Temporal-backed Process primitives).

## Where this lands (520 alignment)

This work belongs to the **guardrails plane** (`buf + CI + deterministic tools`) described by `backlog/520-primitives-stack-v2.md`. It is not an Integration primitive and it is not “integration tests”.

## Mental maps

### AS‑IS

```text
ameide (CLI)
├─ Repo tooling + guardrails
├─ Platform client commands
└─ Ad-hoc JSON outputs
```

### TO‑BE

```text
packages/ameide_coding_helpers
├─ Actions: doctor|scaffold|generate|verify
└─ Reports: proto-first, evidence-first

Invokers
├─ Humans (CLI / direct)
├─ Agents (tool calls)
└─ Process activities (Temporal)
```

## Deliverables

- `packages/ameide_coding_helpers/**` exists (repo-local package).
- A stable proto report schema exists for evidence-first output:
  - tool identity + argv/workdir
  - failures + next steps
  - evidence references (paths/URIs/digests)

## Current implementation (what exists today)

- `packages/ameide_coding_helpers/guardrails` exists and is the canonical home for running Buf guardrails (used by `ameide verify`).
- `packages/ameide_coding_helpers/doctor` exists and powers `ameide doctor` (fast preflight for toolchain + repo invariants).
- The CLI surfaces these as first-class commands:
  - `ameide doctor` (human/agent/process preflight; supports `--json`)
  - `ameide verify` (repo + primitives gate; invokes coding-helpers guardrails)

## Report contract (now vs next)

- **Now:** deterministic actions return `ameide_core_proto.primitive.v1.CheckResult` (proto type) and can be rendered as JSON (CLI `--json`) for agents/processes.
- **Next:** map/emit tool evidence as `ameide_core_proto.process.transformation.v1.ToolRunRecorded` (process facts) so orchestration can store immutable evidence and references, and keep the CLI output a view (not the evidence store).

## Next steps (to complete 558)

- Extract more deterministic actions from the CLI into `packages/ameide_coding_helpers`:
  - `generate` (Buf template invocations + SDK sync helpers)
  - `verify` (policy scripts + workspace invariants; already partially in CLI)
  - `scaffold` (pure file generation; avoid coupling to interactive CLI concerns)
- Add an explicit “no secrets” guardrail for common repo credential files (e.g., `.npmrc`, `.netrc`) to prevent token regressions.

## Cross-references

- Transformation evidence posture: `backlog/527-transformation-process.md`, `backlog/527-transformation-proto.md`
- Tool evidence descriptor discussion: `backlog/527-transformation-integration.md`
- CLI mindmaps (historical): `backlog/557-ameide-cli-runner-mindmaps.md`
 - SDK-only consumption policy: `backlog/300-400/393-ameide-sdk-import-policy.md`
