# 656 — Agentic Memory MVP Increment 2: Operational End-to-End (Trust + Diffs + Pipeline Knobs)

**Goal:** keep the same end-to-end loops as Increment 1, but make the system operationally usable for real teams:

- deterministic diffs and review ergonomics
- explicit trust ranking (published > curated > draft/proposal > ingestion)
- ingestion becomes scheduled/idempotent
- retrieval pipeline adds routing + reranking (still within strict permission trimming)

---

## 0) Implementation anchors (primitives + proto)

Increment 2 expands the same primitives from Increment 1:

- **Domain primitive:** adds PR-grade proposal metadata handling and deterministic diff/rebase/accept semantics; publishes auditable facts.
- **Projection primitive:** adds trust ranking + reranking + diff read models and emits retrieval trace logs.
- **Integration/MCP + CLI:** switch agents to the “front door” retrieval/propose APIs so no tool re-implements ranking/diff logic.

See `backlog/656-agentic-memory-implementation.md` §5 for the “by primitive” implementation spec.

## 1) Read loop upgrades (retrieval pipeline engineering)

### 1.1 Query intent routing (minimal)

Classify:

- semantic truth (“what is …?”) → published/baseline mode
- procedural (“how do I …?”) → procedural-first
- change (“what changed?”) → baseline diff mode

Reserve a slot for query rewriting (even if minimal in v2):

- acronym expansion and aliasing (repo-configured)
- optional layer hints (e.g., bias to Application vs Technology if the repo defaults are known)

### 1.2 Trust ranking (computed, explainable)

Introduce a computed trust/provenance score (projection-owned) and expose it in `why_included`:

- baseline status + lifecycle state
- recency + validity window (if provided)
- approvals present
- evidence count / references
- validator results (pass/fail)

**Contract detail (stability):** the score MUST include a small set of “hard signals” with clear polarity:

- baseline membership: strong positive
- superseded/retired: strong negative
- validator fail: strong negative (or block by policy)
- explicit approval: positive

### 1.3 Reranking (reduce noisy top-k)

Add a rerank stage (cross-encoder or model-based) after candidate recall, before graph expansion.

### 1.4 Deterministic “what changed?” support

- Add a read_context selector for explicit baseline refs.
- Add baseline compare support for documents (version-to-version diff) and graph changes (relationship diff).

---

## 2) Propose loop upgrades (structured proposals)

### 2.1 Structured change kinds

Support proposal payloads beyond “replace body”:

- `CREATE` element
- `UPDATE` element body/metadata (versioned)
- `LINK` / `UNLINK` relationships (with deterministic targets)
- `SUPERSEDE` (explicitly link new → old)

### 2.2 Required metadata for PR-grade governance

Expand proposal metadata (minimum recommended):

- `proposal_status` state machine: `DRAFT (optional) → PROPOSED → TRIAGED → IN_REVIEW → ACCEPTED` + terminal states
- `risk_level` (low/med/high)
- `impacted_domains[]`
- `target_read_context` (selector vocabulary aligned to 527: `published|baseline_ref|head|version_ref`)
- `security_classification` / visibility tag (if applicable)
- `expected_outcome` (short; optimized for curator triage)

**Note:** “promoted” is not a proposal state. It is a **publish outcome** (“the accepted version is now included in the published baseline pointer”).

---

## 3) Curate loop upgrades (diffs + checks)

### 3.1 Deterministic diffs

Provide diffs that are deterministic against pinned targets:

- doc diff: target version body vs proposed body
- graph diff: relationship add/remove/update list

### 3.2 Pre-merge checks (gates before publish)

Before a proposal can be promoted:

- schema validation (required metadata present)
- relationship validity (basic matrices for ArchiMate/BPMN profiles as available)
- “impact radius” view: what references this target (projection query)
- security tag presence (if required by policy)

Add conflict detection outputs (informational in v2):

- “another open proposal targets this element/version”
- “proposal was rebased from an older target”

---

## 4) Publish loop upgrades (baseline ergonomics)

- Baseline compare becomes a first-class read model (“what changed between published and this proposal?”).
- Promotion emits an auditable event trail that can be surfaced in timelines.

Add a minimal policy stub so Increment 3 is incremental, not a redesign:

- “high risk proposals require N approvers” (even if N=1 initially)

---

## 5) Ingest loop upgrades (idempotent, scheduled, duplicate hygiene)

### 5.1 Idempotent ingestion

- scheduled backlog mirror (or operator task)
- only create new versions when content changes

**Define “content changed” deterministically:** compute hashes on normalized bodies (e.g., normalize line endings, trim trailing whitespace) to avoid churn-only versions.

### 5.2 Near-duplicate detection (copy hygiene)

- detect identical/near-identical markdown bodies across ingestion sources
- mark duplicates as mirrors and link them to the canonical ingestion element/version

---

## 6) Quality loop upgrades (golden queries + observability)

### 6.1 Golden query suite (small, real)

- per domain/layer: query → expected element/version ids
- assert trust ordering (published beats ingestion)
- assert “what changed” queries return diff items

Add an explicit regression invariant:

- when both ingested evidence and published truth match a query, published truth MUST outrank ingestion evidence (unless the query explicitly asks for ingestion evidence).

### 6.2 Retrieval trace logging

Log the pipeline:

- query → routing decision → candidate set → filters/ACL decisions → rerank → graph expansion → context → citations

This turns retrieval into operable infrastructure.

**Reference scenario (v2):** “How do I run tests?”

Expand the Increment 1 scenario with:

- Golden query: “how do I run Phase 0/1/2 tests?” returns the published SOP and cites `{repository_id, element_id, version_id}`.
- Golden query: “what changed about the testing SOP?” returns baseline diff items and cites both versions.
- Regression: when ingestion evidence and published SOP both match, published SOP outranks evidence (unless query asks for evidence).
- Trace: the retrieval trace for these queries is captured and stored with the run (routing → filters → rerank → graph expansion → citations).
