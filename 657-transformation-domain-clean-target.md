# 657 — Transformation Domain Clean Target (Memory-First, EDA-Correct)

**Status:** Draft (target-state contract)  
**Priority:** High  
**Related:** `backlog/300-400/303-elements.md`, `backlog/496-eda-principles-v3.md`, `backlog/527-transformation-implementation-migration.md`, `backlog/656-agentic-memory.md`, `backlog/656-agentic-memory-implementation.md`

---

## 0) Why this backlog exists

We already have a working Transformation Domain primitive (`primitives/domain/transformation`) and Projection primitive (`primitives/projection/transformation`). But the current shape still contains “bridge” compromises and missing hard boundaries that will turn into permanent architectural debt if we keep shipping incrementally.

This backlog defines the **clean target** (because we are not in production and can break compatibility):

- **Memory-first:** the Transformation capability is the canonical substrate for organizational memory (Elements + Versions + Relationships + Baselines).
- **EDA-correct (496 v3):** ingestion and read models are derived from **domain facts delivered via Kafka topics** (CloudEvents + Protobuf); no DB tailing as the default posture; Kafka topics are a contract surface in v3.
- **Security-correct:** auth/RBAC is enforced at the **gRPC boundaries**, not only in HTTP routes or tool allowlists.
- **Ontology-first:** ArchiMate-first when possible; backlogs are ingestion only (per 656).

---

## 1) Decisions (locked for the refactor)

1. **Single repository of truth for memory.** Queue vs published is expressed via lifecycle/status + baselines in the same repository (no “second memory repo”). (`backlog/656-agentic-memory.md`)
2. **Elements-only canonical storage.** No parallel canonical model for “docs vs BPMN vs diagrams”. (`backlog/300-400/303-elements.md`)
3. **Projection is the only read path.** Browse/search/history/diff/context assembly are projection-owned. (`backlog/527-transformation-projection.md`)
4. **Kafka-first ingestion for projections (496 v3).** Projections receive domain/process facts via Kafka consumer groups (CloudEvents + Protobuf). Kafka topics are a contract surface. (`backlog/496-eda-principles-v3.md`)
5. **Auth at gRPC boundaries.** Domain and Projection gRPC servers MUST enforce authN/authZ via interceptors (no “trust the caller” posture).

---

## 2) Target architecture (clean separation)

### 2.1 Domain primitive (single-writer, canonical state)

**Owns:**
- Element repositories (Elements, Versions, Relationships, Workspace tree assignments as defined in 303).
- Governance (baselines, approvals, promotion pointers).
- Curation queue objects (proposal/review/decision elements per 656).

**Guarantees:**
- “Facts after persist” via transactional outbox (496).
- Deterministic reference pinning for reproducible reads (`REFERENCE.metadata.target_version_id`, 303/656).
- No query-oriented “search” behavior in the Domain (projection responsibility).

### 2.2 Projection primitive (idempotent consumers, all reads)

**Owns:**
- Workspace browsing read models.
- `read_context` + `citations[]` in every query response shape where decisions depend on it.
- Hybrid retrieval (keyword + vector + graph expansion) per `backlog/656-agentic-memory.md`.
- Curation queue UX read models (proposal list/diff/validator results).

**Guarantees:**
- Idempotent application of facts (inbox; 496).
- Query-time authorization (scope + membership + visibility) consistent with Domain auth.

### 2.3 Integration primitive (MCP as the agent interface)

**Owns:**
- MCP tools/resources that wrap projection reads and domain writes:
  - read: `memory.search`, `memory.get_context`, `memory.get_element`
  - write: `memory.propose` (proposal-only writes for execution agents)

**Guarantees:**
- Tool grants enforce “execution agents propose, curators publish” (656).

---

## 3) Remove “bridge/posture drift” (no patchwork)

### 3.1 Deprecate DB tailing as a normative ingestion path

`primitives/projection/transformation/cmd/relay` (outbox tail) MUST be treated as:
- a local dev convenience, or
- a one-off recovery tool

It MUST NOT be the default posture for long-running environments once broker wiring is in place.

**Clean target (496 v3):** projection receives facts via Kafka consumer groups; topic names are a contract surface (v3).

### 3.2 Remove proto/naming duplication instead of “mapping forever”

Clean target requires a single external vocabulary:
- repository scope is `{tenant_id, organization_id, repository_id}` (no `graph_id` aliases)
- “Enterprise Knowledge / Elements” is a capability surface, not multiple proto namespaces that describe the same model

If multiple proto packages currently represent the same noun set, we converge them (rename/remove), rather than building permanent adapters.

### 3.3 Make authorization non-optional

Auth must not live “at the edges only” (Next.js routes, inference tool allowlists). The clean target is:
- **Domain gRPC:** authorize every command (including baseline promotion).
- **Projection gRPC:** authorize every read (including retrieval/context assembly).
- **Integration/MCP:** authorize tool invocation and scope every call.

---

## 4) Work plan (increments implement all loops)

Rule: each increment must be a working, end-to-end implementation of **all loops**, with increasing capability (no “we only built ingestion this increment”).

### 4.1 The loops (always present)

1. **Read loop:** agents/humans retrieve memory with `read_context + citations`.
2. **Propose loop:** execution agents can only write proposals (never publish).
3. **Curate loop:** curators review proposals, see diffs vs version-pinned targets, accept/reject.
4. **Publish loop:** accepted changes are promoted into baselines / published truth.
5. **Ingest loop:** raw sources (backlogs/chat/imports) enter as ingestion elements and are linked as evidence.
6. **Quality loop:** regression harness proves retrieval + governance behaviors and prevents drift.

### Increment 1 — Minimal end-to-end (safe + citeable)

- Read: keyword-only context retrieval via projection, but always returns `{element_id, version_id}` citations.
- Messaging: facts are delivered to projections via Kafka as CloudEvents+Protobuf; inbox dedupe keys on CloudEvents `id` (496 v3).
- Propose: `memory.propose` creates `ameide:curation.proposal` + `TARGETS_VERSION` pinning.
- Curate: curator can list proposals and accept/reject (diff may be coarse: “full replacement body”).
- Publish: curator can promote accepted versions into a baseline; `published` read_context works.
- Ingest: backlog markdown can be mirrored into `ameide:ingest.backlog_md` elements (manual trigger is fine).
- Quality: smoke tests cover: retrieve → propose → accept → promote → retrieve(published).

### Increment 2 — Operational end-to-end (trusted ranking + better diffs)

- Read: trust ranking is implemented and visible: published > curated > draft/proposal > ingestion.
- Propose: proposal payload supports structured change sets (create/link/unlink/supersede), not only “replace body”.
- Curate: deterministic diffs vs `TARGETS_VERSION` (field-aware diffs for docs; relationship diffs for graph changes).
- Publish: baseline compare + “what changed” evidence is queryable; publish/promotion actions are RBAC-gated.
- Ingest: ingestion is scheduled/idempotent; provenance includes source commit sha + path; chat intake can create ingestion/proposal elements.
- Quality: golden-query regression harness exists (even if small), plus governance contract tests.

### Increment 3 — Scaled end-to-end (hybrid retrieval + CI gates)

- Read: hybrid retrieval (vector + keyword) + bounded graph expansion; `read_context` selectors include baseline refs.
- Propose: proposals can target multiple elements and include validator outputs as evidence.
- Curate: validators (ArchiMate/BPMN profile checks) run as gates and attach evidence to proposals/decisions.
- Publish: promotions are policy-driven (evidence-driven gates); publish actions emit auditable facts.
- Ingest: multi-format imports (BPMN/ArchiMate/docs) enter as evidence, and curated meaning lives in the ontology graph.
- Quality: retrieval quality metrics and drift detection are CI-gated; end-to-end workflow tests run deterministically.

---

## 5) Definition of Done (for this backlog)

This backlog is “done” when:

- Domain + Projection enforce auth at gRPC boundaries (no bypass paths).
- Projection ingests domain facts from Kafka as the normative posture (no DB tailing required).
- Agents consume organizational memory via MCP tools that return `read_context + citations`.
- Execution agents cannot mutate published truth and can only write proposals.
- Proto/naming drift that requires permanent adapters is eliminated (repository_id only; no `graph_id` surface).

---

## 6) Backlog cross-references to update (follow-ups)

This backlog is intentionally narrow; it points to the places that should reference it:

- `backlog/527-transformation-projection.md`: mark DB relay as deprecated/default-off; Kafka consumer as default.
- `backlog/527-transformation-domain-v2.md`: link 656 as the memory contract; state proposal-only writes for execution agents.
- `backlog/527-transformation-implementation-migration.md`: add a WP/plateau for auth + Kafka ingestion + retrieval contract.
- `backlog/656-agentic-memory-implementation.md`: treat 657 as the “clean target refactor” prerequisite for safe memory.
