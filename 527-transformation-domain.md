# 527 Transformation — Domain Primitive Specification

**Status:** Draft (Enterprise Knowledge MVP implemented; versioned reference anchors implemented; governance/registry pending)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Domain primitives** — the canonical writers for:

- the element-centric **Enterprise Knowledge** repository substrate (Enterprise Repository), and
- governance/definition state that transforms and governs that repository (initiatives, baselines/promotions, Definition Registry, methodology overlays).

**IT4IT alignment (intent):** within the Transformation capability boundary, Domain primitives are the **systems of record** for IT value-stream state (design-time truth + approvals/promotions + definitions). Process primitives execute promoted ProcessDefinitions and emit process facts; process execution state does not live in the domain.

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Domain), Application Services (write + minimal strong reads), Application Events (domain facts), Data Objects (elements + definition payloads).
- **Out-of-scope layers:** Strategy/Business catalogs (see `backlog/527-transformation-capability.md`); full read/search surfaces (projection responsibility).

## 1) Domain responsibilities (canonical writers)

Scope identity (required everywhere): `{tenant_id, organization_id, repository_id}`. `repository_id` is the only repository identifier exposed in contracts and APIs (no separate `graph_id`).

### 1.1 `transformation-domain` (core: Enterprise Knowledge + Definition Registry + governance)

The `transformation-domain` primitive is the **single-writer** for core Transformation state (target: `ameide_core_proto.transformation.core.v1`), including:

- **Transformation initiatives** (portfolio/program objects).
- **Enterprise Repository workspace tree** (`WorkspaceNode` / `graph_node`) and element assignments.
- **Enterprise Knowledge substrate** (303): `Element` + `ElementRelationship` + `ElementVersion`, scoped by `{tenant_id, organization_id, repository_id}`.
- **Notation (metamodel) profiles** expressed as element kinds + namespaced `type_key` values and stored in element bodies/versions (no separate canonical tables per notation).
  - **Standards-compliant profiles** (e.g., ArchiMate 3.2, BPMN 2.0): constrain `type_key` to the relevant standard namespace (e.g., `archimate:*`, `bpmn:*`) and require validators/exporters to produce standard outputs.
  - **Extended profiles** (Ameide/vendor extensions): allow additional namespaces (e.g., `ameide:*`) and define custom constraints and export targets.
- **Promotions/baselines** (draft → submit → approve/reject → promote/rollback).
- **Definition Registry** (ProcessDefinition, ScaffoldingPlanDefinition, CompiledWorkflowDefinition, AgentDefinition, ExtensionDefinition, promotion state).

**Canonical writer rule:** external file formats are attachments/import/export bundles; they are not canonical state.

---

## 1.1a The simple storage posture (relational SoR; ontology-agnostic)

Normative “keep it simple” stance:

- The system-of-record is a **relational Postgres write model** for `elements`, `element_versions`, and `element_relationships` (plus governance tables like baselines/approvals).
- A “graph database” is not required for correctness: the graph is represented by `element_relationships`.
- Semantic search is not stored canonically: embeddings/indexes are **projection outputs** derived from `element_versions`.

Ontology support rule:

- Any ontology/notation (ArchiMate, BPMN, Scrum artifacts, TOGAF deliverables) is represented by:
  - `type_key` namespaces (e.g., `archimate:*`, `bpmn:*`, `ameide:*`), and
  - optional validators/exporters (profile-driven),
  not by introducing new canonical tables per methodology.

### 1.2 Methodology overlays (bounded views + validations + workflows)

Methodology is an **overlay** on the same canonical repository substrate:

- **Canonical design-time truth is always Elements** (`Element`, `ElementVersion`, `ElementRelationship`) plus governance/definitions/promotions.
- **Scrum/TOGAF/PMI differences live in:**
  - methodology-specific **UISurface** experiences (navigation, templates, forms),
  - methodology-specific **validators/checks** (what must exist + what must pass; optionally packaged as “validation packs” later),
  - methodology-specific **ProcessDefinitions** (different orchestration paths and gates),
  - and (optionally) methodology-specific **projections** (convenience read models).

In other words: at the “graph level”, everything is still just **elements and relationships**; methodology determines which elements are required, how they are validated, and how workflows gate promotions/releases.

Handshake anchor (recommended):

- Represent the tracked unit of change as an **Element** (a “change element”) with:
  - a minimal `methodology_key` in its versioned body (e.g., `scrum`, `togaf_adm`) or a link to a methodology element, and
  - a small set of **well-known REFERENCE relationships** that anchor governance/workflows:
    - `ref:requirement` → the authoritative requirement element **version** (via `target_version_id`)
    - `ref:deliverables_root` → the deliverables package/root element **version** (via `target_version_id`)
    - optional: `evidence:*`, `ref:baseline`, `ref:release`

This keeps the platform handshake simple: the “anchors” are just well-known `ElementRelationship` rows over the element substrate; UIs and workflows interpret them consistently.

### 1.2a Versioned reference relationships (lock-in; required for audit)

To make governance reproducible and auditable (no “latest version” ambiguity), reference relationships used as workflow/governance anchors MUST be versioned.

Normative rule:

- Any `ElementRelationship` with `relationship_kind = REFERENCE` and `type_key` in the `ref:*` family MUST include:
  - `metadata.target_version_id` = the immutable `ElementVersion.id` of the target element.

Notes:

- The relationship still targets `target_element_id` (element identity); `metadata.target_version_id` selects the authoritative snapshot.
- For non-anchor references (e.g., casual navigation links), `target_version_id` MAY be omitted.

Note: IT4IT is treated as the **default operating model** of the Transformation capability, not as an optional “methodology package”.

Optional overlays (future/optional):

- It is acceptable to introduce methodology-specific proto packages (e.g., a `scrum.v1` catalog) as **non-canonical convenience overlays** for UI/reporting, but they MUST NOT fork canonical design-time truth away from the element substrate.

## 2) Core aggregates (core domain)

Authoritative aggregate set (from `backlog/527-transformation-proto.md`), aligned to the element-centric repository substrate:

- Initiative
- WorkspaceNode
- Element
- ElementRelationship
- ElementVersion
- Baseline/Promotion
- Definition Registry entries (ProcessDefinition / ScaffoldingPlanDefinition / CompiledWorkflowDefinition / AgentDefinition / ExtensionDefinition)

**Profile note:** “ArchiMate model/view/element”, “BPMN definition”, and documents are all Elements with notation-specific kinds and namespaced `type_key` values. Standards-compliant profiles restrict `type_key` to standard namespaces (`archimate:*`, `bpmn:*`); extended profiles may also use `ameide:*`. View/layout content is stored as versioned payloads on the ElementVersion.

Idempotency/order rule: all domain facts carry a monotonic per-aggregate version; consumers are idempotent.

## 3) Enterprise Repository storage model (minimum)

The minimum relational model implied by “element-centric substrate + governance” (illustrative; exact schema defined by protos + migrations):

- Initiatives + workspace:
  - `transformation_initiatives`
  - `workspace_nodes` / `graph_nodes` (tree: parent_id, slug, order, attributes)
- Enterprise Knowledge substrate (303):
  - `elements`
  - `element_versions` (head/published pointers + immutable version rows)
  - `element_relationships` (semantic + containment + reference)
  - `element_assignments` (workspace node → element, ordering, optional fixed version reference)
- Governance:
  - `baselines` / `baseline_items` (baseline = set of `{element_id, version_id}` pointers)
  - `approvals` / `promotions` (policy-driven, auditable)
  - `attachments` / `evidence` (non-canonical bundles linked by reference relationships)
- Definition Registry:
  - `process_definitions`, `agent_definitions`, `extension_definitions`

**Non-negotiables:**

- Every table is tenant-scoped and uses stable IDs.
- Facts are emitted only after persistence; facts include monotonic aggregate versions.

### 3.1 Workspace taxonomy (locked; avoid duplication)

Repository organization (navigation tree) is represented by:

- `WorkspaceNode` (tree) and
- `ElementAssignment` (node → element, ordering, optional fixed version reference).

Do not introduce a second, competing folder system for repository organization (e.g., “folder elements”). If a notation needs internal structure (sections, groups, lanes-as-structure), represent it with `ElementRelationship` of kind `CONTAINMENT` inside the element graph.

### 3.2 Baselines + promotions (closed semantics; v1)

Baselines and promotions exist to make design-time truth **normative** and reproducible.

v1 stance:

- **Baseline** is an immutable snapshot reference set:
  - repository-scoped by `{tenant_id, organization_id, repository_id}`; initiatives may reference baselines (not the other way around)
  - required items: `{element_id, version_id}` for every included element
  - optional: `workspace_snapshot_ref` (attachment/hash) for reproducible “where it lived” browsing
- **Promotion** is a policy-driven transition that:
  - records approval decisions as evidence (process facts + evidence attachments), and
  - updates published pointers for included elements (`published_version_id`), without cascading versioning to referenced elements.

This preserves the “no cascading versioning” rule (views/docs do not force versions of referenced elements) while still allowing strict, reproducible promotion semantics.

### 3.3 Proposals and drafts (agents + humans; v1)

Draft work is represented as **draft baselines / draft version sets**, not as a parallel “proposal graph”:

- Agents propose changes by creating/updating element versions and submitting them into a draft baseline (or equivalent change set).
- Approval is a Process gate (`GateDecisionRecorded` as evidence).
- Apply is a Domain promotion (published pointers updated; domain facts emitted).

Branching/merge semantics (true long-lived branches) are intentionally out of scope for v1; add later only if required.

### 3.4 Evidence bundles (minimum model; v1)

Evidence is required to make promotions/audits defensible, but it is **not canonical state**.

Minimum conceptual model:

- `EvidenceBundle`:
  - scope refs: `{tenant_id, organization_id, repository_id}` and optionally `{initiative_id, baseline_id, promotion_id, process_instance_id}`
  - provenance: `produced_by` (tool/runner/actor), timestamps
  - `items[]`: logs, links, attestations, artifacts (attachments)

### 3.5 Work requests (execution substrate; v1)

To avoid treating “a queue message exists” or “a Job ran” as canonical truth, execution is modeled as a Domain-owned governance object:

- `WorkRequest` is a canonical Domain record representing **explicitly requested** tool/agent work initiated by a Process workflow (or an authorized actor).
- Runners/ephemeral Jobs are execution backends that consume `WorkRequested` facts and record outcomes back into Domain idempotently.
- Kubernetes Job TTL/cleanup is hygiene only; audit/evidence correctness comes from Domain-persisted `WorkRequest` status + linked `EvidenceBundle` + Process facts + Projection citations.

Minimum v1 conceptual shape (schema-backed; do not embed proto text here):

- Identity/scope: `{tenant_id, organization_id, repository_id}`, `work_request_id`
- Provenance:
  - `requested_by`: `{process_instance_id, activity_id}` (required for workflow-driven requests)
  - `requested_at`, `requested_by_actor` (optional: human/agent identity)
- Type:
  - `work_kind` ∈ `{tool_run, agent_work}`
  - `action_kind` (for tool runs): `preflight|scaffold|generate|verify|build|publish|deploy|smoke`
- Idempotency:
  - `client_request_id` / `idempotency_key` (required; stable across retries)
- Execution binding:
  - `queue_ref` (logical; environment binds to Kafka `{topic, consumer_group}` + scaling config)
  - `timeout_policy` / `retry_policy` (bounded)
- Inputs (subset as applicable):
  - repo checkout: `repo_url`, `commit_sha`, `workdir`
  - plan refs: `scaffolding_plan_ref` / definition refs
  - constraints: `allowed_paths`/`deny_paths`, tool allowlist, policy refs
- Status:
  - `status` ∈ `{requested, started, succeeded, failed, canceled}`
  - `started_at`, `finished_at`, `attempt`
- Outcomes:
  - `evidence_bundle_id` (optional until completion; required for success/failure)
  - `summary`, `result` (exit code/status) and/or `artifact_refs[]` (URIs + hashes)

Domain intents/facts (conceptual; v1):

- Intent: `RequestWorkRequested` (create WorkRequest; idempotent)
- Facts: `WorkRequested`, `WorkStarted`, `WorkCompleted`, `WorkFailed` (emitted after persistence)
- Intent: `RecordWorkOutcomeRequested` (attach evidence + terminal status; idempotent on `client_request_id`)

Kafka transport binding (normative):

- `WorkRequested` is emitted (outbox) onto a dedicated Kafka topic bound from `queue_ref` so the WorkRequest queue is not mixed with unrelated domain facts.
  - v1 queue topics (examples): `transformation.work.queue.toolrun.verify.v1`, `transformation.work.queue.toolrun.generate.v1`, `transformation.work.queue.agentwork.coder.v1`
- WorkRequest processors (KEDA-scheduled Jobs) are Kafka consumers and MUST use “ack only after durable outcome recorded” semantics:
  - disable auto-commit,
  - write outcome + evidence linkage into Domain idempotently,
  - only then commit the Kafka offset.

Correlation rules (required):

- Process must propagate `correlation_id` across: Process instance → RequestWork intent → WorkRequested fact → RecordWorkOutcome intent → WorkCompleted/Failed fact.
  - citations/trace links back to the underlying facts/versions that the evidence supports

Evidence bundles are stored as attachments/blobs and referenced from baselines/promotions/definitions using REFERENCE relationships; projection surfaces them in audit views.

### 3.5 Authorization + approval policy (minimum; v1)

Promotion/approval requires explicit policy. v1 treats policy as design-time truth:

- Access is scoped at `{tenant_id, organization_id, repository_id}` and (optionally) `{initiative_id}`.
- Approval policies are stored as schema-backed definitions (recommended as Definition Registry entries):
  - who can approve which gate kinds (baseline/definition/promotion)
  - required evidence types per gate (e.g., verify report, conformance report, security scan)
  - override/break-glass posture (explicitly logged as evidence)

UISurfaces and Agents must rely on these stored policies; they must not hard-code approval rules in clients.

## 4) Definition Registry (inside Transformation)

Transformation is the canonical store for design-time definitions consumed by control-plane operators and runtimes:

- ProcessDefinition (design-time workflow spec, versioned + promotable; referenced by Process CRs; fetched by Process operator/worker)
- ScaffoldingPlanDefinition (diff-friendly, deterministic build input derived from a promoted ProcessDefinition; used by scaffolders/operators/CLIs)
- CompiledWorkflowDefinition (compiled Workflow IR derived from a promoted ProcessDefinition; executed by Process primitives)
- AgentDefinition (fetched by Agent operator; governs runtime)
- ExtensionDefinition (drives scaffolding/promotion processes)

### 4.1 BPMN as the canonical workflow authoring source

For process/governance workflows, the canonical authoring payload is BPMN 2.0 (ProcessDefinition content). To satisfy portability + determinism:

- The BPMN payload is stored as a versioned, element-centric payload (`ElementVersion`) and referenced by the ProcessDefinition entry for promotion/execution.
- Promotion gates compile the promoted BPMN version into:
  - `CompiledWorkflowDefinition` (Workflow IR) for execution, and
  - `ScaffoldingPlanDefinition` for deterministic generation and verification.
- Promotion also materializes a schema-backed execution/binding profile (recommended as an `ExtensionDefinition`) so bindings are explicit, diff-friendly, and promotable independently of the raw BPMN XML.

See `backlog/527-transformation-process.md` §3 for the supported subset, binding rules, and verification gates.

### 4.1.1 ScaffoldingPlanDefinition (v1 shape; deterministic)

`ScaffoldingPlanDefinition` is the deterministic, diff-friendly build input derived from a promoted ProcessDefinition version. It exists so operators/runners can execute scaffolding without re-interpreting BPMN XML.

v1 requires four sections (conceptual; schema-backed definition; do not embed proto text here):

1) **Components** (from Collaboration participants)

- `component_ref` (required): `{kind, name, instance?}`
- ownership metadata (optional): owner/team

2) **Interaction contracts** (from message flows + execution profile bindings)

- `contract_kind`: `event | rpc`
- `proto_refs`: the exact proto symbols referenced by bindings (intents/facts); BPMN never invents schemas
- `producer_component_ref`, `consumer_component_ref`
- topic family references (for events) and service/method references (for rpc), when applicable

3) **Runtime bindings** (from execution profile normalization)

- For each executable BPMN node (by `activity_id`):
  - emitted step evidence requirements (process facts)
  - `send_domain_intent` / `await_domain_facts`
  - declared `allowed_reads`
  - timeout/retry policies (as applicable)

4) **Generation tasks** (what to scaffold/verify)

- `task_kind`: `proto_check | handler_stub | adapter_stub | worker_stub | projection_stub | ui_stub | ci_job | gitops_stub | verify_gate`
- `target_component_ref`
- `inputs` (refs to protos/definitions/templates)
- `expected_outputs` (file paths or output refs)
- `idempotency` posture (safe to re-run; never overwrite implementation-owned code)

GitOps posture (v1 target):

- `gitops_stub` generates a **capability-/primitive-local** GitOps folder (repo-owned shape), while environment-level wiring/aggregation remains a CI concern.
- Verification gates check for the presence and minimal correctness of the local GitOps folder, not for full environment wiring.

Determinism invariants:

- stable ordering and stable IDs for generated plan items (diff-friendly)
- every plan item is traceable to `{process_definition_id, process_definition_version}` and scoped by `{tenant_id, organization_id, repository_id}`

### 4.2 Binding metadata (standards-compliant vs extended)

The domain owns the source-of-truth for binding metadata resolution (and promotion gates enforce correctness):

- Standards-compliant BPMN profile:
  - binding metadata must be carried in `bpmn:documentation` (structured) or a linked Element (REFERENCE relationship),
  - no non-standard namespaces are required for correctness.
- Extended BPMN profile:
  - bindings may use extension namespaces (e.g., `ameide:*`) but portability is reduced and must be surfaced as part of conformance/export results.

This implies schema-backed domain models, not ad-hoc configs:

- CapabilityDefinition (identity/scope/nouns/topology)
- CapabilityEDAContract (topic families, envelopes, intent/fact catalogs)
- CapabilityPrimitiveDecomposition (primitives + responsibilities)

### 4.3 Notation profiles as definition sources (derivation, not duplication)

Notation profiles are treated as representations over the element-centric substrate, but promoted artifacts (models/views/documents) can act as **derivation sources** for schema-backed definitions:

- A normative model/view/document (promoted under a selected profile) may be used to derive/update:
  - `CapabilityDefinition`
  - `CapabilityEDAContract`
  - `CapabilityPrimitiveDecomposition`
  - and any derived generation plans needed to drive scaffolding/verification deterministically.

Examples:

- ArchiMate model/view → capability topology + contract skeletons + decomposition proposals.
- BPMN ProcessDefinition → compiled workflow IR + scaffolding plan for Process primitives (see `backlog/527-transformation-process.md`).
- DMN decision model → decision definition + validation harness plan.
- Documents (Markdown) → definition specs and required evidence checklists.

This preserves “elements-only canonical storage” while making design-time artifacts actionable and operator-friendly.

### 4.4 Notation profile registry + conformance outputs (missing concept; v1 shape)

Standards-grade modeling requires profile governance to be data-driven and versioned.

v1 introduces a profile registry as schema-backed definitions (recommended storage: Definition Registry as typed `ExtensionDefinition` entries):

- `NotationProfileDefinition` (versioned/promotable):
  - mode: `standards_compliant | extended`
  - allowed `type_key` namespaces (per element kinds and per relationship kinds)
  - relationship validity matrix (allowed source/rel/target combinations)
  - validator refs (what checks run, with versions/config)
  - exporter refs (what exports are produced, with versions/config)
- `ValidatorDefinition` / `ExporterDefinition`:
  - stable id + version
  - declared inputs (element versions, baseline refs) and outputs (artifacts/evidence)

Conformance/export outcomes are captured as evidence:

- validators/exporters run as promotion gates (Process primitive), and their results are recorded via `ToolRunRecorded` + evidence bundles
- projections materialize those results as `MetamodelConformanceProjection` and export catalogs, with citations to underlying facts/versions

## 5) Domain APIs (write-first)

- **Write surfaces:** prefer business verbs (no CRUD-as-API); accept via RPC and/or domain intents.
- **Reads:** minimal strong reads only (get-by-id / bounded lists); all browse/search/history/diffs go through projections and projection query services.

## 6) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind domain --name transformation --include-gitops
```

### Outbox

All domain facts emitted via transactional outbox; only domains emit facts.

### Idempotency table

Domain WriteService enforces durable idempotency via `client_request_id` (see `backlog/536-mcp-write-optimizations.md` §7 for full two-layer strategy):

```sql
CREATE TABLE idempotency_keys (
    tenant_id         UUID NOT NULL,
    client_request_id TEXT NOT NULL,
    aggregate_type    TEXT NOT NULL,
    aggregate_id      UUID,
    response_payload  JSONB NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (tenant_id, client_request_id, aggregate_type)
);
```

**Critical constraint:** Idempotency key insertion MUST be in the same transaction as the domain write. This ensures exactly-once semantics even under adapter restarts or horizontal scaling.

## 6.1) Implementation progress (repo snapshot; checklists)

Delivered (Enterprise Knowledge MVP slice):

- [x] Domain runtime exists at `primitives/domain/transformation` (gRPC server, migrations, tests).
- [x] Canonical enterprise repository substrate is implemented (elements-only):
  - [x] Repositories CRUD.
  - [x] Workspace nodes CRUD (tree semantics + ordering).
  - [x] Element assignments CRUD.
  - [x] Elements CRUD.
  - [x] Relationships CRUD.
  - [x] Element version snapshots on create/edit for versioned payloads (views/docs/layout).
- [x] Idempotency is implemented at the write boundary:
  - [x] `domain_inbox(tenant_id,message_id)` dedupes command replays.
- [x] Facts-after-persistence is implemented:
  - [x] Transactional `domain_outbox` rows appended in the same DB transaction as domain writes.
  - [x] Topic family emitted: `transformation.knowledge.domain.facts.v1`.
- [x] Gate: `bin/ameide primitive verify --kind domain --name transformation --mode repo` passes.

Not yet delivered (full domain meaning):

- [ ] Governance aggregates: initiatives, baselines/promotions, approvals, evidence bundles.
- [ ] Definition Registry aggregates: ProcessDefinition / AgentDefinition / ExtensionDefinition + versions/promotions.
- [ ] Broker publishing (dispatcher wiring): `internal/dispatcher` is log-only today; production publisher wiring is pending.

## 6.2) Clarification requests (next steps)

Confirm/decide:

- The canonical “context curation” taxonomy (what is an Element vs Attachment vs EvidenceBundle vs Definition) and the minimum metadata required for reproducibility.
- Baseline/promotion policy: default approval requirements/roles/evidence requirements per gate kind.
- Draft lifecycle extensions: whether true branching/merge semantics are required beyond draft baselines/change sets.
- Durable idempotency: expected retention/TTL for `client_request_id`, and which commands are required to be idempotent vs best-effort.

## 7) Acceptance criteria

1. Core Transformation state and definitions have a single-writer domain boundary.
2. Enterprise Knowledge is element-centric (303); import/export is attachment-only (external file formats are never canonical truth).
3. All state changes emit domain facts with per-aggregate monotonic versions.
4. Domains do not provide search/portal read APIs; projections do.
5. Durable idempotency via `client_request_id` at WriteService boundary (two-layer strategy).
