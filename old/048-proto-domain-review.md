# 048 – Core Proto Package Review

Date: 2025-07-25

## Context

As part of the ongoing migration to a protobuf-first API contract, the **`packages/ameide_core_proto`** module now contains all platform-wide message and service definitions.  A light architectural and code review was carried out to assess current strengths and identify outstanding gaps before the package is promoted to **v1 GA** status.

## Highlights / What works well

* **Clear layout** – standard Bazel structure (`BUILD.bazel`, `proto/…/v1`) with Go/Java package options already set.
* **Well-known types reused** (`Timestamp`, `Struct`, `FieldMask`, `Empty`), preventing custom re-implementation of common concepts.
* **Behaviour annotations** in `ameide.iomon.v1.behavior` capture validation semantics (required, immutable, output-only) – good basis for code-gen.
* **Enumerations** (`MessageRole`, `RunStatus`) include `_UNSPECIFIED` zero value for forward compatibility.
* **Metadata maps** allow future, schema-less extensions without breaking wire format.

## Issues / Gaps

### Original Review Findings

1. **Inconsistent status typing** – `AgentExecution.status` is a `string` even though `RunStatus` enum exists.
2. **BUILD targets** – need shared `proto_library` plus per-language libraries to speed up build graph.
3. **Cross-resource references** – IDs are plain strings; resource-name patterns (`users/{user}`) would help IAM & routing.
4. **Field numbering gaps** – several messages have unused tag numbers; can still be compacted pre-GA.
5. **README** – describe how `dist/` is generated & switch checklist to GitHub task list.
6. **Validation plugin docs/tests** – add example `protoc` invocation and minimal pytest round-trip tests.
7. **Stability signalling** – experimental RPCs should live in `v1alpha` or carry `Experimental` prefix.
8. **Buf Lint/Breaking-change guard** – add `buf.yaml` + CI step.

### Additional Concerns (2025-07-25)

9. **Duplicate Type Definitions** – `NodeType` and `EdgeType` enums are defined in both `workflows.proto` and `orchestration.proto` with different values, creating confusion and potential runtime errors.
10. **Missing Service-Level Annotations** – Services lack annotations for authentication, rate limiting, and SLA requirements despite having the annotation framework.
11. **Inconsistent Error Handling** – Some services return structured errors, others use string fields; no consistent error code taxonomy.
12. **Missing Batch Operations** – Most services only support single-entity CRUD despite having `BatchRequest/Response` in common types.
13. **Streaming API Gaps** – Only Agent and Orchestration services have streaming endpoints; Workflow executions could benefit from streaming.
14. **Field Validation Patterns** – `FieldBehavior.validation_regex` exists but lacks usage examples for common patterns (email, URL, UUID).
15. **Circular Dependencies Risk** – IPAs reference Workflows, Agents, and Tools without clear dependency boundaries.
16. **Missing Tenant Isolation** – `ResourceMetadata.tenant_id` exists but not consistently used across all services.
17. **Pagination Inconsistencies** – Mix of cursor-based and offset pagination with no clear documentation.
18. **Version Management** – Multiple version fields (resource, schema, graph) with no clear versioning strategy.
19. **Missing Integration Points** – No clear extension points for custom implementations despite plugin system in annotations.
20. **Performance Considerations** – Large `google.protobuf.Struct` fields without size limits defined.

## Decisions / Actions

### Original Actions

* [ ] Convert `AgentExecution.status` to `RunStatus` enum.
* [ ] Split BUILD file into: `common_proto`, `ameide_core_proto`, `python_proto`, `go_proto`, etc.
* [ ] Introduce resource name patterns for cross-entity IDs (follow AIP-122).
* [ ] Clean up unused field numbers before v1 freeze.
* [ ] Update README – generation instructions & checkbox list.
* [ ] Add `buf.yaml` with lint + breaking-change rules; integrate into CI.
* [ ] Author minimal schema tests (`packages/ameide_core_proto/tests`).

### Additional Actions

* [ ] Create shared `common/v1/graph_types.proto` for reusable graph-related enums (NodeType, EdgeType).
* [ ] Apply service method annotations for auth, rate limiting, and SLA requirements.
* [ ] Define standard error codes enum and implement consistent error handling.
* [ ] Add batch operations (CreateBatch, UpdateBatch, DeleteBatch) to all services.
* [ ] Add streaming support to Workflow service for execution updates.
* [ ] Document field validation patterns and add examples to proto comments.
* [ ] Create dependency hierarchy diagram to prevent circular dependencies.
* [ ] Enforce tenant_id usage in all multi-tenant request contexts.
* [ ] Standardize on cursor-based pagination and document behavior.
* [ ] Document versioning strategy for resources, schemas, and migrations.
* [ ] Define plugin interfaces and extension points for custom implementations.
* [ ] Add field size constraints to `google.protobuf.Struct` fields and consider typed alternatives.

## Architectural Review Update (2025-07-26)

### Critical Issues (GA-blocking) - P0

1. **Duplicate enum definitions** – `NodeType`, `EdgeType` in multiple files
   - Impact: Runtime errors, type confusion
   - Action: Create shared `common/v1/graph_types.proto` immediately

2. **Mixed pagination styles** – Cursor vs offset with no documentation
   - Risk: Inconsistent API behavior
   - Action: Standardize on cursor-based pagination

3. **Inconsistent error shapes** – String vs structured errors
   - Impact: Poor error handling in SDKs
   - Action: Define ErrorCode enum and structured error response

4. **Missing `tenant_id`** – Security vulnerability across many messages
   - Risk: Cross-tenant data access
   - Action: Add tenant_id to all request/response types

5. **No Buf lint/breaking-change guard** – Changes slip through
   - Impact: Breaking changes without version bump
   - Action: Wire Buf CI pipeline immediately (Day 0-2)

### Implementation Gaps

1. **Missing Storage Implementation**
   - GraphService - No dedicated graph
   - OrchestrationService - No storage layer
   - Event/audit trail - No event storage
   - Missing query capabilities (metadata search, pagination)

2. **Multi-tenancy Not Enforced**
   - TenantRepository exists but not integrated
   - No automatic tenant filtering
   - Missing row-level security

3. **No Transaction Management**
   - Version fields unused (no optimistic locking)
   - No soft delete implementation
   - Missing cascade delete configurations

### Immediate Actions (Day 0-2)

- [ ] Fix duplicate enums - create shared graph_types.proto
- [ ] Add Buf lint & breaking-change CI pipeline
- [ ] Define standard error codes enum
- [ ] Add tenant_id to all messages lacking it
- [ ] Standardize on cursor-based pagination

### Week 1 Actions

- [ ] Implement missing storage repositories (Graph, Orchestration)
- [ ] Add transaction boundaries with UnitOfWork pattern
- [ ] Integrate tenant isolation at graph level
- [ ] Add metadata-based search and filtering
- [ ] Implement soft delete and versioning

### Proto Cleanup Tasks

- [ ] Apply service method annotations for auth/rate limiting
- [ ] Document field validation patterns with examples
- [ ] Create dependency hierarchy to prevent circular refs
- [ ] Add field size constraints to Struct fields
- [ ] Define plugin interfaces and extension points

### Versioning Strategy

1. Proto changes require Buf breaking-change check
2. Use directories for version management:
   - `proto/ameide/user/v1/` - Stable API
   - `proto/ameide/user/v1alpha/` - Experimental features
   - `proto/ameide/user/v2/` - Breaking changes
3. Field deprecation with proper notices
4. Migration guides for breaking changes





### 1  ArchiMate Viewpoints ― what they are and why they sit **above** the “primitive layer”

*In the ArchiMate language a **viewpoint** is *not* a model element.
It is a reusable *filter* that tells a modelling tool which element/relationship types may appear on a diagram and which stakeholder concerns that diagram should address.*
The official specification ships **23 example viewpoints**, arranged in four categories (Basic, Motivation, Strategy, Implementation‑&‑Migration). Each defines: stakeholder, concern, allowed element set, and a sample diagram. ([Visual Paradigm][1])

Because a viewpoint is only a *presentation rule*, it belongs in the **service layer** of your platform (configuration in the View‑service or UI schema), **not** in the “primitive” protobuf package. The primitive layer just needs:

```
message Viewpoint {
  string id          = 1;      // e.g. "business_process_cooperation"
  string name        = 2;
  repeated ArchiMateElementType permitted_element_types = 3;
  repeated ArchiMateRelationshipType permitted_rel_types = 4;
  repeated string stakeholder_roles = 5;
}
```

This message can live in `ea/v1/view_meta.proto` and reference the enums already defined in the primitive EA package. No circular dependency is introduced because the enums do not import the message.

---

### 2  Deeper domain research – **base entities** per domain (MECE)

Below are the minimal concepts that standards and literature repeatedly agree on. These are the entities you must version‑freeze; everything else can be derived, aggregated, or streamed later.

| Domain                                       | Base entities (protobuf messages / enums)                                                                                               | Rationale & key references                                                                                                                                                                                                                |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cross‑cutting (COMMON)**                   | `ResourceMetadata`, `ErrorCode` (enum), `FieldBehavior` (enum), `NodeType` & `EdgeType` (enum)                                          | Required for identity, lifecycle, validation, error surfaces, and graph consistency across the platform.                                                                                                                                  |
| **Enterprise Architecture – ArchiMate Core** | `ArchiMateElement`, `ArchiMateRelationship`, `ArchiMateElementType` (enum), `ArchiMateLayer` (enum), `ArchiMateRelationshipType` (enum) | ArchiMate 3.1 defines only **these** as the core vocabulary. Viewpoints, motivations, and documents build on them. ([Visual Paradigm][1])                                                                                                 |
| **Process Automation**                       | `Workflow`, `Agent`, `ExecutionState` (enum)                                                                                            | Workflows & agents are the actors; a single shared state machine keeps status handling uniform.                                                                                                                                           |
| **Process Mining (XES‑aligned)**             | `EventLog`, `ProcessEvent`, `EventLifecycle` (enum)                                                                                     | XES standard reduces event data to *log → trace → event*; case‑id, activity, timestamp, lifecycle are the only universally present attributes. ([XES Standard][2])                                                                        |
| **Project Management (PMI)**                 | `Project`, `ProjectStatus` (enum); `WBSNode`, `WBSType` (enum)                                                                          | PMI defines the WBS as the root artefact for scope; tasks, milestones, risks, etc. all roll up to WBS nodes. ([Project Management Institute][3])                                                                                          |
| **Strategy & Business Architecture**         | `StrategicObjective`, `KPIPerspective` (enum), `Capability`                                                                             | Kaplan & Norton’s Balanced Scorecard uses objectives grouped into four perspectives (Financial, Customer, Internal Process, Learning & Growth). Capability is the TOGAF/Business‑Architecture anchor. ([Balanced Scorecard Institute][4]) |
| **Data Architecture**                        | `DataEntity`, `DataLineageEdge`, `MasterDataDomain`                                                                                     | Industry sources converge on lineage and master‑data as the irreducible building blocks for data governance. ([IBM][5], [Wikipedia][6])                                                                                                   |
| **Technology / Application Portfolio**       | `Application`, `Interface`, `TechnologyComponent`, `DependencyType` (enum)                                                              | Application Portfolio Management (APM) literature treats applications, their interfaces and technical components as the atomic units. ([Wikipedia][7])                                                                                    |
| **Governance & Change**                      | `ArchitectureDecision`, `Standard`, `Waiver`, `ReviewStatus` (enum)                                                                     | TOGAF distinguishes Architecture Building Blocks, standards, and decision records as the kernel of governance. ([www.opengroup.org][8])                                                                                                   |
| **Collaboration & Knowledge**                | `Comment`, `Revision`, `Notification`                                                                                                   | CRUD + audit trail for any artefact; the minimum to enable reviews and approvals.                                                                                                                                                         |
| **Analytics & Reporting**                    | `MetricDefinition`, `MetricValue`, `VisualizationSpec`                                                                                  | Separates raw metric definition/value storage from the rendering contract (dashboards, heat maps, etc.).                                                                                                                                  |
| **Integration & Ecosystem**                  | `WebhookSubscription`, `EventEnvelope`, `ImportJob`, `ExportJob`                                                                        | Enables external systems to push/pull data without touching core primitives.                                                                                                                                                              |

*All groups are mutually exclusive (each concept appears once) yet collectively exhaustive for the platform’s stated scope.*

---

### 3  Suggested next steps

1. **Add `ea/v1/view_meta.proto`** (or similar) for `Viewpoint` plus service stubs; keep it versioned as *beta* so modelling teams can refine permitted‑element lists without breaking GA contracts.
2. **Validate each domain package** against the table above; if a message or enum is not listed, move it to a *v1beta* sub‑package.
3. **Populate your Buf lint rules** so that new messages cannot be added to a GA package without a design‑review label confirming they belong in the primitive set.
4. **Document these base entities** in the README so downstream teams understand why higher‑order artefacts (diagrams, schedules, risk matrices, etc.) live in beta APIs.

This keeps the **primitive layer lean, future‑proof, and domain‑complete**, while giving every feature team a clear path for evolving richer semantics on top.

[1]: https://www.visual-paradigm.com/guide/archimate/full-archimate-viewpoints-guide/ "Full ArchiMate Viewpoints Guide (Examples Included)"
[2]: https://www.xes-standard.org/?utm_source=threadsgpt.com "XES: start"
[3]: https://www.pmi.org/learning/library/work-breakdown-structure-basic-principles-4883?utm_source=threadsgpt.com "Work Breakdown Structure (WBS) - Basic Principles | PMI"
[4]: https://balancedscorecard.org/bsc-basics-overview/?utm_source=threadsgpt.com "Balanced Scorecard Basics"
[5]: https://www.ibm.com/think/topics/data-lineage?utm_source=threadsgpt.com "What Is Data Lineage? | IBM"
[6]: https://en.wikipedia.org/wiki/Master_data_management?utm_source=threadsgpt.com "Master data management - Wikipedia"
[7]: https://en.wikipedia.org/wiki/Application_portfolio_management?utm_source=threadsgpt.com "Application portfolio management"
[8]: https://www.opengroup.org/public/arch/p4/bbs/bbs_intro.htm?utm_source=threadsgpt.com "Introduction to Building Blocks - The Open Group"

