# 558 — Ameide Coding Helpers (Deterministic Tooling Package)

**Status:** Draft  
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

## Cross-references

- Transformation evidence posture: `backlog/527-transformation-process.md`, `backlog/527-transformation-proto.md`
- Tool evidence descriptor discussion: `backlog/527-transformation-integration.md`
- CLI mindmaps (historical): `backlog/557-ameide-cli-runner-mindmaps.md`

