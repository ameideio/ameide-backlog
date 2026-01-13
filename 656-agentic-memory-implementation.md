# 656 — Agentic Memory: Implementation Proposal (End-to-End)

**Parent (contract):** `backlog/656-agentic-memory.md`  
**Related (canonical substrate):** `backlog/300-400/303-elements.md`  
**Related (retrieval + MCP):** `backlog/527-transformation-projection.md`, `backlog/535-mcp-read-optimizations.md`, `backlog/534-mcp-protocol-adapter.md`  
**Related (proto + EDA):** `backlog/527-transformation-proto.md`, `backlog/496-eda-principles-v2.md`, `backlog/509-proto-naming-conventions-v2.md`, `backlog/520-primitives-stack-v2.md`  
**Related (clean target refactor):** `backlog/657-transformation-domain-clean-target.md`  
**Related (Transformation execution posture):** `backlog/527-transformation-capability-v2.md`  
**Status:** Proposal (implementation plan)  
**Priority:** High  

---

## 0) Purpose

Turn the **contract** in `backlog/656-agentic-memory.md` into a concrete, buildable system:

- organizational memory is stored as **Elements + Versions + Relationships**
- humans curate; execution agents **propose**; curators **promote** into baselines
- retrieval is **projection-owned** and returns **read_context + citations**
- the system is safe (RBAC + tool grants), auditable, and testable

This document is intentionally implementation-oriented: services, schemas, APIs, UI surfaces, and verification.

## 0.1 What “implementation” means here (repo-aligned)

This is not a standalone “memory service”.

The target implementation is a **composition of primitives** (Application layer):

- **Domain primitive** (`primitives/domain/transformation`): canonical writes (elements/versions/relationships + governance + proposals) and outbox facts.
- **Projection primitive** (`primitives/projection/transformation`): *all* reads (browse/search/history/diff) plus retrieval pipeline (`read_context + citations`).
- **Integration primitive** (MCP adapter): agent-facing tools calling Domain/Projection RPCs (no direct DB access).
- **Process primitive** (optional plateau): BPMN “review/publish” orchestration posture (Zeebe), where primitives implement workers and Domains persist truth.

The docs in this 656 suite must therefore name:

- which primitive owns each loop step
- which proto service(s) are invoked
- which facts are emitted/consumed (EDA), and what evidence is required for audit

---

## 1) Inputs and constraints (what we must align to)

### 1.1 Contract requirements (656)

From `backlog/656-agentic-memory.md`:

- **Curation queue as first-class objects**: proposal element types, lifecycle states, required relationship keys, deterministic targeting via `target_version_id`.
- **Read path contract**: relationship-aware retrieval + hybrid search, baseline scoping, citations, and a `read_context` selector.
- **Role separation**: execution agents do not mutate canonical published truth; they propose.
- **Backlogs are ingestion only**: curated memory is expressed in an ArchiMate-first ontology graph; backlog markdown is a source for the curator.
- **MCP-first consumption**: agents consume memory via MCP tools, not filesystem search.

### 1.2 Canonical substrate invariants (303)

From `backlog/300-400/303-elements.md`:

- canonical state is Elements/Versions/Relationships (attachments/evidence are non-canonical)
- reference pinning via `REFERENCE.metadata.target_version_id`
- baselines/promotion define “normative truth”
- workspace tree is separate from element structure (no “folder elements” as navigation)

### 1.3 Retrieval posture (527/535)

From `backlog/527-transformation-projection.md` and `backlog/535-mcp-read-optimizations.md`:

- browse/search/history/diff must be **projection-backed**
- semantic retrieval is a **projection capability** (pgvector + hybrid retrieval)
- auth must be enforced at query time; results must be citeable and reproducible

### 1.4 Proto + EDA posture (496/509/527/520)

This backlog must not invent a parallel “memory protocol”.

**Proto-first contracts (generated SDKs, not ad-hoc JSON):**

- Domain and Projection gRPC services are defined under `packages/ameide_core_proto/src/ameide_core_proto/**`.
- Implementations must follow the `520` stack discipline: `buf generate` is canonical; generated roots are clobber-safe; drift is caught by regen-diff CI.

**EDA-correct between primitives:**

- Between primitives, messages are carried in a CloudEvents envelope (see the active `backlog/496-eda-principles-*` spec).
- Producers use transactional outbox; consumers use inbox dedupe (CloudEvents `id`).
- Protobuf payloads do not embed message metadata for inter-primitive traffic; metadata lives in CloudEvents.

**Repo reality today (important for 656):**

- Transformation facts already declare a stable semantic identity and delivery stream ref in proto options:
  - `stable_type` (semantic identity)
  - `stream_ref` (logical delivery stream)
  - Example: `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_facts.proto`

**Ideal mapping (make it explicit for implementers):**

- `stable_type` SHOULD map to CloudEvents `type` (and be the contract surface used for routing/filters).
- `stream_ref` binds to the delivery plane (Broker/Trigger or Kafka topics) via runtime config/CRDs (never hardcoded into business logic).

---

## 2) Current implementation reality (what already exists)

### 2.1 Element repositories exist today (domain + projection)

Delivered building blocks:

- **Domain write boundary**: `primitives/domain/transformation`
  - Implements `TransformationKnowledgeCommandService` (repositories, nodes, elements, relationships).
  - Implements `TransformationGovernanceCommandService` (baselines, approval decisions, promotion).
  - Implements version-pinned reference enforcement (`REFERENCE.metadata.target_version_id`) in `primitives/domain/transformation/internal/handlers/reference_versions.go`.
  - Stores canonical state in schema tables from `primitives/domain/transformation/migrations/V2__enterprise_repository.sql` and baselines from `primitives/domain/transformation/migrations/V4__governance_baselines.sql`.

- **Projection read boundary**: `primitives/projection/transformation`
  - Implements `TransformationKnowledgeQueryService` read models (elements, relationships, tree).
  - Implements `TransformationGovernanceQueryService` read models (baselines + items) with projection migration `primitives/projection/transformation/internal/adapters/postgres/migrations/V4__transformation_governance_projection.sql`.

### 2.2 UI can already browse repositories/elements

- `services/www_ameide_platform/app/api/repositories/**` provides org-scoped repository browsing and element listing by calling `client.transformationKnowledgeQuery.*`.
- `services/www_ameide_platform/features/elements/**` provides repository browsing UX.
- `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/governance/page.tsx` lists baselines (read-only today).

### 2.3 Agents can already query elements (non-semantic)

- `services/inference/src/inference_service/tools/repository/**` includes tools for list/search/neighborhood via `transformationKnowledgeQuery` (keyword filtering + graph neighborhood).

### 2.4 Major gaps vs the 656 contract

1. **No semantic/hybrid retrieval implementation** (535 explicitly says none exists).
2. **No standardized `read_context + citations`** returned by query surfaces (527 lists this as not yet delivered).
3. **No curation queue UX/workflow** (proposal types and review flows are not implemented as an end-to-end experience).
4. **RBAC enforcement is not implemented at the gRPC boundaries**:
   - domain/projection gRPC servers are created without auth interceptors (`grpc.NewServer()` in `primitives/**/cmd/main.go`);
   - enforcement currently mostly happens in web routes (Next.js) and in inference tool allowlists.
5. **Projection ingestion posture is not “clean target” yet**:
   - bridge-mode outbox tailing exists;
   - the clean target (657) requires EDA-correct delivery as the normative runtime posture (see `backlog/657-transformation-domain-clean-target.md` and the active `backlog/496-eda-principles-*` spec), not “DB tailing forever”.
6. **Backlog-as-memory ingestion** (git markdown → elements) is not implemented.
7. **Chat-to-draft ingestion** exists as a pattern (367-0) but is not generalized to “memory capture”.

### 2.5 Existing proto surfaces we build on (do not reinvent)

656 should be implemented by extending and composing existing Transformation services, not by inventing a parallel API surface.

**Domain command surfaces (canonical writes):**

- `io.ameide.transformation.knowledge.v1.TransformationKnowledgeCommandService`
  - Proto: `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_command_service.proto`
- `io.ameide.transformation.governance.v1.TransformationGovernanceCommandService`
  - Proto: `packages/ameide_core_proto/src/ameide_core_proto/transformation/governance/v1/transformation_governance_command_service.proto`

**Projection query surfaces (all reads):**

- `io.ameide.transformation.knowledge.v1.TransformationKnowledgeQueryService`
  - Proto: `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_query_service.proto`
- `io.ameide.transformation.governance.v1.TransformationGovernanceQueryService`
  - Proto: `packages/ameide_core_proto/src/ameide_core_proto/transformation/governance/v1/transformation_governance_query_service.proto`

**Fact identity + streams (EDA):**

- Knowledge facts: `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_facts.proto`
  - Example semantic identities: `io.ameide.transformation.knowledge.fact.ElementCreated.v1`, `...ElementVersionCreated.v1`
  - Example stream binding: `transformation.knowledge.domain.facts.v1`
- Governance facts: `packages/ameide_core_proto/src/ameide_core_proto/transformation/governance/v1/transformation_governance_facts.proto`
  - Example stream binding: `transformation.governance.domain.facts.v1`

**Implication (ideal target):**

- 656 work should add retrieval/context assembly + curation “front doors”, but the underlying persistence remains elements + baselines.
- Agents should not need to know “how to assemble a proposal element + relationship pins” to propose safely; prefer dedicated RPCs and MCP tools that encode those invariants.

---

## 3) Implementation decisions to lock (so work converges)

### 3.1 Single repository; status/lifecycle expresses “queue vs truth”

Do **not** split memory into multiple repositories.

Implement “what stage is this knowledge in?” via:

- `Element.lifecycle_state` (e.g., `DRAFT`, `IN_REVIEW`, `CURATED`, `APPROVED`, `RETIRED`) and/or
- curation metadata on proposal/decision elements (as defined in 656), and
- baselines/promotion for “normative truth” (published pointers and promoted baselines).

### 3.2 Make the curation queue contract queryable with existing primitives

Implement proposals/reviews/decisions as Elements and Relationships (per 656) so:

- the existing CommandService can create them
- the existing QueryService can list them (via `type_keys`, `search_query`, and targeted neighborhood queries)
- later, dedicated query RPCs can be added without changing storage

### 3.3 ArchiMate-first ontology is the organizing spine

Treat ArchiMate as the primary “meaning graph” when possible:

- Curated memory SHOULD be captured as `archimate:*` elements and semantic relationships, so traversal is ontology-driven and layer-aware by design.
- Documents/BPMN/diagrams remain first-class elements, but should link into ArchiMate elements so they are not the organization substrate.

### 3.4 Kafka-first ingestion is the default runtime posture (657 + 496 v3)

Do not “accidentally depend” on DB tailing for correctness.

- Projections must ingest domain/process facts via the platform’s EDA delivery plane as the default posture:
  - if running the Kafka-first posture (496 v3), use Kafka consumer groups
  - if running the Broker/Trigger posture (496 v2), use Broker/Trigger delivery filtered on CloudEvents `type`
- Any outbox-tailing relay must be treated as local/debug/recovery only (657), not as a long-running dependency in environments.

### 3.5 Align `read_context` with 527 (avoid parallel contracts)

Adopt the 527 selector model for any memory retrieval surface:

- `head | published | baseline_ref | version_ref`

Even if we implement memory-only first, we must not introduce a second incompatible `read_context` vocabulary.

### 3.6 Define “memory front doors” (proto + MCP) as a dev-spec surface

This is the missing “connected to the repo” layer: implementers need a stable, explicit interface that matches how Ameide builds everything else (proto → generated SDKs → primitives).

**Rule:** do not require clients (agents, CLI, UI) to construct low-level “element CRUD + relationship pinning” payloads to perform memory operations.

Instead, define a narrow **memory façade** with:

- **Projection reads**: `GetContext`, `Search`, `GetReadContext` returning `read_context + citations`.
- **Domain writes**: `ProposeChange`, `RebaseProposal`, `AcceptProposal`, `RejectProposal`, `PromotePublishedBaselinePointer`.

**Where it lives (ideal target):**

- Proto package: `io.ameide.transformation.memory.v1` (or `...curation.v1` + `...retrieval.v1` if you prefer splitting read vs write).
- Implemented by Transformation Domain/Projection primitives (façade methods can be implemented as thin adapters around existing repositories).
- Exposed to agents primarily via MCP tools that map 1:1 to these RPCs (see `backlog/534-mcp-protocol-adapter.md`).

**Why this is worth the refactor:**

- reduces drift across agent/CLI/UI implementations
- makes authorization and audit inevitable (gRPC interceptors + consistent evidence)
- keeps “memory” from becoming an accidental second protocol surface outside the proto stack

---

## 4) End-to-end flows to implement

### 4.1 Flow A — Agent/human retrieves memory context (read path)

Goal: an agent can ask “what is the current truth?” and get citeable context.

Implementation:

- Add a projection-backed retrieval RPC (or MCP tool) that:
  1. selects a `read_context` (default: published/published-baseline)
  2. performs candidate recall (keyword now; hybrid later)
  3. expands via relationships (bounded depth/budget)
  4. returns `context_items[]` + `read_context` + `citations[]` (version-pinned)

MVP acceptance:
- works for documents/backlogs + basic graph expansion
- citations always include `{repository_id, element_id, version_id}`

### 4.2 Flow B — Execution agent proposes a memory change (write path)

Goal: execution agents never mutate canonical memory directly.

Implementation:
- In the single memory repository, create:
  - proposal element `ameide:curation.proposal` with required metadata (and `lifecycle_state=DRAFT`/`IN_REVIEW`)
  - `ameide:curation.targets_version` relationship (REFERENCE) that pins `target_version_id`
  - payload attachment/sub-element for the proposed new body (initially “full replacement”)

MVP acceptance:
- can propose updates to a backlog document element (replace body)
- proposal links deterministically to the exact target version

### 4.3 Flow C — Curator reviews, accepts, and promotes

Goal: human curator can approve/reject proposals and publish changes.

Implementation:

1. Review UI lists proposals by status, target repository, and age.
2. “Accept” action:
   - writes a new `ElementVersion` in the memory repository (or updates element head + creates version)
   - links proposal → new version via `ameide:curation.accepted_as` (version-pinned)
   - advances lifecycle/state to “curated” (approved later by promotion)
3. Optional baseline workflow:
   - create a baseline, add updated versions as baseline items, submit/approve/promote
   - promotion updates `published_version_id` pointers (already implemented in domain)

MVP acceptance:
- accept/reject works; accepted changes appear in the repository and are reachable via browse
- a baseline can be promoted and updates `published_version_id` pointers

### 4.4 Flow D — Backlog markdown becomes memory documents

Goal: backlog markdown is ingested as raw sources, then curators articulate the durable memory in an ArchiMate-first graph.

Implementation:

- Implement a “backlog mirror” job/command that:
  - maps each `backlog/**/*.md` file to a deterministic element id in the memory repository as an **ingestion element**
  - writes the markdown as a versioned document element with `type_key=ameide:ingest.backlog_md`
  - records provenance metadata (source repo + commit sha + path)
  - records provenance metadata (source repo + commit sha)

MVP acceptance:
- sync can be run repeatedly (idempotent, produces new versions only when content changes)
- the synced docs are searchable/listable for curators, but they are not treated as the canonical organization mechanism
- curators can create/update ArchiMate elements/relationships that reference ingested backlog elements as evidence

### 4.5 Flow E — Chat assistant captures business/process drafts (ingestion)

Goal: chat sessions can create “draft memory” without bypassing curation.

Implementation:
- Create an “intake element” in the memory repository with provenance (marked ingestion/draft).
- Transform intake → draft element(s) + proposal linking targets (when updating existing truth).
- Route to the same curation queue.

MVP acceptance:
- chat-to-draft creates a proposal and links it to a target baseline/version when applicable

---

## 5) Development specification (ideal target; refactor-friendly)

This section is intentionally “close to build”: it names concrete primitives, proto seams, and the EDA flow that makes the loops real in this repo.

### 5.1 Domain primitive (Transformation) — canonical writes + governance

**Code location:** `primitives/domain/transformation`

**Responsibilities (must be true):**

- Persist canonical state: Elements + ElementVersions + ElementRelationships + workspace assignments + baselines.
- Enforce invariants on proposal/decision artifacts (656 type_keys + required relationship pinning).
- Enforce RBAC at the gRPC boundary (no “UI-only auth” posture).
- Emit facts via transactional outbox for every accepted write (496).

**Proto surfaces (existing):**

- `io.ameide.transformation.knowledge.v1.TransformationKnowledgeCommandService`
- `io.ameide.transformation.governance.v1.TransformationGovernanceCommandService`

**Refactor (recommended, ideal): introduce explicit memory “front door” RPCs**

Clients should not assemble low-level element CRUD payloads to do memory operations safely.

Define one of:

- `io.ameide.transformation.memory.v1.TransformationMemoryCommandService` (preferred), or
- add methods on existing command services (acceptable but mixes responsibilities).

Minimum operations (must exist by Increment 1/2):

- `ProposeChange` (creates `ameide:curation.proposal` + `targets_version` pin + payload)
- `RebaseProposal` (re-targets a proposal to the current selected version; links `rebased_to`)
- `AcceptProposal` / `RejectProposal` (writes new ElementVersion on accept; writes decision artifacts always)
- `PromotePublishedBaselinePointer` (or equivalent) so `selector=published` is deterministic

**Invariant checks (must be enforced server-side):**

- proposals MUST include `ameide:curation.targets_version` with `REFERENCE.metadata.target_version_id`
- accept MUST block if proposal is stale relative to the selected `read_context`
- accept MUST create a new `ElementVersion` and link `accepted_as` pinned to that version

### 5.2 Projection primitive (Transformation) — all reads + retrieval pipeline

**Code location:** `primitives/projection/transformation`

**Responsibilities (must be true):**

- Provide all browse/search/history/diff read surfaces.
- Provide the retrieval pipeline (keyword → hybrid → rerank → graph expansion → distillation), but always return citeable results.
- Enforce permission trimming at query time (principal/scope/roles/purpose), not after results are assembled.
- Materialize curation queue read models (list proposals, diffs, conflicts, evidence) so UI/agents don’t re-implement joins.

**Proto surfaces (existing):**

- `io.ameide.transformation.knowledge.v1.TransformationKnowledgeQueryService`
- `io.ameide.transformation.governance.v1.TransformationGovernanceQueryService`

**Refactor (recommended, ideal): introduce explicit memory retrieval RPCs**

Define one of:

- `io.ameide.transformation.memory.v1.TransformationMemoryQueryService` (preferred), or
- extend `TransformationKnowledgeQueryService` (acceptable but mixes responsibilities).

Minimum operations (Increment 1): `GetContext` (keyword-only recall is fine) that returns:

- `read_context` (effective selector and resolved baseline id)
- `items[]` and `citations[]` (each citation includes `{repository_id, element_id, version_id}`)
- `why_included` per item (minimal)

Increment 2+: add:

- “what changed?” baseline diff reads
- retrieval trace output (routing → filters → rerank → expansion → citations) for operability

Increment 3+: add:

- hybrid recall (BM25/tsvector + pgvector) with baseline+ACL filtering
- retrieval mode + budget parameters (`local|global|drift`, bounded)

### 5.3 Integration primitive — MCP as the agent interface (no bespoke protocols)

**Responsibilities:**

- Expose `memory.*` tools over MCP that map 1:1 to the memory front door RPCs.
- Enforce tool grants (“execution agents propose, curators publish”) in addition to backend RBAC.

**Contract anchor:** `backlog/534-mcp-protocol-adapter.md`

Minimum MCP tools (v1):

- `memory.get_context` → Projection `GetContext`
- `memory.propose` → Domain `ProposeChange`

### 5.4 UISurface — curator and editor UX

**Responsibilities:**

- Curator can list proposals, view pinned targets, see diffs, accept/reject/rebase, and promote published truth.
- UI must consume projection read models; it must not join domain tables directly.

### 5.5 Agent + CLI integration (reduce cognitive load)

**CLI (primary “front door” for agents running locally or in tasks):**

- `ameide memory context` calls `memory.get_context` and prints citations
- `ameide memory propose` creates a pinned proposal
- `ameide memory sync-backlogs` runs the ingestion mirror

**Agent runtime:**

- Agents must treat `read_context` as mandatory input/output, and must cite only what was returned.

### 5.6 EDA implementation notes (what flows where)

**What matters for 656 (regardless of delivery plane):**

- Domain emits facts after persist; projection ingests them idempotently.
- Facts must be sufficient to rebuild:
  - curation queue read models
  - baseline pointer state
  - retrieval indexes (chunks/embeddings) as derived artifacts

If you add explicit “proposal accepted/rejected” facts, they must still be derivable from canonical element/governance facts (no split-brain).

### 5.7 Testing (tie each increment to a runnable verification surface)

Use the scenario-driven plan in:

- `backlog/656-agentic-memory-implementation-mvp/README.md`

and ensure each increment produces:

- **Increment 1:** one end-to-end smoke (permission trimming + citations + proposal lifecycle).
- **Increment 2:** golden queries + deterministic diff/rebase assertions + trace logs.
- **Increment 3:** CI-gated evaluation + blue/green index canary comparing ranking + citation sets.

---

## 6) MVP delivery increments (split)

This implementation plan is intentionally split so each increment is a concrete, end-to-end deliverable (not “implement part of the stack”).

- Increment 1: `backlog/656-agentic-memory-implementation-mvp/1-minimal-end-to-end.md`
- Increment 2: `backlog/656-agentic-memory-implementation-mvp/2-operational-end-to-end.md`
- Increment 3: `backlog/656-agentic-memory-implementation-mvp/3-scaled-end-to-end.md`

---

## 7) Open questions (explicit)

1. Do we model curated memory as “documents/models” only, or do we add first-class `ameide:memory.claim` / `ameide:memory.decision` elements for higher retrieval precision?
2. Which embedding provider/model is v1 (and how do we cost-control multi-tenant indexing)?
3. Do we need a “staging baseline pointer” (in addition to published) to support policy-driven promotion plateaus?
