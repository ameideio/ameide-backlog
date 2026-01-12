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
2. **Retrieval**: how agents find and consume memory via graph + embedding retrieval (and when to fall back to deterministic sources).

## 1) Problem statement

### 1.1 “Docs are not memory”

Human backlogs are long-form and exploratory; agents need short, current, enforceable rules:

- one command to run
- explicit guardrails (“don’t touch X”)
- what “done” means
- where evidence lives

### 1.2 Agents are not only coders

Agents operate in multiple roles:

- **coding agents**: change code in a Git repo, run CLI checks, open PRs
- **assistants in chat**: help humans define architecture and business processes that must be stored as **versioned elements** (e.g., BPMN processes, ArchiMate views, reference documents)

“Memory” therefore cannot be treated as “coding instructions only”. It must include business and architecture artifacts as first-class, versioned knowledge.

### 1.3 Multiple sources of truth cause drift

In practice, agent guidance can come from:

- **canonical element memory** (documents/diagrams stored as Elements + Versions)
- `AGENTS.md` (coding agents only; execution guardrails while operating on a Git working tree)
- backlog docs (a Git-view of memory docs; transitional)
- CLI `--help` output (tool runtime truth)

Without an explicit precedence model and validation gates, agents will follow stale or conflicting instructions.

### 1.4 Submodule memory drift is real

With backlogs in a separate repo/submodule, we can end up with:

- code at commit A but memory at commit B (or vice versa)
- “pinned memory” that is accurate for older code but contradicts the current CLI

We need a deliberate structure so “what an agent should do” stays aligned to the code it runs.

## 2) Definitions

- **Memory artifact**: a versioned element intended for agent context (raw Markdown, BPMN, ArchiMate, etc.).
- **Backlog**: a document-shaped memory artifact (Markdown element), organized by architecture layer.
- **Curation queue**: a queue of “memory candidates” awaiting consolidation and promotion into canonical memory.
- **Profile**: an agent type/template with distinct guardrails (e.g., code/gitops/sre).
- **AGENTS.md**: a Codex-oriented implementation mechanism that scopes **coding agent** behavior while operating on a Git working tree; it does not define the platform’s global memory system.
- **Evidence**: machine-readable artifacts produced by the CLI (JUnit, logs, reports) that prove a check ran.
- **Read context**: the precise state that was read (head/baseline/version) and why it is reproducible.

## 3) Memory model (big picture)

### 3.1 Canonical memory lives in the element repository

Canonical memory is stored as **Elements + Relationships + Versions** (303), written by domain primitives (527) and served by projection query services with `read_context` and `citations[]`.

This includes:

- raw Markdown (policies, runbooks, “backlogs”, reference docs)
- BPMN process definitions and diagrams
- ArchiMate models and views

### 3.2 Organize memory by architecture layers (required)

All memory artifacts are organized by architecture layers:

- Strategy
- Business
- Application
- Technology
- Implementation & Migration

This is not only an IA decision; it is a retrieval primitive. Agents should be able to filter and traverse memory by layer, and follow relationships across layers.

### 3.3 Layered memory (authoritative precedence)

From highest authority (must be obeyed) to lowest:

1. **Published memory elements** (promoted/published element versions and baselines)
2. **Draft memory elements** (draft versions awaiting promotion)
3. **Curation queue items** (explicitly non-authoritative)
4. **CLI contracts** (`ameide ... --help` + generated evidence layout) as runtime truth for tool execution
5. **`AGENTS.md` files** (coding agents only; local execution guardrails)

Rule: when there is disagreement, higher layers win. Lower layers must be updated or marked superseded.

### 3.4 Where memory lives (two planes)

1. **Git-curated memory (bootstrap / transitional):**
   - `AGENTS.md` files (coding agent instructions only)
   - backlog repo/submodule (a Git-view of memory docs)

2. **Platform memory (target):**
   - store memory artifacts as **Elements + Versions** in the Transformation Domain (303/527)
   - curate memory in the element editor (human-first)
   - serve memory via Projection query services with `read_context` and `citations[]`

The two-plane model allows us to start with Git (simple, reviewable) and evolve to auditable, queryable memory inside the platform.

## 4) Curation (how we create and maintain memory)

### 4.1 Curation queue (agents propose, curator promotes)

Coding agents will rarely do a good job curating canonical memory while also shipping code.

Decision: coding agents should **append to a curation queue** instead of directly editing canonical memory. A dedicated curator (human, or a dedicated “curator agent” with approval gates) consolidates and publishes.

Assistants operating in chat (e.g., while defining a business process) may generate draft BPMN/ArchiMate/document elements, but those drafts still enter curation (draft state and/or queue) before becoming published memory.

Minimum requirements for a queue item:

- a proposed memory artifact kind (policy/contract/runbook/recipe/backlog/BPMN/ArchiMate)
- scope and intended audience (coding vs business-process vs architecture)
- citations/evidence pointers (CLI output, PR URLs, existing element IDs, etc.)
- proposed destination (update existing canonical element vs create a new element)

### 4.2 Memory artifact types (curated, multi-modal)

We standardize a small set of memory artifact types:

- **Policy**: non-negotiable rules (security posture, “don’t edit gitops from code profile”).
- **Contract**: stable interfaces/commands (CLI front doors, evidence layout).
- **Runbook**: step-by-step operational procedure (usually for SRE profile).
- **Recipe**: common workflow with guardrails (e.g., “make change → run tests → open PR”).
- **Backlog doc**: decision record organized by architecture layer (Markdown).
- **BPMN**: business process definition (diagram) stored as a versioned element.
- **ArchiMate**: architecture model/view stored as versioned elements.
- **Index**: machine-readable registry of memory artifacts (see §4.4).

### 4.3 Lifecycle and deprecation rules

Every memory artifact must declare:

- `status`: `draft|active|superseded|deprecated`
- `owners`
- `last_verified` (date) or a CI gate that implies verification
- explicit **“superseded by”** links when status is not active

### 4.4 Memory indexing (required for retrieval)

We introduce a single “memory registry” index that is easy for agents and tools to consume:

- `backlog/agent-memory/index.yaml` (future file; this backlog defines the requirement)

Target-state note: the “index” should ultimately be served as a Projection query over memory elements (filterable by layer/profile/status), not as a hand-maintained file. The file is a bootstrap mechanism only.

Each entry includes:

- id (stable key)
- title
- scope (repo/profile/path)
- status
- owners
- tags (e.g., `tests`, `gitops`, `security`, `cli`)
- canonical link(s) (file path(s))

### 4.5 Change management gates (prevent drift)

Required checks:

- A change to the CLI that alters behavior must update either:
  - CLI help output, or
  - the corresponding memory artifact(s), or
  - mark older memory as superseded
- A change to template guardrails must update the template-scoped `AGENTS.md`.
- A change to “one-command” workflows must update the relevant memory entry and keep it short.

## 5) Retrieval (how agents consume memory)

### 5.1 Retrieval first principle: embedding/graph-first

Default retrieval should be embedding/graph-first over the element repository:

- graph neighborhood expansion over `ElementRelationship` (bounded depth)
- semantic search over element versions (hybrid retrieval + RRF)
- filter by architecture layer, profile, and status (published vs draft)

Deterministic fallbacks exist when:

- the agent is a **coding agent** operating on a Git working tree and needs local execution rules (`AGENTS.md`)
- the element repository is unavailable
- a runtime/tool contract must be verified directly (CLI help output)

### 5.2 Retrieval surfaces

We standardize two retrieval surfaces:

1. **Projection-backed retrieval** (platform memory; target):
   - `getContext` / `semanticSearch` style tools as in `backlog/535-mcp-read-optimizations.md`
   - always return `read_context` + `citations[]` for audit-grade use (see `backlog/527-transformation-projection.md`)

2. **Filesystem retrieval** (bootstrap/fallback):
   - `AGENTS.md` discovery (coding agents only)
   - memory registry + Git-view docs (backlog submodule) when platform memory is unavailable

### 5.3 Retrieval output contract (agent-friendly)

When the agent requests memory, the system returns:

- the minimal “instruction payload” (what to do next)
- the authoritative source path(s)
- the `read_context` (when platform-backed)
- citations/evidence pointers (when applicable)

## 6) Implementation plan

### 6.1 Now (bootstrap + curation queue)

- Keep backlogs in a dedicated repo (already done).
- Define template profiles and require template-scoped `AGENTS.md` (see 650/654).
- Add a memory registry index and enforce it in CI (new work).
- Add a curation queue workflow and require coding agents to append candidates instead of editing canonical memory.

### 6.2 Next (CLI-assisted retrieval)

- Add CLI commands that help agents discover relevant memory:
  - `ameide doctor` (already exists) expands to also report memory pointers and profile detection
  - optional: `ameide memory show <topic>` backed by the registry index

### 6.3 Future (platform memory as Elements)

- Model curated memory artifacts as Elements in the Transformation Domain.
- Serve them via Projection with citations and versioned references.
- Make embedding/graph retrieval the default; keep filesystem sources as fallback only.

## 7) Success criteria

- An agent can start from any template profile and immediately find the correct rules and the one no-brainer command.
- “What to run” and “what is forbidden” cannot silently drift from the CLI and templates.
- Retrieval always produces a small answer with links to authoritative sources.
- Deprecation is explicit and searchable; agents do not follow superseded instructions by default.
- Coding agents do not directly mutate canonical memory; they produce curation-queue candidates with citations/evidence.
- Business process and architecture artifacts (BPMN/ArchiMate/docs) are curated by humans in the element editor and stored as versioned elements.

## 8) References

- Agentic coding suite (profiles, CLI front doors, templates): `backlog/650-agentic-coding-overview.md`, `backlog/654-agentic-coding-cli-surface.md`
- Test evidence and citations: `backlog/430-unified-test-infrastructure-v2-target.md`, `backlog/527-transformation-projection.md`
- Element graph substrate: `backlog/300-400/303-elements.md`, `backlog/527-transformation-domain.md`
- Semantic retrieval posture: `backlog/535-mcp-read-optimizations.md`
