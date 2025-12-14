# 527 — Transformation Capability (Define the capability, then realize it via primitives)

**Status:** Draft (authoritative once accepted)  
**Audience:** Architecture, platform engineering, operators/CLI, agent/runtime teams  
**Scope:** Define **Transformation** as a *business capability* implemented as Ameide primitives (Domain/Process/Projection/Integration/UISurface/Agent), with 496-native EDA contracts.

This backlog is the *capability definition* counterpart to the *method* in `524-transformation-capability-decomposition.md`.

Discovery driver: treat `524-transformation-capability-decomposition.md` as the authoritative discovery workflow. This document records the **future-state outcome** for the Transformation capability, but package/topic/primitive decisions should be backed by the artifacts produced in Steps 0–3 (glossary, value streams, EDA topology, proto proposal) and validated by a Step 5 acceptance slice.

## Layer header (Strategy → Application realization)

- **Primary ArchiMate layer(s):** Strategy (Capability) and Application (realization via Application Components + Services/Interfaces/Events).
- **Secondary layers referenced:** Business, Technology, Implementation & Migration.
- **Primary element types used:** Capability, Value Stream, Business Process, Application Component, Application Service/Interface/Event, Technology Service, Work Package/Gap.
- **Prohibited unless qualified:** process, service, domain, event (qualify per `529-archimate-alignment-470plus.md`).

## Future state (executive summary)

Transformation is Ameide’s change-the-business capability. In the future state it provides:

- A typed **Enterprise Repository** as the system of record for transformation initiatives and design-time artifacts.
- Multi-methodology **profiles** (Scrum, TOGAF/ADM, PMI) as bounded contexts that govern how the repository is used.
- A **Definition Registry** for platform definitions (ProcessDefinition, AgentDefinition, ExtensionDefinition, etc.) stored as domain data and consumed by operators and runtimes.
- 496-native EDA contracts so Process primitives and Agents can execute work deterministically without runtime RPC coupling.

## TOGAF ADM positioning (how to read this doc)

If we treat the Transformation capability itself as something we architect using TOGAF ADM, this backlog primarily captures the “future state” architecture across **Phases B–E**:

- **B (Business Architecture):** Transformation value streams, business processes, governance outcomes.
- **C (Information Systems Architectures):** contract-first EDA topology, repository/definition data model, query surfaces.
- **D (Technology Architecture):** realization assumptions via primitives/operators/broker/DB/workflow runtime (delegated to `520-primitives-stack-v2.md`).
- **E (Opportunities & Solutions):** primitive decomposition and initial slices that prove the seam.

**Phases F–H** are delivered/operated by applying the `524` method repeatedly inside a transformation initiative:

- **F (Migration Planning):** acceptance slices + work packages and migration stance.
- **G (Implementation Governance):** promotion gates, CI guardrails, operator conditions, and process facts.
- **H (Architecture Change Management):** the continuous loop where agents and governance flows drive tenants through `524` as change work.

## 0) Problem

Today “Transformation” exists as:

- a **legacy service** (`services/transformation`) implementing a partial `TransformationService` CRUD surface,
- an emerging **Scrum contract** (`ameide_core_proto.transformation.scrum.v1` intents/facts/query + DB tables + outbox migration),
- a **target architecture intent** across backlogs (Transformation stores definitions; Process operator fetches definitions; runtime is bus-only),

…but we lack a single authoritative backlog that says:

> “Transformation is a capability. These are its domains/processes/surfaces/integrations, and these are its EDA contracts.”

This causes ambiguity about:

- system-of-record boundaries (who is allowed to write what),
- what is design-time vs runtime,
- how capability architecture (like `523-commerce*`) is stored and linked to primitives.

## 1) Definition: what the Transformation capability is

Transformation is the platform’s **change-the-business capability**: it captures *what should be built*, *how it should be built*, and *how it is governed*.

It is not “a modeling UI” and not “a Temporal service”; it is a capability whose primitives provide:

- the **Enterprise Repository** (system of record) for transformation initiatives and design-time architecture/process artifacts (typed models, not files),
- the **definition registry** for design-time definitions (process/agent/extension) as first-class domain data (not ad-hoc configs),
- the **EDA-native contracts** that Process primitives and Agents can consume to execute work deterministically.

## 1.1) Strategy: outcomes and value streams (Transformation)

Primary outcomes (future state):

- Tenants can create a transformation initiative and build/operate capability architecture using a repeatable methodology.
- “Design-time truth” (ArchiMate/BPMN/definitions) is governable, versioned, promotable, and queryable.
- Runtime execution is deterministic: Process primitives and Agents drive work from contracts and facts, not ad-hoc RPC calls.

Value streams (starter set; refine via `524` Step 1):

- **Initiate**: create initiative → lock glossary/identity axes → establish repository structure.
- **Design**: model capability/value streams → derive contracts/topology → propose proto packages/topics → promote baselines.
- **Realize**: decompose to primitives → scaffold → verify → promote definitions.
- **Govern**: run methodology gates (ADM phases, Scrum timeboxes, PMI stage gates) → manage drift and compliance.

## 2) Core invariants (non-negotiable)

1. **Capability is realized by primitives.** Transformation is not a monolith; it is realized via Domain/Process/Projection/Integration/UISurface/Agent application components.
2. **Design-time vs runtime separation.**
   - Design-time artifacts are stored in Transformation (as domain data).
   - Runtime workflows execute in Process primitives.
3. **496-native EDA.**
   - commands/intents vs facts, outbox for writers, idempotent consumers, sagas in Process primitives.
4. **No runtime RPC coupling.**
   - Runtime Process workflows MUST NOT call Transformation synchronously to mutate/read domain state; they must use bus intents/facts.
   - Synchronous reads of definitions by operators are *control-plane only*.
5. **Tenant isolation and traceability metadata** on all messages.
6. **Typed design-time store.**
   - ArchiMate is stored as a typed model (elements/relationships/views as first-class tables).
   - “Files” exist only as attachments/export formats (imports/exports), not the canonical writer surface.
7. **Multi-methodology by profile.**
   - Transformation supports multiple methodology profiles; Scrum, TOGAF/ADM, and PMI are profiles; others can be added.
8. **Profiles govern; they don’t redefine the repository.**
   - Methodology profiles apply governance and constraints over the Enterprise Repository; they do not fork the repository’s canonical data model.

## 3) Capability boundaries and nouns

Transformation capability owns (at minimum):

- **Transformation initiative** (portfolio/program object; “what we are doing and why”)
- **Enterprise Repository workspace tree** (nodes/folders/scopes for artifacts)
- **Artifact registry** (typed objects, attachments, revisions/promotions)
- **Typed ArchiMate model** (elements/relationships/properties) and **views** (view definitions)
- **Typed BPMN process models** (ProcessDefinitions as first-class design-time artifacts)
- **Methodology profiles** (Scrum, TOGAF/ADM, PMI), implemented as bounded contexts
- **Definitions** used by the platform (ProcessDefinition, AgentDefinition, ExtensionDefinition, etc.)

Rules:

- Graph is a read-optimized projection/index; Transformation is canonical for transformation artifacts and definitions.
- Other capabilities may reference Transformation artifacts by stable IDs; they do not become writers of those artifacts.

## 4) Application realization (primitive decomposition)

### 4.0 Enterprise Repository (design-time system of record)

Transformation’s Domain primitive owns an Enterprise Repository schema aligned with TOGAF-style organization, but implemented as a multi-methodology store with typed design-time languages.

- **Repository hierarchy:** folders/nodes/scopes for initiatives, work packages, architecture artifacts, process models, and supporting documentation.
- **Typed architecture model (ArchiMate):** elements, relationships, properties/tags, stable identifiers; views are first-class objects.
- **Typed process models (BPMN):** ProcessDefinitions are versioned/promotable first-class objects.
- **Generic artifacts:** Markdown/ADRs/diagrams/attachments and import/export bundles exist as attachments (not canonical state).

Canonical writer rule: the Enterprise Repository is the writer for the typed model; external file formats are import/export attachments that can be re-derived.

Repository scope (future-state “first-class tables” conceptually):

- Initiative → WorkspaceNode → Artifact → Revision → Promotion/Baseline
- ArchiMate: Model → Element / Relationship → View → ViewNode/ViewEdge (layout) → ViewRevision
- BPMN: ProcessDefinition → ProcessVersion → ProcessArtifactRefs (links to repository artifacts)

#### ArchiMate 3.2 metamodel support (full; non-MVP)

The Enterprise Repository stores the full ArchiMate 3.2 element set and relationship set as typed records. This is not a “subset MVP”: the future state assumes complete concept coverage so teams are not blocked by missing metamodel primitives.

Element types (grouped by ArchiMate layer/category):

- **Strategy:** Resource, Capability, Value Stream, Course of Action.
- **Business:** Business Actor, Business Role, Business Collaboration, Business Interface, Business Process, Business Function, Business Interaction, Business Event, Business Service, Business Object, Contract, Representation, Product.
- **Application:** Application Component, Application Collaboration, Application Interface, Application Function, Application Interaction, Application Process, Application Event, Application Service, Data Object.
- **Technology:** Node, Device, System Software, Technology Collaboration, Technology Interface, Technology Function, Technology Process, Technology Interaction, Technology Event, Technology Service, Path, Communication Network, Artifact.
- **Physical:** Equipment, Facility, Distribution Network, Material.
- **Motivation:** Stakeholder, Driver, Assessment, Goal, Outcome, Principle, Requirement, Constraint, Meaning, Value.
- **Implementation & Migration:** Work Package, Deliverable, Plateau, Gap.
- **Other (cross-cutting):** Location, Grouping, Junction.

Relationship types (minimum full set expected):

- Structural: Composition, Aggregation, Assignment, Realization, Serving, Access, Association, Specialization.
- Dependency: Influence.
- Dynamic: Triggering, Flow.

View model support:

- Views are first-class objects with explicit membership and layout (view nodes/edges), so UIs can edit/view without relying on “file” formats as canonical state.
- Validation rules are ArchiMate-consistent by default (relationship-type compatibility by endpoint types), but can be configured to “warn” (not “reject”) during import/migration phases.

#### Data model (Enterprise Repository; minimum tables)

This is the minimum relational data model implied by “typed store + revisions/promotions” (names illustrative; schema details are defined by `ameide_core_proto.transformation.core.v1` and migrations):

- `transformation_initiatives` (tenant/org, key, name, stage, metadata)
- `workspace_nodes` (tree: parent_id, slug, order, attributes)
- `artifacts` (type, workspace_node_id, canonical_ref, tags)
- `artifact_revisions` (artifact_id, revision_id, created_by, created_at, content_ref?, digest, schema_version)
- `artifact_promotions` / `baselines` (initiative_id, label, promoted_revision_ids, policy, approved_by, approved_at)
- `attachments` (artifact_id or revision_id, uri, content_type, size, checksum)

Typed ArchiMate:

- `archimate_models` (initiative_id, model_id, name, versioning metadata)
- `archimate_elements` (model_id, element_id, element_type, name, documentation, properties JSON, external_refs)
- `archimate_relationships` (model_id, relationship_id, relationship_type, source_element_id, target_element_id, name?, properties JSON)
- `archimate_views` (model_id, view_id, viewpoint?, name, documentation, properties JSON)
- `archimate_view_nodes` (view_id, element_id, x/y/w/h, style JSON, z_index, label?)
- `archimate_view_edges` (view_id, relationship_id, source_node_ref?, target_node_ref?, bendpoints JSON, style JSON)
- `archimate_view_revisions` (view_id, revision_id, snapshot pointers for nodes/edges)

Typed BPMN:

- `bpmn_process_definitions` (initiative_id, process_definition_id, name, status, metadata)
- `bpmn_process_versions` (process_definition_id, version, spec_ref, created_at, created_by)
- `bpmn_artifact_refs` (process_version_id, artifact_id or archimate_view_id, role)

Definition Registry:

- `process_definitions` (definition_id, version, bpmn_process_version_id or spec_ref, status, governance)
- `agent_definitions` (definition_id, version, risk_tier, tool_grants, compiled_artifact_ref?)
- `extension_definitions` (definition_id, version, kind, schema_ref, config_ref)

Indexing/non-negotiables:

- Every table is tenant-scoped and uses stable IDs.
- Facts are emitted only after persistence; domain facts include monotonic aggregate versions for idempotent downstream consumption.

### 4.1 Domain primitives (system-of-record)

**A) `transformation-domain` (core)**

Responsibilities:

- Own canonical Transformation domain state (target: `ameide_core_proto.transformation.core.v1`) including:
  - transformations (initiative records),
  - workspace nodes,
  - milestones/metrics/alerts,
  - Enterprise Repository typed artifacts (ArchiMate model + views, BPMN ProcessDefinitions, generic artifacts),
  - design-time definition storage (see §6).
- Emit domain facts for significant state changes (see §5).

Compatibility note (migration): `ameide_core_proto.transformation.v1` and its `TransformationService` are treated as a legacy façade during migration; new canonical contracts live in `…transformation.core.v1` per `509-proto-naming-conventions.md`.

**B) `transformation-scrum-domain` (methodology bounded context)**

Responsibilities:

- Own Scrum artifacts as a true domain (Product Backlog, PBI, Sprint, Increment, DoD, Goals).
- Accept change via `ScrumDomainIntent` (bus).
- Emit `ScrumDomainFact` via transactional outbox (bus).
- Provide `ScrumQueryService` as read-only RPC.

**C) `transformation-togaf-domain` (methodology bounded context, profile)**

Responsibilities:

- Own TOGAF/ADM profile configuration as domain data bound to a transformation initiative (phases, deliverables, gates, policies).
- Emit domain facts for profile configuration changes.

**D) `transformation-pmi-domain` (methodology bounded context, profile)**

Responsibilities:

- Own PMI profile configuration and governance artifacts (stage gates, plans/baselines, risks/issues/changes as needed by the chosen PMI subset).
- Emit domain facts for profile configuration changes.

### 4.2 Process primitives (governance flows executed by Process primitives)

Responsibilities:

- Timeboxes, governance, approvals, and orchestration driven by design-time ProcessDefinitions:
  - TOGAF ADM governance flows (phase progression, required deliverables, review/approval gates).
  - Scrum timebox governance (Sprint start/end windows, reminders, readiness cues) per `506-scrum-vertical-v2.md`
  - PMI governance flows (stage gates, approvals, reporting cadence) per the PMI profile definition.
  - “requirement → running” orchestration (scaffold, verify, promote) for primitives and definitions
- Emit process facts (governance cues) distinct from domain facts.
- Never become a system-of-record for methodology artifacts.

### 4.3 Projection primitives (read models)

Responsibilities:

- Build read models for portals and agents:
  - portfolio dashboards,
  - “capability architecture” browsing,
  - methodology analytics (throughput, cycle time),
  - definition indexing and dependency views.
- Idempotent consumption (inbox or UPSERT) per `496-eda-principles.md`.

Projection read model inventory (full scope; supports the full UISurface):

- **WorkspaceTreeProjection**: denormalized workspace tree + artifact counts + promotion status per node.
- **ArtifactIndexProjection**: full-text and faceted search over artifacts/revisions (type, tags, owners, baseline membership).
- **BaselineHistoryProjection**: promotion/baseline timeline, approval history, diffs between baselines.
- **ArchiMateGraphProjection**: element/relationship graph for impact analysis (neighbors, dependency paths, realizations).
- **ArchiMateViewProjection**: view rendering payloads (nodes/edges/layout/style) optimized for canvas UI.
- **ArchiMateMatrixProjection**: relationship matrices by type (e.g., application components realize services; services serve business processes).
- **BpmnCatalogProjection**: ProcessDefinitions + versions + links to artifacts/views + governance state.
- **DefinitionRegistryProjection**: process/agent/extension definitions with versions, promotion state, deployment references.
- **GovernanceStatusProjection**: TOGAF/PMI/Scrum gate state per initiative (what is pending, what is approved, what is blocked).
- **AuditTrailProjection**: human/audit friendly timeline from domain facts + process facts (who/what/when/why).

Query surfaces (read-only Application Services) are backed by these projections and should be stable, paginated, and filterable; they are the primary read path for UISurfaces and Agents.

### 4.4 UISurface primitives (experiences)

Responsibilities:

- Transformation portal / design tooling UIs:
  - initiative views,
  - workspace browser,
  - modeling editors (typed ArchiMate views, BPMN, Markdown),
  - definition management and promotion flows.
- UISurfaces are thin: they query projections and emit commands/intents.

### 4.5 Agent primitives (assistants invoked by UISurface or Process)

Responsibilities:

- Agents are primitives: invoked interactively (chat in UISurface) or non-interactively (by Process workflows).
- Read transformation context and propose/prepare changes.
- Emit commands/intents only through governed seams.
- Tooling grants and risk-tier enforcement are stored as definitions (AgentDefinitions), typically one per role (SA/TA/PM/etc.).

### 4.6 Integration primitives (external boundaries)

Responsibilities:

- Backstage scaffolder, GitHub, CI, container registry, ticketing, etc.
- Strict idempotency on inbound webhooks and retries.

## 5) EDA contracts (message topology)

Transformation capability uses three message classes (496):

- **Domain intents** (imperative): request a state change.
- **Domain facts** (immutable): emitted after persistence.
- **Process facts** (governance): emitted by Process primitives for orchestration cues.

Proto packages (authoritative):

- Core Transformation (target): `ameide_core_proto.transformation.core.v1`
- TOGAF profile: `ameide_core_proto.transformation.togaf.v1`
- Scrum profile: `ameide_core_proto.transformation.scrum.v1`
- PMI profile: `ameide_core_proto.transformation.pmi.v1`

Topic families (runtime seam):

- Core Transformation:
  - `transformation.domain.intents.v1` → `ameide_core_proto.transformation.core.v1.TransformationDomainIntent` (to be defined)
  - `transformation.domain.facts.v1` → `ameide_core_proto.transformation.core.v1.TransformationDomainFact` (to be defined)
- TOGAF profile:
  - `togaf.domain.intents.v1` → `ameide_core_proto.transformation.togaf.v1.TogafDomainIntent` (to be defined)
  - `togaf.domain.facts.v1` → `ameide_core_proto.transformation.togaf.v1.TogafDomainFact` (to be defined)
  - `togaf.process.facts.v1` → `ameide_core_proto.process.togaf.v1.TogafProcessFact` (to be defined)
- Scrum profile (already exists):

- `scrum.domain.intents.v1` → `ameide_core_proto.transformation.scrum.v1.ScrumDomainIntent`
- `scrum.domain.facts.v1` → `ameide_core_proto.transformation.scrum.v1.ScrumDomainFact`
- `scrum.process.facts.v1` → `ameide_core_proto.process.scrum.v1.ScrumProcessFact`
- PMI profile:
  - `pmi.domain.intents.v1` → `ameide_core_proto.transformation.pmi.v1.PmiDomainIntent` (to be defined)
  - `pmi.domain.facts.v1` → `ameide_core_proto.transformation.pmi.v1.PmiDomainFact` (to be defined)
  - `pmi.process.facts.v1` → `ameide_core_proto.process.pmi.v1.PmiProcessFact` (to be defined)

Action for this backlog:

- Define core Transformation topic families for initiative/repository/definition lifecycle changes, using the same “aggregator envelope” pattern as Scrum.
- Define TOGAF/ADM profile topic families (domain intents/facts + process facts) following the same contract shape, so “multi-methodology by profile” is real and executable.
- Define PMI profile topic families (domain intents/facts + process facts) following the same contract shape.

### 5.1 Core Transformation domain (Enterprise Repository) — aggregate set (authoritative)

The core Transformation domain (`ameide_core_proto.transformation.core.v1`) treats the Enterprise Repository as a set of single-writer aggregates (exact persistence model may vary, but writer boundaries are strict):

- **Initiative** (transformation initiative)
- **WorkspaceNode** (repository hierarchy)
- **Artifact** (typed, versioned domain object; includes ArchiMate/BPMN/definitions/generic artifacts)
- **ArtifactRevision** (immutable revision of an artifact)
- **Baseline / Promotion** (a promotable set of revisions with governance metadata)

Typed artifacts within the Enterprise Repository (typically implemented as their own aggregates or as Artifact specializations):

- **ArchiMateModel**
- **ArchiMateElement**
- **ArchiMateRelationship**
- **ArchiMateView** (plus ViewNode/ViewEdge layout state)
- **BpmnProcessDefinition** (plus ProcessVersion)
- **ProcessDefinition** (Definition Registry)
- **AgentDefinition** (Definition Registry)
- **ExtensionDefinition** (Definition Registry)

Idempotency/order rule: all Domain facts must carry a monotonic per-aggregate version; consumers (Projections, Processes, Agents) are idempotent.

### 5.2 Core Transformation Domain intents (catalog)

Core intents are imperative requests to change Enterprise Repository state. They are transported on `transformation.domain.intents.v1` as `TransformationDomainIntent` with a stable envelope (`meta` + `subject`) and a `oneof intent { ... }` payload.

#### Initiative / repository lifecycle

- `CreateTransformationInitiativeRequested`
- `UpdateTransformationInitiativeRequested`
- `CreateWorkspaceNodeRequested` (create folder/scope node)
- `MoveWorkspaceNodeRequested` (reparent within tree)
- `RenameWorkspaceNodeRequested`
- `ReorderWorkspaceNodeChildrenRequested`

#### Artifact lifecycle (generic)

- `CreateArtifactRequested` (typed artifact “root” with identity and kind)
- `ReviseArtifactRequested` (create new ArtifactRevision; content is typed or attachment-ref)
- `AttachArtifactBlobRequested` (store/update out-of-band blob; revision references it)
- `TagArtifactRequested` / `UntagArtifactRequested`
- `DeprecateArtifactRequested` / `ArchiveArtifactRequested`

#### Baseline / promotion lifecycle (Phase G/H governance surfaced as domain state)

- `CreateBaselineDraftRequested`
- `AddArtifactRevisionToBaselineRequested`
- `RemoveArtifactRevisionFromBaselineRequested`
- `SubmitBaselineForApprovalRequested`
- `ApproveBaselineRequested` / `RejectBaselineRequested`
- `PromoteBaselineRequested` (create a promoted baseline/plateau record)
- `RollbackBaselineRequested` (create a new baseline pointing to previous set; no destructive deletes)

#### ArchiMate model intents (typed; no file-as-canonical)

- `CreateArchiMateModelRequested`
- `UpsertArchiMateElementRequested` (create/update name/type/properties)
- `UpsertArchiMateRelationshipRequested` (create/update endpoints/type/properties)
- `DeleteArchiMateElementRequested` (logical delete; governance may restrict)
- `DeleteArchiMateRelationshipRequested` (logical delete)

#### ArchiMate view + layout intents

- `CreateArchiMateViewRequested`
- `UpdateArchiMateViewPropertiesRequested` (name/viewpoint/properties)
- `UpsertArchiMateViewNodeRequested` (place/resize/style an element in a view)
- `UpsertArchiMateViewEdgeRequested` (route/style relationship in a view)
- `DeleteArchiMateViewNodeRequested`
- `DeleteArchiMateViewEdgeRequested`
- `SnapshotArchiMateViewRevisionRequested` (capture a stable view revision for promotion/baselines)

#### BPMN intents (design-time ProcessDefinitions)

- `CreateBpmnProcessDefinitionRequested`
- `CreateBpmnProcessVersionRequested` (new version)
- `LinkBpmnProcessToArtifactsRequested` (attach repository references: views, docs, requirements)
- `DeprecateBpmnProcessDefinitionRequested` / `ArchiveBpmnProcessDefinitionRequested`

#### Definition Registry intents

- `UpsertProcessDefinitionRequested` (binds to BPMN version/spec ref; sets status)
- `UpsertAgentDefinitionRequested` (role-based agents; tool grants/risk tier)
- `UpsertExtensionDefinitionRequested`
- `PromoteDefinitionRequested` (definitions are promoted like baselines; consumed by operators)

### 5.3 Core Transformation Domain facts (catalog)

Core facts are immutable past-tense events transported on `transformation.domain.facts.v1` as `TransformationDomainFact` with `oneof fact { ... }`, emitted only after persistence.

#### Initiative / repository facts

- `TransformationInitiativeCreated`
- `TransformationInitiativeUpdated`
- `WorkspaceNodeCreated`
- `WorkspaceNodeMoved`
- `WorkspaceNodeRenamed`
- `WorkspaceNodeChildrenReordered`

#### Artifact facts

- `ArtifactCreated`
- `ArtifactRevised`
- `ArtifactBlobAttached`
- `ArtifactTagged` / `ArtifactUntagged`
- `ArtifactDeprecated` / `ArtifactArchived`

#### Baseline / promotion facts

- `BaselineDraftCreated`
- `BaselineDraftUpdated` (add/remove revisions)
- `BaselineSubmittedForApproval`
- `BaselineApproved` / `BaselineRejected`
- `BaselinePromoted`
- `BaselineRollbackCreated`

#### ArchiMate model facts

- `ArchiMateModelCreated`
- `ArchiMateElementUpserted`
- `ArchiMateRelationshipUpserted`
- `ArchiMateElementDeleted`
- `ArchiMateRelationshipDeleted`

#### ArchiMate view/layout facts

- `ArchiMateViewCreated`
- `ArchiMateViewPropertiesUpdated`
- `ArchiMateViewNodeUpserted`
- `ArchiMateViewEdgeUpserted`
- `ArchiMateViewNodeDeleted`
- `ArchiMateViewEdgeDeleted`
- `ArchiMateViewRevisionSnapshotted`

#### BPMN facts

- `BpmnProcessDefinitionCreated`
- `BpmnProcessVersionCreated`
- `BpmnProcessArtifactLinksUpdated`
- `BpmnProcessDefinitionDeprecated` / `BpmnProcessDefinitionArchived`

#### Definition Registry facts

- `ProcessDefinitionUpserted`
- `AgentDefinitionUpserted`
- `ExtensionDefinitionUpserted`
- `DefinitionPromoted`

## 6) Definition Registry (design-time artifacts) inside Transformation

Target: Transformation becomes the canonical store for design-time artifacts:

- `ProcessDefinition` (design-time; referenced by Process CRs; fetched by Process operator as control-plane config)
- `AgentDefinition` (design-time; fetched by Agent operator; governs agent runtime)
- `ExtensionDefinition` / `PrimitiveImplementationDraft` (design-time; drives scaffolding/promotion processes)
- Capability architecture artifacts (ArchiMate models/views, Markdown, proto proposals), as transformation workspace elements

This implies a schema-backed model, not just markdown docs:

- `CapabilityDefinition` (capability identity, scope, nouns/topology)
- `CapabilityEDAContract` (topic families, envelopes, intent/fact catalogs)
- `CapabilityPrimitiveDecomposition` (primitives + responsibilities)

This backlog does not force the exact proto/package yet, but it makes the requirement explicit: the methodology in `524` must become domain data in Transformation.

### 6.2 Query service surface (full scope; non-MVP)

In addition to projections, the Transformation capability exposes stable, read-only query APIs (Application Services) for portal/agents and control-plane operator reads. These are explicitly separate from runtime bus coupling.

Minimum query services expected in `ameide_core_proto.transformation.core.v1`:

- **TransformationQueryService**
  - list/get initiatives, list milestones/metrics/alerts
- **WorkspaceQueryService**
  - get workspace tree, browse nodes, list artifacts under node(s), search
- **ArtifactQueryService**
  - get artifact, list revisions, diff revisions, list attachments, list baseline membership
- **BaselineQueryService**
  - list baselines, get baseline, compare baselines, show approval history
- **ArchiMateQueryService**
  - get model, list elements/relationships, get element/relationship, graph traversal queries
- **ArchiMateViewQueryService**
  - get view render payload, list views, get view revision
- **BpmnQueryService**
  - list process definitions/versions, get process spec refs, list linked artifacts
- **DefinitionRegistryQueryService**
  - list/get ProcessDefinitions, AgentDefinitions, ExtensionDefinitions, show version/promotion status

Query service shape rules:

- Read-only RPCs only (no mutations).
- Pagination/filtering are mandatory for list/search surfaces.
- Responses must be projection-backed for scale; the domain DB is not used as a “read API” directly except for small lookups.

## 6.1) “Agent drives tenants through 524” (future capability behavior)

In the future state, a role-based Agent (invoked by UISurface chat and/or Process workflows) guides a tenant through the `524` method inside a Transformation initiative:

- Creates/updates the required artifacts (glossary, value streams, EDA topology, proto proposal, decomposition, acceptance slice).
- Uses governance flows (TOGAF/ADM, Scrum, PMI) to enforce review/approval gates and promotions.
- Never bypasses writer boundaries: it proposes changes and emits intents; domains persist and emit facts; processes orchestrate.

## 7) Legacy mapping (current state)

### 7.1 `services/transformation` (legacy)

Current behavior:

- Implements partial `TransformationService` CRUD and agent definition upsert/get.
- Does not implement workspace/milestones/metrics/alerts.
- Does not implement EDA contracts for Scrum intents/facts or outbox publishing.

Role in target:

- Either becomes a façade while Domain primitives take over, or is evolved into a Domain-primitive-shaped implementation (outbox + bus + strict boundaries).

### 7.2 `primitives/domain/transformation` (scaffold)

Current behavior:

- Scaffold only, unimplemented handlers for `ScrumQueryService`.
- Has a generic outbox migration but is not wired to Transformation schema or Scrum tables.

Role in target:

- Becomes the canonical implementation of the Transformation bounded contexts (core + scrum), consistent with the primitives stack and 496.

## 8) Fit/Gap summary (what this backlog drives)

### Fits

- Protos exist for Transformation core entities and Scrum contracts.
- DB migrations exist for Scrum tables and a Scrum domain outbox.
- Operator backlogs already define the runtime seam: Process ↔ Transformation via bus.

### Gaps

- No canonical “Transformation capability” decomposition doc (this backlog fixes that).
- No core Transformation intent/fact topic families defined for initiative/workspace/definition lifecycle.
- No implemented Scrum domain runtime (intent ingestion + outbox publication + query RPC).
- Ambiguous writer boundaries between services (repository vs transformation vs future domain primitive).

## 9) Acceptance criteria

1. Transformation is described and operated as a **capability realized via primitives**, not as a single service.
2. There is an explicit **core Transformation EDA contract** (topic families + envelopes) mirroring the Scrum pattern.
3. There is a clear **migration stance**: what becomes canonical writer and what becomes projection/facade.
4. One end-to-end slice exists that proves the seam (example):
   - `ScrumDomainIntent` → domain persistence → outbox → `ScrumDomainFact` → Process reacts → emits `ScrumProcessFact` → UISurface reads via `ScrumQueryService` / projections.

## Phase G — Implementation Governance plan (full implementation)

This section is the Phase G “how we govern implementation” plan for delivering the full future state (no MVP shortcuts), using the `524` method as the execution loop and the v2 primitive stack (`520-primitives-stack-v2.md`) as the conformance baseline.

### G.0 Governance gates (non-negotiable)

- **Contract gates:** `buf lint`, `buf breaking`, deterministic `buf generate`, regen-diff, and SDK-only import policy (`520-primitives-stack-v2.md`).
- **Runtime gates:** compile/test for each primitive runtime; determinism constraints for Process primitives; state discipline for Agents; offsets/idempotency for Projections.
- **Deployment gates:** images are built/published deterministically and scanned; GitOps sync + smoke probes must pass; operators surface conditions.
- **Promotion gates:** design-time artifacts and definitions are promoted only through governed flows (Process facts + approvals), never by direct DB edits.

### G.1 Work packages (plateaus) for full scope

These work packages are executed as repeatable `524` loops, each ending in a promotable baseline and an in-cluster proof.

1. **G1 — Contract foundation (core + profiles)**
   - Define and freeze v1 proto surfaces for:
     - `ameide_core_proto.transformation.core.v1` (Enterprise Repository + Definition Registry + core intents/facts + query services)
     - `ameide_core_proto.transformation.togaf.v1` (TOGAF/ADM profile intents/facts)
     - `ameide_core_proto.transformation.pmi.v1` (PMI profile intents/facts)
     - `ameide_core_proto.process.{togaf,pmi}.v1` (process facts)
   - Lock topic families and envelope invariants (meta, subject, aggregate ref/version) to match Scrum’s proven seam.
   - Scaffold contract consumers/producers via CLI so compilation drift is immediate:
     - `ameide primitive scaffold --kind domain --name transformation`
     - `ameide primitive scaffold --kind process --name togaf`
     - `ameide primitive scaffold --kind process --name pmi`
     - `ameide primitive scaffold --kind projection --name transformation`
     - `ameide primitive scaffold --kind uisurface --name transformation`
     - `ameide primitive scaffold --kind agent --name transformation`

2. **G2 — Enterprise Repository implementation (Domain)**
   - Implement full ArchiMate 3.2 typed store (all element and relationship types) + views/layout + revision/promotion model.
   - Implement BPMN ProcessDefinition storage/versioning and linkage to repository artifacts.
   - Implement transactional outbox publishing for core facts; implement query/read APIs for portal and operator control plane.
   - Ensure the domain primitive is buildable and publishable end-to-end:
     - `buf lint && buf breaking && buf generate`
     - `go test ./...` (and/or language-equivalent tests)
     - `ameide primitive publish --kind domain --name transformation`

3. **G3 — Profile domains (Scrum/TOGAF/PMI)**
   - Scrum: complete the existing bounded context implementation and outbox publishing per `506-scrum-vertical-v2.md`.
   - TOGAF/ADM: implement profile configuration domain (phases, deliverables, gates, policies) + facts.
   - PMI: implement profile configuration and governance artifacts domain + facts.

4. **G4 — Governance execution (Process primitives)**
   - Implement Temporal-backed Process primitives for TOGAF/ADM and PMI governance (ingress router + deterministic workflows + process outbox).
   - Ensure all cross-domain writes occur via domain intents; processes emit only process facts and governance cues.
   - Scaffold/process delivery loop is mandatory:
     - `ameide primitive scaffold --kind process --name togaf` and `ameide primitive scaffold --kind process --name pmi`
     - `ameide primitive publish --kind process --name togaf` and `ameide primitive publish --kind process --name pmi`

5. **G5 — Projections and portal (Projection + UISurface)**
   - Projections: repository browsing, architecture dependency views, promotion history, governance dashboards, methodology analytics.
   - UISurface (full UX scope):
     - Initiative dashboard (status, baselines/plateaus, governance state)
     - Repository browser (workspace tree + artifacts + revisions + promotions)
     - ArchiMate model browser (elements/relationships lists, filters, search)
     - ArchiMate view editor (canvas layout, relationship routing, style, viewpoints)
     - Relationship matrix / impact analysis (driven by projections)
     - BPMN process browser/editor (ProcessDefinition + versions + links to artifacts)
     - Definition registry manager (ProcessDefinitions/AgentDefinitions/Extensions, versioning, promotion)
     - Governance console (TOGAF phase gates, PMI stage gates, approvals, audit trail)
     - Chat + activity feed (agent conversation, generated proposals, applied changes)
   - UISurface implementation rule: scaffold via CLI and keep generated vs implementation-owned boundaries per `520-primitives-stack-v2.md`.
     - `ameide primitive scaffold --kind uisurface --name transformation`
     - `ameide primitive publish --kind uisurface --name transformation`

6. **G6 — Agent-driven `524` execution (Agent + ProcessDefinition)**
   - Store AgentDefinitions per role; wire tool grants and risk-tier gating.
   - Store the `524` workflow itself as a ProcessDefinition and execute it per initiative, with the agent producing and updating required artifacts.

7. **G7 — Hardening and migration**
   - Multi-tenant isolation, auditability, backup/restore, retention policies, and SLOs for repository and projections.
   - Migration off legacy `ameide_core_proto.transformation.v1` / `services/transformation` façades, with coexistence windows and explicit deprecation milestones.

### G.2 Delivery mechanics (what “implementation governance” enforces day-to-day)

- The `524` Step 6 loop is mandatory for every work package: scaffold → generate → compile/test → publish images → GitOps sync → smoke probes.
- Promotion is explicit: baselines/plateaus are created only when contract gates and deployment gates are green.
- Drift is managed continuously via `524` Step 7 (TOGAF Phase H): new slices are created when contracts or runtime signals indicate change is needed.
