# 527 Transformation — Proto Contract Specification

**Status:** Draft  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **proto contracts** for the Transformation capability following `backlog/496-eda-principles.md` and `backlog/509-proto-naming-conventions.md`.

> **No embedded proto text.** This is a **file index + invariants** specification. Full proto definitions live in the canonical file locations and are the source of truth.

---

## Layer header (Application contracts)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Service (RPC/query), Application Interface (topic families), Application Event (facts), Data Object (proto messages/envelopes).
- **Out-of-scope layers:** Strategy/Business definition (see `backlog/527-transformation-capability.md`) and Technology runtime selection.

## 1) Packages (authoritative)

- Core Transformation (target): `ameide_core_proto.transformation.core.v1`
- Scrum profile (exists): `ameide_core_proto.transformation.scrum.v1`
- TOGAF profile (target): `ameide_core_proto.transformation.togaf.v1`
- PMI profile (target): `ameide_core_proto.transformation.pmi.v1`
- Process facts:
  - Scrum: `ameide_core_proto.process.scrum.v1` (exists)
  - TOGAF: `ameide_core_proto.process.togaf.v1` (target)
  - PMI: `ameide_core_proto.process.pmi.v1` (target)

## 2) Topic families (runtime seams)

| Topic family | Message type | Purpose |
|------------|--------------|---------|
| `transformation.domain.intents.v1` | `TransformationDomainIntent` | Requests to mutate core Transformation state |
| `transformation.domain.facts.v1` | `TransformationDomainFact` | Facts emitted after persistence |
| `scrum.domain.intents.v1` | `ScrumDomainIntent` | Requests to mutate Scrum profile state |
| `scrum.domain.facts.v1` | `ScrumDomainFact` | Scrum facts emitted after persistence |
| `scrum.process.facts.v1` | `ScrumProcessFact` | Scrum workflow facts (Temporal) |
| `togaf.domain.intents.v1` | `TogafDomainIntent` | Requests to mutate TOGAF profile state |
| `togaf.domain.facts.v1` | `TogafDomainFact` | TOGAF facts emitted after persistence |
| `togaf.process.facts.v1` | `TogafProcessFact` | TOGAF governance workflow facts |
| `pmi.domain.intents.v1` | `PmiDomainIntent` | Requests to mutate PMI profile state |
| `pmi.domain.facts.v1` | `PmiDomainFact` | PMI facts emitted after persistence |
| `pmi.process.facts.v1` | `PmiProcessFact` | PMI governance workflow facts |

## 3) Envelope invariants (per 496)

All envelopes (domain intents, domain facts, process facts) must include:

- Tenant isolation: `tenant_id` (required)
- Traceability: `message_id`, `correlation_id`, `causation_id` (when available), timestamps
- Producer identity and schema/version metadata
- Aggregate/process subject with stable identity and monotonic versioning where applicable

## 4) Core aggregate set (Enterprise Repository; authoritative writers)

Core Transformation treats the Enterprise Repository as a set of single-writer aggregates:

- Initiative
- WorkspaceNode
- Artifact, ArtifactRevision
- Baseline/Promotion
- Typed artifacts within the repository:
  - ArchiMateModel / Element / Relationship
  - ArchiMateView (+ layout nodes/edges + view revisions)
  - BpmnProcessDefinition (+ versions + links)
  - Definition Registry entries (ProcessDefinition / AgentDefinition / ExtensionDefinition)

## 5) Core intent catalog (to define)

Core intents are imperative requests to change Enterprise Repository state. Transported on `transformation.domain.intents.v1` as `TransformationDomainIntent` with `meta + subject + oneof intent`.

### Initiative / repository lifecycle

- `CreateTransformationInitiativeRequested`
- `UpdateTransformationInitiativeRequested`
- `CreateWorkspaceNodeRequested`
- `MoveWorkspaceNodeRequested`
- `RenameWorkspaceNodeRequested`
- `ReorderWorkspaceNodeChildrenRequested`

### Artifact lifecycle (generic)

- `CreateArtifactRequested`
- `ReviseArtifactRequested`
- `AttachArtifactBlobRequested`
- `TagArtifactRequested` / `UntagArtifactRequested`
- `DeprecateArtifactRequested` / `ArchiveArtifactRequested`

### Baseline / promotion lifecycle

- `CreateBaselineDraftRequested`
- `AddArtifactRevisionToBaselineRequested`
- `RemoveArtifactRevisionFromBaselineRequested`
- `SubmitBaselineForApprovalRequested`
- `ApproveBaselineRequested` / `RejectBaselineRequested`
- `PromoteBaselineRequested`
- `RollbackBaselineRequested`

### ArchiMate intents (typed; no file-as-canonical)

- `CreateArchiMateModelRequested`
- `UpsertArchiMateElementRequested`
- `UpsertArchiMateRelationshipRequested`
- `DeleteArchiMateElementRequested`
- `DeleteArchiMateRelationshipRequested`

### ArchiMate view/layout intents

- `CreateArchiMateViewRequested`
- `UpdateArchiMateViewPropertiesRequested`
- `UpsertArchiMateViewNodeRequested`
- `UpsertArchiMateViewEdgeRequested`
- `DeleteArchiMateViewNodeRequested`
- `DeleteArchiMateViewEdgeRequested`
- `SnapshotArchiMateViewRevisionRequested`

### BPMN intents (design-time ProcessDefinitions)

- `CreateBpmnProcessDefinitionRequested`
- `CreateBpmnProcessVersionRequested`
- `LinkBpmnProcessToArtifactsRequested`
- `DeprecateBpmnProcessDefinitionRequested` / `ArchiveBpmnProcessDefinitionRequested`

### Definition Registry intents

- `UpsertProcessDefinitionRequested`
- `UpsertAgentDefinitionRequested`
- `UpsertExtensionDefinitionRequested`
- `PromoteDefinitionRequested`

## 6) Core fact catalog (to define)

Core facts are immutable past-tense events transported on `transformation.domain.facts.v1` as `TransformationDomainFact` with `meta + subject + oneof fact`.

### Initiative / repository facts

- `TransformationInitiativeCreated`
- `TransformationInitiativeUpdated`
- `WorkspaceNodeCreated`
- `WorkspaceNodeMoved`
- `WorkspaceNodeRenamed`
- `WorkspaceNodeChildrenReordered`

### Artifact facts

- `ArtifactCreated`
- `ArtifactRevised`
- `ArtifactBlobAttached`
- `ArtifactTagged` / `ArtifactUntagged`
- `ArtifactDeprecated` / `ArtifactArchived`

### Baseline / promotion facts

- `BaselineDraftCreated`
- `BaselineDraftUpdated`
- `BaselineSubmittedForApproval`
- `BaselineApproved` / `BaselineRejected`
- `BaselinePromoted`
- `BaselineRollbackCreated`

### ArchiMate model facts

- `ArchiMateModelCreated`
- `ArchiMateElementUpserted`
- `ArchiMateRelationshipUpserted`
- `ArchiMateElementDeleted`
- `ArchiMateRelationshipDeleted`

### ArchiMate view/layout facts

- `ArchiMateViewCreated`
- `ArchiMateViewPropertiesUpdated`
- `ArchiMateViewNodeUpserted`
- `ArchiMateViewEdgeUpserted`
- `ArchiMateViewNodeDeleted`
- `ArchiMateViewEdgeDeleted`
- `ArchiMateViewRevisionSnapshotted`

### BPMN facts

- `BpmnProcessDefinitionCreated`
- `BpmnProcessVersionCreated`
- `BpmnProcessArtifactLinksUpdated`
- `BpmnProcessDefinitionDeprecated` / `BpmnProcessDefinitionArchived`

### Definition Registry facts

- `ProcessDefinitionUpserted`
- `AgentDefinitionUpserted`
- `ExtensionDefinitionUpserted`
- `DefinitionPromoted`

## 7) Acceptance criteria

1. Canonical proto files exist under `packages/ameide_core_proto/src/ameide_core_proto/transformation/**`.
2. Topic families follow the “aggregator envelope” pattern (intents/facts/process facts).
3. Envelopes include required tenant + traceability metadata and monotonic versions for idempotency.
4. No backlog docs embed full proto text (file index only).

