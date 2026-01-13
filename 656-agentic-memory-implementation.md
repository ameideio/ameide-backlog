# 656 — Agentic Memory: Implementation Proposal (End-to-End)

**Parent (contract):** `backlog/656-agentic-memory.md`  
**Related (canonical substrate):** `backlog/300-400/303-elements.md`  
**Related (retrieval):** `backlog/527-transformation-projection.md`, `backlog/535-mcp-read-optimizations.md`  
**Related (clean target refactor):** `backlog/657-transformation-domain-clean-target.md`  
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
   - the clean target (657) requires Broker/Trigger delivery (496 v2) as the normative runtime posture (no direct Kafka topic coupling as a contract).
6. **Backlog-as-memory ingestion** (git markdown → elements) is not implemented.
7. **Chat-to-draft ingestion** exists as a pattern (367-0) but is not generalized to “memory capture”.

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

- Projections must ingest domain/process facts via Kafka consumer groups as the default posture (496 v3).
- Any outbox-tailing relay must be treated as local/debug/recovery only (657), not as a long-running dependency in environments.

### 3.4 Align `read_context` with 527 (avoid parallel contracts)

Adopt the 527 selector model for any memory retrieval surface:

- `head | published | baseline_ref | version_ref`

Even if we implement memory-only first, we must not introduce a second incompatible `read_context` vocabulary.

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
- citations always include `{element_id, version_id}`

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

## 5) Workstreams (what to build)

### 5.1 Data model + schemas

1. **Repository seeding**
   - ensure a single org memory repository exists (bootstrap script / operator task)

2. **Curation object conventions**
   - formalize the `type_key` and required metadata keys from 656
   - formalize relationship keys and their directionality (no “implementation choice” ambiguity)

3. **Semantic retrieval storage (projection)**
   - implement pgvector-backed tables and indexes in the projection DB (per 535)
   - store per-chunk provenance: `{tenant_id, org_id, repository_id, element_id, version_id}`

### 5.2 Service/API surfaces

1. **Auth interceptors for gRPC**
   - add unary/stream interceptors to domain + projection servers
   - validate JWT (or mTLS identity) and map to `{tenant_id, org_id, role_codes}`
   - enforce:
     - only privileged roles can perform “approval/promotion” actions (baseline submit/approve/promote)
     - execution agents can only create/update proposals and draft/curated (non-approved) content
     - execution agents cannot mutate `published_version_id` pointers (directly or indirectly)

2. **Retrieval RPC(s)**
   - add `GetReadContext` / `GetContext` / `SemanticSearch` (naming per 535) to the projection query service(s)
   - return `read_context` + `citations[]`

3. **Curation workflow RPC(s)**
   - optional (MVP can use generic element CRUD)
   - recommended to add:
     - `ListProposals(filters...)`
     - `AcceptProposal(proposal_id, ...)`
     - `RejectProposal(proposal_id, reason)`

### 5.3 Projection build-out

1. **Curation queue projection**
   - materialize “proposal list” read model with filters (status/target/age/layer)
   - materialize “proposal diff inputs” (target pinned version + proposed payload refs)

2. **Semantic indexing pipeline**
   - consume domain facts (element created/updated, relationship created/updated)
   - produce contextual chunks:
     - document text
     - plus bounded relationship context (title/type/neighbor names)
   - embed asynchronously and upsert into `semantic_chunks`

3. **Governance-aware retrieval**
   - default retrieval reads against “published”
   - support baseline_ref selection for deterministic runs

### 5.4 UI surfaces

1. **Curation queue UI**
   - list proposals, view details, show target version, show proposed payload
   - actions: accept/reject/withdraw

2. **Baseline UX (memory publishing)**
   - extend existing governance page to:
     - create baseline
     - add items
     - submit/approve/promote

### 5.5 CLI + agent integrations

1. **CLI “no-brainer” entry points**
   - add `ameide memory context` (retrieval → citations)
   - add `ameide memory propose` (create proposal element + relationships)
   - add `ameide memory sync-backlogs` (mirror job)

2. **Inference tools**
   - add tool(s) calling semantic/context retrieval RPCs (replacing ad-hoc list+filter loops)
   - ensure tool grants match the “propose-only” write model

3. **MCP exposure (primary)**
   - implement an MCP server (or MCP adapter) as the standard agent interface to memory:
     - `memory.search`, `memory.get_context`, `memory.get_element`, `memory.propose`
   - back the tools by projection reads + domain command writes (no direct DB access).

### 5.6 Testing and verification

1. **Contract tests (service-level)**
   - proposals must include `targets_version` with `target_version_id`
   - accept must create a new version and link `accepted_as` pinned to that version
   - unauthorized principals cannot mutate canonical memory

2. **Retrieval regression harness (535 + 656)**
   - golden queries + expected version-pinned results
   - drift detection when graph grows

3. **E2E UI tests**
   - create proposal → accept → promote baseline → verify published pointer changes

---

## 6) MVP delivery increments (split)

This implementation plan is intentionally split so each increment is a concrete, end-to-end deliverable (not “implement part of the stack”).

- Increment 1: `backlog/656-agentic-memory-implementation-mvp/1-minimal-end-to-end.md`
- Increment 2: `backlog/656-agentic-memory-implementation-mvp/2-operational-end-to-end.md`
- Increment 3: `backlog/656-agentic-memory-implementation-mvp/3-scaled-end-to-end.md`

---

## 7) Open questions (explicit)

1. Do we want memory baselines to be “latest PROMOTED baseline” or do we introduce an explicit “active published baseline pointer” per repository?
2. Do we model curated memory as “documents/models” only, or do we add first-class `ameide:memory.claim` / `ameide:memory.decision` elements for higher retrieval precision?
3. Which embedding provider/model is v1 (and how do we cost-control multi-tenant indexing)?
