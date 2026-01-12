---
title: "656 – Agentic memory (curation + retrieval)"
status: draft
owners:
  - platform-devx
  - agents
  - platform
created: 2026-01-12
---

# 656 – Agentic memory (curation + retrieval)

## 0) Purpose

Define how Ameide maintains “memory” for agents in a way that is:

- scoped (applies to the correct repo/template/profile)
- authoritative (clear precedence)
- retrievable under limited context windows
- versioned (no silent drift)
- auditable (what was read, why it was trusted)

This backlog is split into:

1. **Curation**: how we author, version, deprecate, and validate memory.
2. **Retrieval**: how agents find and consume memory deterministically (and when to use semantic retrieval).

## 1) Problem statement

### 1.1 “Docs are not memory”

Human backlogs are long-form and exploratory; agents need short, current, enforceable rules:

- one command to run
- explicit guardrails (“don’t touch X”)
- what “done” means
- where evidence lives

### 1.2 Multiple sources of truth cause drift

In practice, agent guidance can come from:

- repo-root `AGENTS.md` (hard rules)
- template-scoped `AGENTS.md` (profile rules: code vs gitops vs sre)
- backlog docs (design decisions and contracts)
- CLI `--help` output (actual behavior)

Without an explicit precedence model and validation gates, agents will follow stale or conflicting instructions.

### 1.3 Submodule memory drift is real

With backlogs in a separate repo/submodule, we can end up with:

- code at commit A but memory at commit B (or vice versa)
- “pinned memory” that is accurate for older code but contradicts the current CLI

We need a deliberate structure so “what an agent should do” stays aligned to the code it runs.

## 2) Definitions

- **Memory artifact**: a short, agent-consumable contract (rules, runbooks, checklists, invariants).
- **Backlog**: human-oriented design/decision record; may be longer and historical.
- **Profile**: an agent type/template with distinct guardrails (e.g., code/gitops/sre).
- **Evidence**: machine-readable artifacts produced by the CLI (JUnit, logs, reports) that prove a check ran.
- **Read context**: the precise state that was read (head/baseline/version) and why it is reproducible.

## 3) Memory model (big picture)

### 3.1 Layered memory (authoritative precedence)

From highest authority (must be obeyed) to lowest:

1. **Template-scoped `AGENTS.md`** (profile-local guardrails and “how to operate here”)
2. **Repo-root `AGENTS.md`** (global workflow and mandatory checks)
3. **CLI contracts** (`ameide ... --help` + generated evidence layout) as runtime truth
4. **Curated memory artifacts** (short docs indexed for retrieval)
5. **Backlogs** (design intent, history, rationale)

Rule: when there is disagreement, higher layers win. Lower layers must be updated or marked superseded.

### 3.2 Where memory lives (two planes)

1. **Git-curated memory (now):**
   - `AGENTS.md` files (repo + templates)
   - backlog repo (pinned as a submodule in the code repo)

2. **Platform memory (future):**
   - store memory artifacts as **Elements + Versions** in the Transformation Domain (303/527),
   - serve them via Projection query services with `read_context` and `citations[]`.

The two-plane model allows us to start with Git (simple, reviewable) and evolve to auditable, queryable memory inside the platform.

## 4) Curation (how we create and maintain memory)

### 4.1 Memory artifact types (curated, short)

We standardize a small set of memory artifact types:

- **Policy**: non-negotiable rules (security posture, “don’t edit gitops from code profile”).
- **Contract**: stable interfaces/commands (CLI front doors, evidence layout).
- **Runbook**: step-by-step operational procedure (usually for SRE profile).
- **Recipe**: common workflow with guardrails (e.g., “make change → run tests → open PR”).
- **Index**: machine-readable registry of memory artifacts (see §4.3).

### 4.2 Lifecycle and deprecation rules

Every memory artifact must declare:

- `status`: `draft|active|superseded|deprecated`
- `owners`
- `last_verified` (date) or a CI gate that implies verification
- explicit **“superseded by”** links when status is not active

### 4.3 Memory indexing (required for retrieval)

We introduce a single “memory registry” index that is easy for agents and tools to consume:

- `backlog/agent-memory/index.yaml` (future file; this backlog defines the requirement)

Each entry includes:

- id (stable key)
- title
- scope (repo/profile/path)
- status
- owners
- tags (e.g., `tests`, `gitops`, `security`, `cli`)
- canonical link(s) (file path(s))

### 4.4 Change management gates (prevent drift)

Required checks:

- A change to the CLI that alters behavior must update either:
  - CLI help output, or
  - the corresponding memory artifact(s), or
  - mark older memory as superseded
- A change to template guardrails must update the template-scoped `AGENTS.md`.
- A change to “one-command” workflows must update the relevant memory entry and keep it short.

## 5) Retrieval (how agents consume memory)

### 5.1 Retrieval first principle: deterministic before semantic

Default retrieval should be deterministic and minimal:

- read the nearest-scoped `AGENTS.md`
- run the no-brainer CLI front door
- consult curated memory artifacts via the registry index

Semantic retrieval (embeddings) is used when:

- the agent is exploring, designing, or triaging unknown behavior
- the deterministic sources do not answer the question

### 5.2 Retrieval surfaces

We standardize two retrieval surfaces:

1. **Filesystem retrieval** (Git-curated memory):
   - `AGENTS.md` discovery by directory scope
   - memory registry + referenced short docs

2. **Projection-backed retrieval** (platform memory; future):
   - `getContext`/`semanticSearch` style tools as in `backlog/535-mcp-read-optimizations.md`
   - always return `read_context` + `citations[]` for audit-grade use (see `backlog/527-transformation-projection.md`)

### 5.3 Retrieval output contract (agent-friendly)

When the agent requests memory, the system returns:

- the minimal “instruction payload” (what to do next)
- the authoritative source path(s)
- the `read_context` (when platform-backed)
- citations/evidence pointers (when applicable)

## 6) Implementation plan

### 6.1 Now (Git-curated memory)

- Keep backlogs in a dedicated repo (already done).
- Define template profiles and require template-scoped `AGENTS.md` (see 650/654).
- Add a memory registry index and enforce it in CI (new work).

### 6.2 Next (CLI-assisted retrieval)

- Add CLI commands that help agents discover relevant memory:
  - `ameide doctor` (already exists) expands to also report memory pointers and profile detection
  - optional: `ameide memory show <topic>` backed by the registry index

### 6.3 Future (platform memory as Elements)

- Model curated memory artifacts as Elements in the Transformation Domain.
- Serve them via Projection with citations and versioned references.
- Use semantic retrieval for exploration, not for core guardrails.

## 7) Success criteria

- An agent can start from any template profile and immediately find the correct rules and the one no-brainer command.
- “What to run” and “what is forbidden” cannot silently drift from the CLI and templates.
- Retrieval always produces a small answer with links to authoritative sources.
- Deprecation is explicit and searchable; agents do not follow superseded instructions by default.

## 8) References

- Agentic coding suite (profiles, CLI front doors, templates): `backlog/650-agentic-coding-overview.md`, `backlog/654-agentic-coding-cli-surface.md`
- Test evidence and citations: `backlog/430-unified-test-infrastructure-v2-target.md`, `backlog/527-transformation-projection.md`
- Element graph substrate: `backlog/300-400/303-elements.md`, `backlog/527-transformation-domain.md`
- Semantic retrieval posture: `backlog/535-mcp-read-optimizations.md`
