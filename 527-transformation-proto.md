# 527 Transformation — Proto Contract Specification

> **DEPRECATED (2026-01-12):** This document includes compile-to-Temporal / “Temporal-backed execution” posture language for BPMN-authored processes.  
> Current direction: BPMN-authored Process primitives execute on **Camunda 8 / Zeebe**; Temporal remains for coded platform workflows.  
> See `backlog/527-transformation-proto-v2.md`.

> **DEPRECATED (EDA v2 transport posture, 2026-01):** This document also assumes topic-family contracts and treats CloudEvents as a transport binding.  
> Current standard is `backlog/496-eda-principles-v6.md` (hybrid + `io.ameide.*`, Git-backed owners). Treat “topic families” as legacy guidance only; prefer CloudEvents `type` as the contract surface.

> **DEPRECATED (Definition Registry, 2026-01):** the “Definition Registry” as a separate canonical store is deprecated under the Git-backed Enterprise Repository posture.  
> ProcessDefinitions (BPMN) and other definitions live as versioned files in the tenant Enterprise Repository (e.g., `processes/**`). Any `ameide_core_proto.transformation.registry.v1` contracts should be treated as migration-era only until replaced by Git-backed definition references and governance/audit pointers.

**Status:** Draft (core envelope unification pending; legacy UI façade removed)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **proto contracts** for the Transformation capability following the historical docs `backlog/496-eda-principles-v2.md` and `backlog/509-proto-naming-conventions.md`.

> **No embedded proto text.** This is a **file index + invariants** specification. Full proto definitions live in the canonical file locations and are the source of truth.

---

## Layer header (Application contracts)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Service (RPC/query), Application Interface (topic families), Application Event (facts), Data Object (proto messages/envelopes).
- **Out-of-scope layers:** Strategy/Business definition (see `backlog/527-transformation-capability.md`) and Technology runtime selection.

## 0.1) Implementation progress (repo snapshot; checklists)

Delivered (today, in repo):

- [x] Enterprise Knowledge contracts exist and are used by the MVP:
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/` (commands, queries, facts, view layout payloads).
  - [x] Domain emits `transformation.knowledge.domain.facts.v1` and Projection ingests those facts.
- [x] WorkRequests contract supports suite-driven verification and structured outputs:
  - [x] WorkRequest `tool_run` may select a deterministic verification suite (e.g., `verification_suite_ref=transformation.verify.ui_harness.gateway_overlay.v1`) without introducing a new `action_kind=e2e`.
  - [x] WorkResult includes an `outputs` map so workflows can consume deterministic structured results (e.g., `outputs.commit_sha`, `outputs.pr_url`) without log scraping.
- [x] Governance/registry packages exist (not in MVP slice yet):
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/transformation/governance/v1/`
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/transformation/registry/v1/`
- [x] Transformation process facts contracts exist (not in MVP slice yet):
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/process/transformation/v1/`
- [x] Legacy UI façade removed (deprecated; replaced by split subcontext services):
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/` no longer exists.
- [x] Scrum profile contracts exist (not in MVP slice):
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/`
  - [x] Scrum process facts exist under `packages/ameide_core_proto/src/ameide_core_proto/process/scrum/v1/`

Not yet delivered (target state in this spec):

- [ ] Core Transformation envelope unification under `ameide_core_proto.transformation.core.v1` (subject/meta catalogs; currently partial).
- [ ] Optional methodology overlay contracts (TOGAF/PMI) if/when we decide they add value beyond element-centric artifacts + validations.

## 1) Packages (authoritative)

- Core Transformation common/meta (partial today; target for full unification): `ameide_core_proto.transformation.core.v1`
- Enterprise Knowledge substrate (canonical repository model, 303 target-state): `ameide_core_proto.transformation.knowledge.v1`
- Governance (exists; target for broadened coverage): `ameide_core_proto.transformation.governance.v1`
- Definition Registry (legacy/migration only; deprecated): `ameide_core_proto.transformation.registry.v1`
- Scrum profile (exists): `ameide_core_proto.transformation.scrum.v1`
- Legacy UI façade (removed): `ameide_core_proto.transformation.v1`
- Optional methodology overlays (not required for the platform handshake): `ameide_core_proto.transformation.togaf.v1`, `ameide_core_proto.transformation.pmi.v1`
- Process facts:
  - Scrum: `ameide_core_proto.process.scrum.v1` (exists)
  - Transformation (exists): `ameide_core_proto.process.transformation.v1`
  - Optional overlays (not required for the platform handshake): `ameide_core_proto.process.togaf.v1`, `ameide_core_proto.process.pmi.v1`

## 2) Topic families (runtime seams)

Target posture: split “Enterprise Knowledge” (element-centric repository substrate) from “Governance” and optional methodology overlays at the topic-family level.
Current implementations emit/consume `transformation.<subcontext>.domain.facts.v1` streams (e.g., `transformation.knowledge.*`, `transformation.registry.*`, `transformation.governance.*`). Avoid introducing a generic `transformation.domain.*` seam unless/until we actually unify envelopes under `ameide_core_proto.transformation.core.v1`.

| Topic family | Message type | Purpose |
|------------|--------------|---------|
| `transformation.process.facts.v1` | `TransformationProcessFact` | IT4IT-aligned value-stream workflow facts (Temporal); evidence of orchestration (distinct from domain facts) |
| `transformation.work.domain.facts.v1` | `TransformationWorkDomainFact` | Canonical WorkRequest lifecycle facts (`WorkRequested/Started/Completed/Failed`) for persistence/projection (audit trail); execution backends MUST NOT scale on fact streams |
| `transformation.work.queue.toolrun.verify.v1` | `WorkExecutionRequested` | **Execution queue intent** (point-to-point) for verify-class tool runs; used for KEDA lag-based scaling |
| `transformation.work.queue.toolrun.generate.v1` | `WorkExecutionRequested` | **Execution queue intent** (point-to-point) for scaffold/generate-class tool runs (`ActionKind=SCAFFOLD|GENERATE`); used for KEDA lag-based scaling |
| `transformation.work.queue.toolrun.verify.ui_harness.v1` | `WorkExecutionRequested` | **Execution queue intent** (point-to-point) for non-agentic UI harness verification (BuildKit + shadow workloads + Gateway API header overlays + Playwright); used for KEDA lag-based scaling |
| `transformation.work.queue.agentwork.coder.v1` | `WorkExecutionRequested` | **Execution queue intent** (point-to-point) for coder-class agent work; used for KEDA lag-based scaling |
| `transformation.knowledge.domain.facts.v1` | `TransformationKnowledgeDomainFact` | Enterprise Knowledge facts emitted after persistence |
| `transformation.registry.domain.facts.v1` | `TransformationRegistryDomainFact` | Definition Registry facts emitted after persistence |
| `transformation.governance.domain.facts.v1` | `TransformationGovernanceDomainFact` | Governance (initiatives/baselines/promotions/approvals) facts emitted after persistence |
| `scrum.domain.intents.v1` | `ScrumDomainIntent` | Optional overlay: Scrum-native state mutations (not required when Scrum is represented as elements + links + workflows/projections) |
| `scrum.domain.facts.v1` | `ScrumDomainFact` | Optional overlay: Scrum facts emitted after persistence |
| `scrum.process.facts.v1` | `ScrumProcessFact` | Optional overlay: Scrum workflow facts (Temporal) |
| `togaf.domain.intents.v1` | `TogafDomainIntent` | Optional overlay (not required): TOGAF-native profile state mutations |
| `togaf.domain.facts.v1` | `TogafDomainFact` | Optional overlay (not required): TOGAF facts emitted after persistence |
| `togaf.process.facts.v1` | `TogafProcessFact` | Optional overlay (not required): TOGAF governance workflow facts |
| `pmi.domain.intents.v1` | `PmiDomainIntent` | Optional overlay (not required): PMI-native profile state mutations |
| `pmi.domain.facts.v1` | `PmiDomainFact` | Optional overlay (not required): PMI facts emitted after persistence |
| `pmi.process.facts.v1` | `PmiProcessFact` | Optional overlay (not required): PMI governance workflow facts |

Agent handover note (event-driven first):

- Inter-agent/role delegation for delivery work (Product Owner→Solution Architect→Executor) is intentionally treated as a **separate handover seam** from the Transformation/Scrum domain/process seams above.
- Canonical semantics live in `backlog/505-agent-developer-v2.md` (work intents/facts + evidence refs). Any interactive transport (e.g., A2A) is a binding over that seam, not a competing contract.

Execution substrate note (WorkRequests; Activity-driven):

- Long-running tool runs and agent work are represented as Domain-owned `WorkRequest` records: Process requests work via Domain intents/commands; Domain emits `WorkRequested` facts after persistence on `transformation.work.domain.facts.v1` (audit trail), and emits `WorkExecutionRequested` execution intents after persistence onto dedicated execution queue topics (e.g., `transformation.work.queue.toolrun.verify.v1`, `transformation.work.queue.toolrun.generate.v1`, `transformation.work.queue.toolrun.verify.ui_harness.v1`, `transformation.work.queue.agentwork.coder.v1`). KEDA-scaled execution backends consume `WorkExecutionRequested` and record outcomes back into Domain idempotently.
- Process workflows progress on Activity completion results; Activities may wait for WorkRequest completion (poll/long-poll + heartbeat) and return `{work_request_id, outcome, evidence_refs}`. Work domain facts remain audit/projection inputs, not workflow step-completion signals.

## 3) Envelope invariants (per 496)

All envelopes (domain intents, domain facts, process facts) must include:

- Tenant isolation: `tenant_id` (required)
- Organization scope: `organization_id` (required everywhere as scope metadata)
- Repository scope: `repository_id` (required; no separate `graph_id`)
- Traceability: `message_id`, `correlation_id`, `causation_id` (when available), timestamps
- Producer identity and schema/version metadata
- Aggregate/process subject with stable identity and monotonic versioning where applicable

Execution invariants (additional; v1):

- Work execution messages MUST include a stable `work_request_id` and a stable `client_request_id` / `idempotency_key` so retries and at-least-once delivery converge to one canonical outcome.

## 3.0.1 Transport bindings: CloudEvents (mandatory in-cluster)

For inter-primitive (microservice-to-microservice) traffic inside Kubernetes, CloudEvents is the **mandatory** envelope and Protobuf is the payload, per `backlog/496-eda-principles-v2.md`.

Integration adapters MUST map external systems to the same CloudEvents contract so internal consumers see one consistent envelope.

Normative mapping (canonical → CloudEvents v1.0):

| Canonical field | CloudEvents attribute | Notes |
|---|---|---|
| `message_id` | `id` | Stable per message; may be derived from canonical idempotency key when applicable |
| producer identity | `source` | Must be a URI-reference; prefer a controlled URN scheme (e.g., `urn:ameide:domain:<name>` / `urn:ameide:process:<name>` / `urn:ameide:agent:<id>`) |
| canonical message kind | `type` | Prefer a stable identifier derived from `event_fact.type` or equivalent stable type key (not raw proto package/type names) |
| subject identity | `subject` | Aggregate/process subject identifier (when meaningful) |
| occurred timestamp | `time` | |
| payload bytes/object | `data` | If publishing protobuf bytes, use `datacontenttype=application/protobuf` |
| schema reference | `dataschema` | Required for in-cluster inter-primitive traffic; at boundaries it is strongly recommended |

Required Ameide invariants that do not fit core CloudEvents attributes MUST be carried as CloudEvents extension attributes (lowercase alphanumeric; no underscores). Minimum required set is defined in `backlog/496-eda-principles-v2.md` (tenant + correlation/causation + tracing). Transformation may also include additional scope extensions such as `orgid` and `repositoryid` when needed.

## 3.1 Process-facts catalog (step events; to define)

This catalog makes “each BPMN step produces an event” precise without violating domain boundaries.

Rule:

- **Every executable BPMN node transition emits a process fact** on `transformation.process.facts.v1`.
- Domain facts remain the integration-grade business events (after persistence). Process facts are orchestration evidence.

Recommended v1 event set (stable; generic; avoids proto explosion per workflow):

- `ProcessInstanceStarted`
- `ProcessInstanceCompleted`
- `ProcessInstanceFailed`
- `ActivityTransitioned` (single generic event for step transitions)
- `GateDecisionRecorded` (Approve/Reject/Override/Cancel as evidence)
- `ToolRunRecorded` (runner/CLI/buf/verify executions captured as evidence)
- `ReadPerformed` (projection read used for a decision; evidence of what was read and under which `read_context`)

Default emission invariant (v1):

- For **deployable/scaffoldable** workflows, the runner MUST emit `ActivityTransitioned` for every executable node:
  - `STARTED`
  - `COMPLETED`
  - `FAILED` (when applicable)
- Suppressing step events is not allowed for deployable/scaffoldable workflows. (If a workflow is “model-only”, suppression MAY be allowed, but then “each step produces an event” does not apply.)

Minimum required fields (conceptual; exact proto lives under `ameide_core_proto.transformation.core.v1` / `...process...` packages):

- Scope: `{tenant_id, organization_id, repository_id}` (required)
- Identity:
  - `process_definition_id` + `process_definition_version` (required)
  - `process_instance_id` (required)
  - `activity_id` (required for `ActivityTransitioned`; BPMN element id)
- Transition:
  - `transition` enum: `STARTED | COMPLETED | FAILED | SKIPPED` (as applicable)
  - `attempt`/`retry_count` where applicable
  - `occurred_at` timestamp
- Traceability:
  - `message_id`, `correlation_id`, `causation_id` (when available)
  - optional `domain_intent_ref` and `domain_fact_ref` links when an activity triggers/awaits domain changes
- Tool evidence (for `ToolRunRecorded`):
  - `tool` (e.g., `ameide` (CLI), `buf`, `go test`)
  - `arguments`/`invocation_ref`
  - `result` (exit code, summary)
  - evidence attachment refs (logs/artifacts)

- Read evidence (for `ReadPerformed`):
  - `query_ref` (stable identifier of the projection query used)
  - `read_context` (baseline/time/revision selector)
  - `result_ref` (hash/reference to the result payload, or citations sufficient for replay)
  - `decision_ref` (optional; link back to the activity/gateway that used the read)

Gateway clarity (v1):

- Gateways SHOULD be represented as `ActivityTransitioned` events as well:
  - `activity_id` = gateway BPMN element id
  - optional fields (if added later) may include `selected_flow_id` / `branch_label` / `decision_summary`
- This keeps “why did it go this way?” visible in the run timeline without introducing a new per-gateway proto catalog.

Correlation invariant:

- If an activity emits a domain intent, it MUST set/propagate correlation/causation so projections can join the activity transition evidence to the resulting domain facts deterministically.

## 3.2 Read context + citations (query invariants; v1)

The EDA envelope invariants above apply to bus messages (intents/facts). Projection query responses additionally MUST be reproducible:

- All query responses MUST return a `read_context` describing what state was read (head/published/baseline/version selectors).
- Audit- and approval-grade views MUST return `citations[]` that reference the underlying domain facts/process facts and/or element versions used to compute the answer.

Normative cross-reference:

- `backlog/527-transformation-projection.md` §4.2.1 defines the v1 `read_context` + citation expectations.

## 4) Core aggregate set (Enterprise Repository; authoritative writers)

Core Transformation treats the Enterprise Repository (Enterprise Knowledge) as a set of single-writer aggregates:

- Initiative
- WorkspaceNode
- Element
- ElementRelationship
- ElementVersion
- Baseline/Promotion
- Definition Registry entries (ProcessDefinition / ScaffoldingPlanDefinition / CompiledWorkflowDefinition / AgentDefinition / ExtensionDefinition)

### “Element-based, any ontology” and relational storage (normative clarification)

The canonical write model is relational:

- `Element` / `ElementVersion` / `ElementRelationship` are persisted in Postgres.
- Graph traversal and semantic search are *read projections* derived from facts:
  - graph projection for traversal/impact analysis,
  - vector projection (e.g., pgvector) keyed by `{element_id, version_id}` for semantic retrieval.

Methodology overlays do not require new canonical aggregates:

- Scrum/TOGAF/PMI artifacts are represented as elements and links under a chosen ontology namespace (`type_key`) and validated/exported by profile-aware tooling.
- The minimal “platform handshake” can be expressed as well-known `REFERENCE` relationships from the change element (e.g., `ref:requirement`, `ref:deliverables_root`) with version selection carried explicitly (e.g., `metadata.target_version_id`), rather than requiring a methodology-specific work-item model.

### Notation (metamodel) profiles: standards-compliant vs extended (invariants)

Transformation treats ArchiMate/BPMN/etc. as **profiles over the element model** (303). Storage remains element-centric; profiles define validity, validation, and exports.

- **Namespaced `type_key` (required):**
  - Standards namespaces: `archimate:*`, `bpmn:*` (and other standards as added)
  - Ameide extension namespace: `ameide:*`
- **Standards-compliant profile invariant:** validators/exporters MUST reject non-standard namespaces for that profile (e.g., an ArchiMate 3.2 compliant diagram cannot contain `ameide:*` types/relations).
- **Extended profile invariant:** may include `ameide:*` (or vendor namespaces), but downstream consumers must explicitly opt in to that profile.

**Profile note:** “ArchiMate model/view/element”, “BPMN definition”, and documents are represented as Elements with notation-specific kinds and namespaced `type_key` values. View/layout content is stored as versioned payload on ElementVersion.

**ProcessDefinition note (BPMN as source of truth):**

- ProcessDefinitions are **design-time definitions** stored and promoted by the Definition Registry (Domain).
- The canonical authoring payload is **BPMN 2.0** (portable; standards-compliant profiles avoid non-standard namespaces for correctness).
- Prefer value-stream-first ProcessDefinition IDs with methodology as a qualifier (examples: `r2r.governance.scrum.v1`, `r2r.governance.togaf_adm.v1`).
- Promoted ProcessDefinition versions are compiled into:
  - `CompiledWorkflowDefinition` (Workflow IR for Temporal-backed execution), and
  - `ScaffoldingPlanDefinition` (deterministic generation/verification plan).
- Process primitives execute promoted compiled Workflow IR and emit **process facts**; they never replace domain facts as the system of record.

**Design-time derivation note (general; any notation):**

- The Definition Registry contains both **authored** definitions (human-authored or imported) and **derived** definitions produced by promotion gates.
- Any notation profile (ArchiMate/C4/UML/DMN/docs/custom) may be used as a design-time source for deterministic derivations, with traceability to the source `{element_id, version_id}`.
- BPMN is the workflow-specific case: promoted ProcessDefinition versions deterministically derive `CompiledWorkflowDefinition` + `ScaffoldingPlanDefinition` with traceability to `{process_definition_id, version}`.

## 5) Core intent catalog (to define)

Enterprise Knowledge write ingress is the `TransformationKnowledgeCommandService` RPC surface (`ameide_core_proto.transformation.knowledge.v1`).

If/when a bus-carried intent topic is added for Enterprise Knowledge, it MUST follow `backlog/509-proto-naming-conventions.md` / `backlog/496-eda-protobuf-ameide.md` (one topic family ↔ one aggregator message) and be treated as an optional ingress alongside RPC (not a second writer).

### Initiative / repository lifecycle

- `CreateTransformationInitiativeRequested`
- `UpdateTransformationInitiativeRequested`
- `CreateWorkspaceNodeRequested`
- `MoveWorkspaceNodeRequested`
- `RenameWorkspaceNodeRequested`
- `ReorderWorkspaceNodeChildrenRequested`

### Element lifecycle (generic)

- Element metadata lifecycle is expressed via `Element` updates and version snapshots.
- Publishing/promotions are expressed via baseline/promotion workflows that reference `{element_id, version_id}` pairs.

### Baseline / promotion lifecycle

- `CreateBaselineDraftRequested`
- `AddElementVersionToBaselineRequested`
- `RemoveElementVersionFromBaselineRequested`
- `SubmitBaselineForApprovalRequested`
- `ApproveBaselineRequested` / `RejectBaselineRequested`
- `PromoteBaselineRequested`
- `RollbackBaselineRequested`

### Enterprise Knowledge intents (element-centric substrate)

- `CreateElementRequested`
- `UpdateElementRequested`
- `DeleteElementRequested`
- `CreateElementRelationshipRequested`
- `UpdateElementRelationshipRequested`
- `DeleteElementRelationshipRequested`
- `SnapshotElementVersionRequested`
- `PublishElementVersionRequested` (optional; may be expressed as promotion workflow)

### ArchiMate profile intents (derived/convenience; optional)

- `RecordArchimateViewLayoutRequested` (creates a new version for an ArchiMate view element)

### BPMN intents (design-time ProcessDefinitions)

- `CreateBpmnProcessDefinitionRequested`
- `CreateBpmnProcessVersionRequested`
- `LinkBpmnProcessToElementsRequested`
- `DeprecateBpmnProcessDefinitionRequested` / `ArchiveBpmnProcessDefinitionRequested`

### Definition Registry intents

- `UpsertProcessDefinitionRequested`
- `UpsertScaffoldingPlanDefinitionRequested` (derived from promoted ProcessDefinition)
- `UpsertCompiledWorkflowDefinitionRequested` (derived from promoted ProcessDefinition)
- `UpsertAgentDefinitionRequested`
- `UpsertExtensionDefinitionRequested`
- `PromoteDefinitionRequested`

## 6) Core fact catalog (to define)

Enterprise Knowledge facts are immutable past-tense events transported on `transformation.knowledge.domain.facts.v1` as `TransformationKnowledgeDomainFact` with `meta + subject + oneof fact`.

### Initiative / repository facts

- `TransformationInitiativeCreated`
- `TransformationInitiativeUpdated`
- `WorkspaceNodeCreated`
- `WorkspaceNodeMoved`
- `WorkspaceNodeRenamed`
- `WorkspaceNodeChildrenReordered`

### Element lifecycle facts (generic)

- Element lifecycle is expressed via `Element*` and `ElementVersion*` facts plus baseline/promotion facts.

### Baseline / promotion facts

- `BaselineDraftCreated`
- `BaselineDraftUpdated`
- `BaselineSubmittedForApproval`
- `BaselineApproved` / `BaselineRejected`
- `BaselinePromoted`
- `BaselineRollbackCreated`

### Enterprise Knowledge facts

- `ElementCreated`
- `ElementUpdated`
- `ElementDeleted`
- `ElementRelationshipCreated`
- `ElementRelationshipUpdated`
- `ElementRelationshipDeleted`
- `ElementVersionSnapshotted`
- `ElementVersionPublished` (optional; may be expressed as promotion workflow)

### ArchiMate profile facts (derived/convenience; optional)

- `ArchimateViewLayoutRecorded`

### BPMN facts

- `BpmnProcessDefinitionCreated`
- `BpmnProcessVersionCreated`
- `BpmnProcessElementLinksUpdated`
- `BpmnProcessDefinitionDeprecated` / `BpmnProcessDefinitionArchived`

### Definition Registry facts

- `ProcessDefinitionUpserted`
- `ScaffoldingPlanDefinitionUpserted`
- `CompiledWorkflowDefinitionUpserted`
- `AgentDefinitionUpserted`
- `ExtensionDefinitionUpserted`
- `DefinitionPromoted`

## 7) Acceptance criteria

1. Canonical proto files exist under `packages/ameide_core_proto/src/ameide_core_proto/transformation/**`.
2. Topic families follow the “aggregator envelope” pattern (intents/facts/process facts).
3. Envelopes include required tenant + traceability metadata and monotonic versions for idempotency.
4. No backlog docs embed full proto text (file index only).

## 8) Clarification requests (next steps)

Confirm/decide:

- Lock-in: use split subcontext streams (`transformation.<subcontext>.domain.facts.v1` for knowledge/registry/governance/work) and `transformation.process.facts.v1`; do not introduce a generic `transformation.domain.*` seam.
- The canonical envelope pattern (e.g., `meta + subject + oneof`) and which fields are required across all Transformation message families (including producer identity and monotonic versioning rules).
- The v1 plan for TOGAF/PMI: whether to stub packages now (empty catalogs) or defer until governance workflows are defined.
- The v1 BPMN execution posture: compile-to-Temporal (recommended) vs direct runtime interpretation, and whether collaboration diagrams are required for “deployable/scaffoldable” ProcessDefinitions.
