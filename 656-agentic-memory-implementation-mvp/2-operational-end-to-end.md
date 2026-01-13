# 656 — Agentic Memory MVP Increment 2: Operational End-to-End (Trust + Diffs + Pipeline Knobs)

**Goal:** keep the same end-to-end loops as Increment 1, but make the system operationally usable for real teams:

- deterministic diffs and review ergonomics
- explicit trust ranking (published > curated > draft/proposal > ingestion)
- ingestion becomes scheduled/idempotent
- retrieval pipeline adds routing + reranking (still within strict permission trimming)

---

## 1) Read loop upgrades (retrieval pipeline engineering)

### 1.1 Query intent routing (minimal)

Classify:

- semantic truth (“what is …?”) → published/baseline mode
- procedural (“how do I …?”) → procedural-first
- change (“what changed?”) → baseline diff mode

### 1.2 Trust ranking (computed, explainable)

Introduce a computed trust/provenance score (projection-owned) and expose it in `why_included`:

- baseline status + lifecycle state
- recency + validity window (if provided)
- approvals present
- evidence count / references
- validator results (pass/fail)

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

- `proposal_status` state machine: `PROPOSED → TRIAGED → IN_REVIEW → ACCEPTED → PROMOTED` + terminal states
- `risk_level` (low/med/high)
- `impacted_domains[]`
- `target_read_context` (baseline id or published)
- `security_classification` / visibility tag (if applicable)

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

---

## 4) Publish loop upgrades (baseline ergonomics)

- Baseline compare becomes a first-class read model (“what changed between published and this proposal?”).
- Promotion emits an auditable event trail that can be surfaced in timelines.

---

## 5) Ingest loop upgrades (idempotent, scheduled, duplicate hygiene)

### 5.1 Idempotent ingestion

- scheduled backlog mirror (or operator task)
- only create new versions when content changes

### 5.2 Near-duplicate detection (copy hygiene)

- detect identical/near-identical markdown bodies across ingestion sources
- mark duplicates as mirrors and link them to the canonical ingestion element/version

---

## 6) Quality loop upgrades (golden queries + observability)

### 6.1 Golden query suite (small, real)

- per domain/layer: query → expected element/version ids
- assert trust ordering (published beats ingestion)
- assert “what changed” queries return diff items

### 6.2 Retrieval trace logging

Log the pipeline:

- query → routing decision → candidate set → filters/ACL decisions → rerank → graph expansion → context → citations

This turns retrieval into operable infrastructure.

