# 656 — Agentic Memory: Organizational Memory, Curation, and Retrieval Contract

**Epic:** Agentic Platform Foundations  
**Status:** Draft (contract)  
**Priority:** High  

---

## 0) Summary

This backlog defines the **platform contract for organizational memory** used by humans and agents:

- **What “memory” is:** a single repository-scoped, versioned **element graph** plus **baselines** that represent “published / normative truth”.
- **How memory evolves:** humans curate directly; execution agents propose changes into the **same repository** using lifecycle/status; curators review and promote reviewed changes into baselines.
- **How memory is organized:** **ArchiMate-first** when possible: the ontology (type keys + semantic relationships) is the primary structure; documents/BPMN/diagrams are additional views/evidence linked into the same graph.
- **How memory is retrieved:** through **projection-owned hybrid retrieval** (semantic + relationship-aware graph expansion) that returns **read_context with citations**, exposed to agents primarily via **MCP tools**.
- **AGENTS.md scope:** `AGENTS.md` is **coding-agent-only** guidance; it must not be treated as “organizational memory”. Agents interact with organizational memory only via the contracts defined here (read_context + proposals).

This is the canonical place where the agentic coding suite and chat assistants learn **how to read, cite, and propose updates** to the shared organizational memory.

---

## 1) Goals

1. Maintain a **single, trustworthy organizational memory** that lets humans and agents consistently define, evolve, and govern:
   - business processes (e.g., BPMN)
   - architecture artifacts (e.g., ArchiMate)
   - supporting documents/decisions
   - (explicit) *legacy backlog markdown* as **ingestion input**, not as the long-term organization mechanism

2. Let humans curate memory **directly in the element editor**, across multiple artifact types, with:
   - clear lifecycle (draft → published → superseded)
   - auditability of edits, reviews, and promotions

3. Ensure agents can **reliably retrieve the right context** for tasks/conversations via:
   - relationship-aware retrieval (graph expansion)
   - modern retrieval pipeline (hybrid search + reranking + query routing)
   - stable, citeable references (element/version citations)

4. Prevent low-quality or conflicting updates by execution-focused agents by:
   - routing writes into a **curation queue view** (status/lifecycle-based)
   - requiring **review + promotion** before canonical “published truth” changes

---

## 2) Non-goals

1. Training or fine-tuning foundation models from organizational memory.
2. A second canonical storage system (no “memory database” outside the element substrate).
3. “Prompt-only” guardrails (permissions must be enforced by RBAC/policy and tool grants).
4. Cross-tenant retrieval or cross-org retrieval (memory remains repository-scoped).
5. Treating markdown backlogs as a durable “memory UX” surface (backlogs are an ingestion source; curated memory lives in the ArchiMate-first graph).

---

## 3) Reference architecture links (how 656 fits)

This backlog is intentionally **303-first** and **527/535-shaped**.

### 3.1 Canonical substrate (303-first)

- **303-elements.md** is the authoritative substrate and must be treated as the source of truth for:
  - **Elements-only canonical storage** (no separate canonical model per notation; attachments/evidence are non-canonical blobs referenced from elements).
  - **Version-pinned references** via `REFERENCE.target_version_id` for reproducible reads.
  - **Baselines + promotion** as the mechanism for “published / normative truth”.

(These are the invariants 656 relies on; 656 does not redefine them.)

### 3.2 Retrieval is a Projection concern (527/535-shaped)

- **527-transformation-projection.md** (line 20): projection owns browse/search/history/diff.
- **535-mcp-read-optimizations.md** (line 1): embedding/vector/hybrid retrieval is a projection capability; agents consume it through `read_context` + citations.
- **657-transformation-domain-clean-target.md**: defines the required “clean target” posture (Kafka-first ingestion + gRPC auth boundaries) so retrieval and memory writes are safe and EDA-correct.

### 3.3 Agentic coding suite integration
- **650-agentic-coding-overview.md**, **653-agentic-coding-test-automation.md**, **654-agentic-coding-cli-surface.md**: depend on memory.
- **656** defines the memory contract those agents must use (read_context + propose-to-queue), so they stop treating memory as an external ad-hoc store.

### 3.4 Chat assistant ingestion bridge
- **367-0-feedback-intake.md** provides the pattern: AI-assisted capture → persisted as elements with provenance.
- **656** generalizes this to: chat → structured, versioned elements (draft) → review/promotion.

### 3.5 Guardrails and permissions prerequisites
- **329-authz.md** and **322-rbac.md**: org isolation + RBAC.
- **535-mcp-read-optimizations.md**: requires auth at query time to prevent data leakage.

### 3.6 Runtime realization (primitives, not a standalone system)

This contract is implemented **by Ameide primitives** (Application layer). “Memory” is not a new storage system; it is a governed use of the canonical element substrate.

**Canonical capability that implements memory (today):** Transformation.

**Primitives involved (ideal target):**

- **Domain primitive (canonical writes):** persists Elements/Versions/Relationships/Baselines + proposal/decision artifacts and emits facts via outbox.
  - Implementation: `primitives/domain/transformation`
  - Proto contracts live under: `packages/ameide_core_proto/src/ameide_core_proto/transformation/**`
- **Projection primitive (all reads):** browse/search/history/diff + `read_context + citations` + retrieval pipeline (hybrid + graph expansion) and projection-owned trust scoring.
  - Implementation: `primitives/projection/transformation`
- **Process primitive (governance orchestration, optional plateau):** when promotion/review is modeled as BPMN, the process engine coordinates; primitives do side effects; domains persist truth.
  - Reference posture: `backlog/527-transformation-capability-v2.md`
- **Integration primitive (agent interface):** MCP tools/resources call projection reads + domain writes; agents do not speak directly to databases.
  - Existing shape: `backlog/534-mcp-protocol-adapter.md`
- **Agent primitive (execution):** coding/SRE/GitOps agents consume `read_context` and are **proposal-only** writers; they do not publish.
- **UISurface primitive (human UI):** element editor + governance UI for proposal review/promotion.

### 3.7 Proto + event identity conventions (repo-aligned)

This repository already expresses “what happened” and “where to deliver it” in proto options:

- Proto packages for Transformation commonly use a reverse-DNS prefix `io.ameide.*` (example: `io.ameide.transformation.knowledge.v1`).
- Domain facts encode:
  - a stable semantic identity (`stable_type`)
  - a logical delivery stream reference (`stream_ref`)

Example (existing, in-tree): `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_facts.proto`.

**How this interacts with EDA (ideal state):**

- Inter-primitive messages are carried in a **CloudEvents envelope** (see the active `backlog/496-eda-principles-*` spec).
- `ce-type` SHOULD be the proto-declared stable semantic identity (today: `stable_type`).
- `ce-id` is the global idempotency key; consumers dedupe on it (inbox).

656 does not require choosing one delivery plane (Broker/Trigger vs direct Kafka topics). It requires:

- deterministic ids, dedupe, and replay semantics
- stable semantic identities for messages
- projection rebuildability from facts

### 3.8 Deprecated / superseded documents (keep for history)

The following documents are kept for historical context but should not be treated as implementation targets for organizational memory:

- `backlog/300-400/304-context.md` (legacy “graph context” terminology; predates baseline-first memory + citation discipline).
- `backlog/300-400/352-proto-review-graph.md` (implementation parity work; not an architecture target; “graph_id” is legacy naming).

The following documents remain relevant, but must be read through the 303/656 lens:

- `backlog/300-400/329-authz.md` (authz architecture; some schema examples use legacy nouns/tables).
- `backlog/534-mcp-protocol-adapter.md` (MCP adapter shape; `read_context` + citations must align to 303/656).
- `backlog/535-mcp-read-optimizations.md` (projection-owned retrieval; vector is derived; defaults and citation format must align to 656 for agent memory tools).

---

## 4) Canonical organizational memory model (elements + versions + relationships)

### 4.1 Definition of “memory”

**Organizational memory** is:

1. A repository-scoped element graph (elements, versions, relationships).
2. One or more **baselines** that define what is “published / normative truth”.

Embeddings, vector indexes, search rankings, and read_context assembly are **projection outputs**, not canonical state.

### 4.1.1 Memory types (semantic vs episodic vs procedural)

Organizational memory must explicitly separate “what is true” from “what happened” and “how to do things”:

- **Semantic memory (canonical truth):** published/baseline-scoped elements that define architecture, policies, standards, and business/process meaning.
- **Procedural memory (how-to):** runbooks/SOPs/checklists that are still subject to curation and baselines, but retrieved with “how do I …” intent.
- **Episodic memory (observations/audit):** run logs, agent outputs, timelines, investigations, and decision trails. These must be linkable as provenance/evidence, but should not become canonical truth by default.

Retrieval MUST bias toward semantic/procedural published truth for “what is” / “how to”, and treat episodic items as supporting evidence unless explicitly requested.

### 4.1.2 Machine-checkable trust contract (version drift prevention)

The system must make “what is canonical” **machine-checkable**, not interpretive:

- Every retrieval must make baseline/version selection explicit (`read_context`).
- Every returned item must be version-pinned (`{repository_id, element_id, version_id}` citations).
- Superseded content must be detectable and suppressible by default (e.g., `supersedes`/`superseded_by` semantics or explicit lifecycle).

Optional but recommended validity semantics (version metadata; used for filtering/ranking):

- `effective_from`, `effective_to`
- `applies_to_environment` (e.g., `local|dev|staging|prod`)
- `applies_to_region` / `applies_to_tenant_segment` (as needed)

These fields exist to mitigate “accurate-but-outdated” answers (version drift) and to allow deterministic scoping beyond semantic similarity.

### 4.1.3 Trust + provenance score (computed signal)

Retrieval ranking must be biased by a computed trust/provenance score (projection-owned), derived from:

- lifecycle/baseline status (published > curated > draft > ingestion)
- review/approval presence (who approved; when)
- freshness/recency (and validity window if defined)
- number and quality of supporting references/evidence
- conformance/validator results (BPMN/ArchiMate/profile checks)

This score is a ranking signal and an audit surface (explain “why this was included”); it is not canonical state.

### 4.2 Element kinds and type_keys (recommended)

All of the following are stored as Elements; differences are expressed via `element_kind` + namespaced `type_key`:

- **ArchiMate-first canonical nodes/edges (preferred)**
  - Canonical “memory meaning” SHOULD be expressed using `archimate:*` `type_key`s and standards-compliant relationship verbs when possible.
  - This is what enables layer-aware organization and traversal without inventing a parallel “memory taxonomy”.

- **Documents / Decisions / Policies (supporting)**
  - `element_kind`: `DOCUMENT`
  - `type_key`: `ameide:doc`, `ameide:decision`, `ameide:policy`
  - These MUST be linked into the ontology graph via semantic/reference edges to relevant ArchiMate elements (so they’re not orphan blobs).

- **BPMN artifacts**
  - `element_kind`: `MODEL` / `VIEW` (depending on your platform convention)
  - `type_key`: `bpmn:*` (standards-compliant profile) or `ameide:bpmn:*` (extended)
  - BPMN elements SHOULD be connected to ArchiMate via semantic links where possible (e.g., process realization/traceability).

- **ArchiMate artifacts**
  - `element_kind`: `MODEL` / `VIEW`
  - `type_key`: `archimate:*` (standards-compliant profile) or `ameide:archimate:*` (extended)

- **Ingestion-only (legacy backlogs, captured chat transcripts, raw imports)**
  - `element_kind`: `DOCUMENT` (or `BINARY` when appropriate)
  - `type_key`: `ameide:ingest.backlog_md`, `ameide:ingest.chat`, `ameide:ingest.import`
  - These are evidence/sources for curators and agents, not the target structure for organizational memory.

- **Governance / curation records** (defined in §6; stored in the same repository)
  - `element_kind`: `DOCUMENT` (or a dedicated `RECORD` kind if you support it)
  - `type_key`: `ameide:curation.proposal`, `ameide:curation.review`, `ameide:curation.decision`

### 4.3 Relationship kinds and usage

Relationship kinds remain exactly:

- `CONTAINMENT`: internal structure (document sections, model hierarchies)
- `SEMANTIC`: domain meaning (realizes, depends-on, controls, etc.)
- `REFERENCE`: evidence/attachments, version-pinned references, workflow anchors

**Rule:** internal structure belongs in `CONTAINMENT` relationships, not in workspace folders.

---

## 5) Ingestion paths (how memory gets created/updated)

This section defines *write paths* into organizational memory without mixing in retrieval.

### 5.1 Human curation in the element editor

- Humans edit elements directly → a new `ElementVersion` is created.
- Lifecycle changes (draft/published/superseded) are implemented via:
  - metadata state on versions and/or
  - baseline promotion and supersedes relationships

### 5.2 Import pipelines for external formats (BPMN, ArchiMate, documents)

- External files are first stored as **attachments/evidence** (non-canonical blobs).
- Import transforms attachments into canonical elements/relationships.
- The attachment remains linked as evidence to the resulting element versions.

### 5.3 Chat-to-draft capture (assistant intake bridge)

Generalize the feedback-intake pattern to all memory domains:

1. Capture a chat interaction into an **Intake element** (draft) with provenance:
   - source channel (chat), participants, timestamps, agent/tool identifiers
2. Transform Intake → draft canonical elements (e.g., a BPMN draft process + related doc).
3. Route to curation workflow (review/promotion) before it becomes published truth.

### 5.4 Execution agents (coding, SRE, GitOps) write via proposals only

- Execution-focused agents MUST NOT write directly to canonical “published truth”.
- They produce **proposal elements** into the curation queue (see §6).
- Execution agents MAY create “curated but not approved yet” content (status-limited), so retrieval can include it without waiting for full human approval.

---

## 6) Curation queue (first-class governance objects)

This section defines the human-in-the-loop workflow that protects canonical memory quality.

Key rule: there is **one repository**; “queue vs published” is expressed via **lifecycle/status** and baselines, not by splitting repositories.

### 6.1 Core objects

#### Proposal element
- `type_key`: `ameide:curation.proposal`
- Purpose: represent a suggested change, with evidence, provenance, and deterministic targets.

Required metadata (minimum):
- `proposal_status`: `DRAFT | IN_REVIEW | ACCEPTED | REJECTED | WITHDRAWN`
- `proposed_by`: `{actor_type: human|agent, actor_id, tool_id}`
- `proposal_summary`: short text
- `proposal_change_kind`: `CREATE | UPDATE | LINK | UNLINK | SUPERSEDE`
- `created_at`, `updated_at`

#### Review/Decision elements
- `type_key`: `ameide:curation.review` and/or `ameide:curation.decision`
- Purpose: capture reviewer discussion, validation results, acceptance/rejection rationale.

### 6.2 Required relationships (keys and semantics)

All relationship `type_key`s are in the `ameide:curation.*` namespace.

1. `TARGETS_VERSION`
   - `relationship_kind`: `REFERENCE`
   - `type_key`: `ameide:curation.targets_version`
   - From: proposal → target element
   - **Requires** `target_version_id` to ensure deterministic review and reproducible reads.

2. `PROPOSES`
   - `relationship_kind`: `CONTAINMENT` or `REFERENCE` (implementation choice)
   - `type_key`: `ameide:curation.proposes`
   - From: proposal → one or more “change payload” sub-elements or attachments
   - Used to attach structured patches, diff artifacts, generated models, or supporting docs.

3. `ACCEPTED_AS`
   - `relationship_kind`: `REFERENCE`
   - `type_key`: `ameide:curation.accepted_as`
   - From: proposal → accepted element(s)
   - **Requires** `target_version_id` to pin the exact new version(s) created during acceptance.

4. `REJECTED_BECAUSE`
   - `relationship_kind`: `REFERENCE`
   - `type_key`: `ameide:curation.rejected_because`
   - From: proposal → decision/reason element

### 6.3 Lifecycle and state transitions

- **Create proposal** (agent/human) → `proposal_status = DRAFT`
- **Submit for review** → `IN_REVIEW`
- **Accept** → create new `ElementVersion`(s) and link via `ACCEPTED_AS` → `ACCEPTED`
- **Reject** → link decision via `REJECTED_BECAUSE` → `REJECTED`
- **Withdraw** → `WITHDRAWN`

### 6.4 Promotion to published truth

Acceptance does not automatically equal “published”.

- Acceptance creates canonical versions.
- A curator (or governed automation) then **promotes** those versions into the relevant baseline(s).
- Promotion is auditable and repeatable.

### 6.5 Curation queue UX (projection)

A dedicated projection powers the queue:

- list proposals by status, target, proposer, age, layer
- show deterministic diff: proposal payload vs `TARGETS_VERSION`
- run profile validators (BPMN/ArchiMate validity matrices) and show results
- allow actions: approve/apply, request changes, reject
- emit process facts for audit timelines

### 6.6 Proposal state machine (PR-grade governance)

Proposals are not informal. The curation queue MUST be a defined state machine with required metadata sufficient for:

- accountability (who proposed; provenance)
- scoping (target baseline/read_context; impacted domains)
- risk signaling (e.g., `risk_level`, `security_classification` where applicable)

Minimum recommended state progression:

- `PROPOSED → TRIAGED → IN_REVIEW → ACCEPTED → PROMOTED`
- plus terminal states: `REJECTED` (with rationale), `WITHDRAWN`

---

## 7) Retrieval and read_context contract (projection-owned)

This section defines the *read path* contract that agents must use.

### 7.1 Defaults and baseline selection

- Retrieval returns **all relevant context**, but MUST give precedence to normative truth:
  1. **Published/normative** (published pointers and/or a promoted baseline)
  2. **Curated but not approved yet** (reviewed/validated, not promoted)
  3. **Draft/proposals**
  4. **Ingestion-only sources** (legacy backlog markdown, raw chat transcripts)

Default mode for agents is “include all, rank by trust”, so knowledge does not stall while still privileging approved truth.

### 7.2 Relationship-aware hybrid retrieval

The retrieval projection implements a retrieval **pipeline** (not “vector search + chunks”):

1. **Query intent + routing**
   - classify intent: “what is true now?” vs “how do I?” vs “what changed?” vs “incident/trace”
   - classify version intent: `published/baseline` default, explicit baseline/version when provided

2. **Candidate recall (hybrid)**
   - keyword/BM25 + vector search
   - strict metadata filtering (baseline/lifecycle/validity/ACL)

3. **Re-ranking**
   - cross-encoder or model-based reranking to reduce noisy top-k

4. **Graph expansion (relationship-aware)**
   - follow `CONTAINMENT` for local structure (sections, sub-processes)
   - follow selected `SEMANTIC` edges for domain neighborhood
   - cap expansion by depth/edge budget to control token/latency

5. **Context distillation**
   - compress retrieved items into a smaller, higher-signal bundle (projection-owned), retaining citations and relationship traces

### 7.3 read_context API/tool surface (minimum)

**Request**
- `repository_id`
- `query` (text)
- `baseline`: `PUBLISHED | DRAFT | BASELINE_ID`
- `version_intent` (optional): `CURRENT | AS_OF | DIFF` (projection-owned classifier may infer)
- `filters` (optional):
  - `type_keys[]`, `element_kinds[]`
  - validity filters: `effective_at`, `applies_to_environment`, `applies_to_region`
- `graph_expansion`:
  - `depth`, `edge_types[]`, `max_nodes`
- `budget`:
  - `max_tokens`, `max_items`

**Response**
- `context_items[]` where each item includes:
  - `element_id`, `version_id`
  - `title/name`
  - `excerpt` (or structured payload summary)
  - `why_included` (relevance + trust score components + edge-path)
  - **citations**: stable identifiers that pin the exact element version(s)
  - optional: `relationship_trace[]` (which edges were traversed; explainable graph expansion)

Agents MUST surface citations in their outputs when they claim facts from memory.

### 7.4 ArchiMate-first organization (layers come from the ontology)

Primary organization should come from the ontology itself:

- The ArchiMate layer and semantics come from `archimate:*` `type_key`s and semantic relationship verbs.
- Documents/BPMN/diagrams SHOULD be connected to ArchiMate elements so traversal yields “why is this relevant?” without relying on external folder taxonomy.
- `metadata.architecture_layer` MAY exist for ingestion-only sources, but it is not the primary long-term organization mechanism.

---

## 7.5 MCP-first access for agents

Organizational memory is exposed to agents via **MCP tools** (projection-backed), not by “grep the repo”:

- `memory.search` (hybrid/keyword now; semantic later)
- `memory.get_context` (relationship-aware expansion with budgets)
- `memory.get_element` / `memory.get_neighborhood` (deterministic reads by id/version)
- `memory.propose` (create proposal elements + relationships)

---

## 8) Security, permissions, and guardrails

### 8.1 Scoping and isolation

All memory operations are scoped to `{tenant_id, organization_id, repository_id}`.

### 8.2 RBAC and tool grants

Define roles by capability, not by prompt:

- **Curator**: can edit canonical elements, review proposals, and promote baselines.
- **Contributor** (human): can create drafts and submit for review.
- **Execution agent (coding)**: can read_context and write proposals; cannot promote.
- **Execution agent (SRE/GitOps)**: may propose changes to runbooks/processes; cannot promote.

### 8.3 Query-time authorization

Retrieval projections MUST enforce authorization at query time:

- respect repository membership
- respect element-level visibility (if supported)
- prevent cross-tenant leakage

### 8.4 Permission-trimmed retrieval (non-negotiable)

Access control must be enforced **before** results reach an agent/model:

- Retrieval must be permission-trimmed using metadata filters / ACL-aware indexes at the same granularity as retrieval (often chunk-level, not document-level).
- Avoid “global embedding index + post-filtering” patterns that risk leakage or relevance pollution.
- Retrieval requests must carry principal context (tenant/org/roles and purpose), and the projection must log the policy decision.

---

## 9) Quality and evaluation

### 9.1 Retrieval quality

Create a regression harness with:

- a suite of **golden queries** per domain (process, architecture, backlog)
- expected “must include” context items (element/version ids)
- metrics:
  - context precision/recall and ranking quality (MRR/nDCG)
  - citation correctness (every cited item exists and supports the quoted claim)
  - version drift detection (prefers published/valid content; flags outdated hits)
  - stability under graph growth (no catastrophic drift)

### 9.1.1 Observability (retrieval trace is operable infrastructure)

Every retrieval run must be traceable end-to-end:

- query → rewritten query/routing decision → candidate sets → filters applied → rerank results → graph expansion → final context → citations
- include baseline/read_context and principal/purpose in logs/metrics (no PII by default)

### 9.2 Governance quality

Measure:

- proposal acceptance rate by agent type
- average time-to-review and time-to-publish
- validator failure rates (profile conformance)
- conflict rate (multiple proposals targeting same versions)

---

## 10) Definition of Done (contract-level)

This backlog is “done” when the following are true:

1. The platform clearly defines: **memory = element repositories + baselines**, and retrieval is projection-owned.
2. A **proposal-based curation queue** exists as first-class elements with:
   - lifecycle states
   - required metadata
   - required relationships (`TARGETS_VERSION`, `ACCEPTED_AS`, etc.)
3. A documented **read_context contract** exists with:
   - baseline selection rules
   - graph expansion controls
   - citations pinned to element versions
4. Execution agents are technically prevented (RBAC/tooling) from direct canonical edits.
5. A retrieval regression harness exists and runs in CI for projection changes.

---

## 11) Follow-on backlogs (if not implemented inside 656)

- **Curation Queue UX + projection**: full review UI, diff viewer, validator integration.
- **Chat-to-draft intake generalized**: extend 367 feedback intake pattern beyond feature requests.
- **Ontology conformance and mapping**: lock the ArchiMate profile constraints and cross-notation mapping rules (BPMN↔ArchiMate, doc↔element links).
- **Retrieval evaluation datasets**: golden queries + multi-turn conversation recall tests.
- **Index lifecycle**: blue/green index deployment, re-embedding jobs, drift monitoring, caching strategy.
