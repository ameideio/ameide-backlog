---
title: "656 — Agentic Memory MVP Increment 2 (v6): Operational End-to-End (Trust + Diffs + Knobs)"
status: draft
owners:
  - platform
  - transformation
created: 2026-01-18
related:
  - 656-agentic-memory-v6.md
  - 656-agentic-memory-implementation-v6.md
  - 694-elements-gitlab-v6.md
  - 496-eda-principles-v6.md
  - 527-transformation-v6-index.md
---

# 656 — Agentic Memory MVP Increment 2 (v6): Operational End-to-End (Trust + Diffs + Knobs)

Increment 2 keeps the same functional loop as Increment 1, but makes it operable for real teams:

- trust is explicit and explainable,
- “what changed?” becomes deterministic,
- proposals become PR-grade (metadata + checks),
- indexing and retrieval become observable (traceable),
- everything remains permission-trimmed and Git-citeable.

Normative posture: `backlog/656-agentic-memory-v6.md`, `backlog/656-agentic-memory-implementation-v6.md`, `backlog/694-elements-gitlab-v6.md`, `backlog/496-eda-principles-v6.md`.

## Functional storyline (what changes from Increment 1)

1) A user asks “what changed about the Testing SOP?” and gets a baseline diff view.
2) An agent proposes a change with explicit risk/impact metadata.
3) The curator sees deterministic diffs (Git diff + derived graph diff) plus pre-merge checks.
4) The system can explain why a result is trusted: “published baseline outranks evidence”.

## 1) Read loop upgrades (retrieval pipeline engineering)

### 1.1 Query intent routing (minimal)

Classify queries into at least:

- “truth” → `published` by default,
- “change” → baseline diff mode (`baseline_ref` vs `published`),
- “procedure” → bias to procedural docs/runbooks (repo-configured paths/tags).

### 1.2 Trust ranking (projection-owned, explainable)

Introduce a computed trust/provenance score and expose it in `why_included`.

Hard signals (minimum set):

- `published baseline membership`: strong positive
- `superseded/retired` markers (if present): strong negative
- `validator fail` evidence: strong negative or policy-blocking
- `explicit approval` (Domain governance): positive

### 1.3 Deterministic “what changed?” support (Git-native)

Support a “diff read model” that can answer:

- “what changed between `published` and `<baseline_ref>`?”
- “what changed between `published` and this MR?”

Minimum output:

- `base_commit_sha`, `compare_commit_sha`
- `changed_paths[]` with a deterministic diff summary
- citations that reference both commits (before/after anchors)

## 2) Propose loop upgrades (PR-grade proposals)

### 2.1 Structured change kinds (still Git-first)

Express proposals as file change sets, but classify them for review UX:

- `CREATE_FILE`, `UPDATE_FILE`, `DELETE_FILE`
- `ADD_RELATIONSHIP` / `REMOVE_RELATIONSHIP` (when relationship files exist)
- `UPDATE_LINKS` (inline link edits)

### 2.2 Required metadata (minimum recommended)

Add proposal metadata to support governance and later automation:

- `risk_level` (`low|med|high`)
- `impacted_areas[]` (freeform tags; module/domain names)
- `expected_outcome` (short)
- `target_read_context` (the base `commit_sha` and selector used)

## 3) Curate loop upgrades (diffs + checks)

### 3.1 Deterministic diffs

Provide diffs that are deterministic against pinned bases:

- Git diff for file changes (`base_commit_sha` → `proposal_commit_sha`)
- Derived graph diff (optional): link/relationship additions/removals computed by the Projection

### 3.2 Pre-merge checks (gates before publish)

Before publish (merge to `main`), run and surface:

- schema validation (required metadata present),
- profile validators (when applicable),
- “impact radius” view (what links to the changed files / entities),
- conflict detection (“another open MR touches same path range”, “proposal is stale”).

## 4) Publish loop upgrades (policy stub)

Add a minimal policy stub so Increment 3 can be incremental:

- “high risk proposals require N approvers” (even if N=1 initially)
- “validator failures block publish” (policy-controlled)

## 5) Ingest/index loop upgrades (scheduled + idempotent)

Make indexing repeatable and cheap:

- scheduled reindex of `published` baseline,
- incremental indexing keyed by commit SHA (only reindex changed paths),
- deterministic change detection (normalized content hashing to avoid churn-only updates).

## 6) Quality loop upgrades (golden queries + observability)

### 6.1 Golden query suite (small, real)

For the “Testing SOP” scenario:

- “how do I run Phase 0/1/2 tests?” returns the published SOP and cites the published commit.
- “what changed about the testing SOP?” returns a diff view and cites both commits.
- Regression: when both evidence and published truth match, published truth outranks evidence (unless explicitly asked for evidence).

### 6.2 Retrieval trace logging

Store a trace record per retrieval:

- query → routing decision → candidate set → filters/authz → rerank → graph expansion → citations

This makes retrieval debuggable and safe to change.
