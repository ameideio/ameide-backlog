# Artifact, Element & Initiative Entities

## Artifact
### Fields
- `artifact_id` (UUID): canonical identifier.
- `graph_id` (FK): owning graph.
- `name` / `description`: human-readable metadata.
- `artifact_subtype` (enum): requirement, standard, reference_block, phase_deliverable, etc.
- `visibility` (enum): inherited default from graph (`default_artifact_visibility`) with overrides per artifact (public, restricted, confidential).
- `sensitivity_label` (enum): business impact tagging used for access decisions and reporting.
- `originating_phase` (enum): TOGAF ADM phase.
- `owning_unit` (FK): organizational unit responsible for the artifact.
- `continuum_attributes` (jsonb): Architecture/Solutions Continuum tags.
- `tags` (array<string>): faceted search aids.
- `governance_owner_group_id` (FK): default governance steward for artifact lifecycle operations.
- `retention_state` (enum): active, retention_hold, purge_scheduled.
- `retention_expires_at` (timestamp): when retention hold may be lifted or auto-purge executed.
- `created_at` / `updated_at`: audit trail.
### Relationships
- Has many `ArtifactVersion` records.
- Linked to `TraceLink` edges as source/target nodes.
- Referenced by `InitiativeReference` when consumed by transformations.
- Inherits default visibility and SLA expectations from `Repository` configuration; overrides recorded in `ArtifactVisibilityGrant`.
- Associated with `GovernanceOwnerGroup` via `governance_owner_group_id` for lifecycle routing.
### Lifecycle
- **Registered**: minimal metadata captured.
- **Enriched**: subtype-specific fields and ownership completed.
- **Deprecated**: no longer actively versioned, awaiting archival.

### Operational Hooks
- Default visibility, retention, and governance stewards hydrate from graph settings; overrides emit `ArtifactRetentionEvent` entries.
- Auto-classification suggestions leverage graph `auto_classification_mode`; accepted suggestions create pending `ClassificationAssignment` drafts.
- Artifact-level sync metadata (search index hash, knowledge graph node id) is derived from latest approved `ArtifactVersion`.

### Subtypes

#### Requirement
##### Fields
- `requirement_type` (enum): business, system, constraint, assumption.
- `priority` (enum/int): MoSCoW or numeric ranking.
- `source_reference` (string/FK): origin artifact or stakeholder.
- `verification_method` (enum): inspection, analysis, demonstration, test.
##### Relationships
- Trace-linked to `PhaseDeliverableArtifact` via trace type `satisfies` (rendered as `satisfied_by` when viewing the requirement) and to `Initiative` via `implemented_by`.
- In scope of `PlacementRule` enforcing minimum lifecycle states.
##### Lifecycle
- Follows artifact lifecycle with additional checkpoints for verification closure.

#### Standard
##### Fields
- `owning_authority` (FK): standards body or governance board.
- `compliance_category` (enum): technology, security, process.
- `enforcement_level` (enum): mandatory, recommended, deprecated.
- `expiration_date` (date): planned review/recertification.
##### Relationships
- Referenced by transformations through `InitiativeReference` (mandated_standard).
- Linked to `RequiredCheck` ensuring compliance evidence.
##### Lifecycle
- Draft → In Review → Approved → Retired, with scheduled recertification aligned to `expiration_date`.

#### Reference Block
##### Fields
- `applicability_scope` (enum): enterprise, segment, capability.
- `maturity_level` (enum): draft, candidate, approved, legacy.
- `dependency_graph` (jsonb): component dependencies / prerequisites.
##### Relationships
- Reused by `PhaseDeliverableArtifact` via TraceLink (`reuses`, `builds_on`).
- May bundle into `ReleasePackage`.
##### Lifecycle
- Evolves through draft/proposed/approved; legacy blocks remain accessible for audit.

#### Phase Deliverable
##### Fields
- `adm_phase` (enum): Preliminary through H.
- `deliverable_type` (enum): Architecture Vision, Roadmap, Migration Plan, etc.
- `transformation_id` (FK): originating transformation workspace.
- `format_metadata` (jsonb): document URLs, model references.
##### Relationships
- Publishes `ArtifactVersion` snapshots that enter graph governance.
- Linked to `InitiativeMilestone` readiness criteria.
##### Lifecycle
- Drafted in transformation, promoted to approved artifact version upon governance completion.

#### Governance Record
##### Fields
- `record_type` (enum): waiver, contract, compliance assessment, decision minutes.
- `decision_context` (string): meeting or board reference.
- `resolution` (enum): approved, rejected, conditional.
##### Relationships
- Opens or closes `ReviewCase` and attaches to `ApprovalDecision`.
- Trace-linked to impacted artifacts/transformations for audit.
##### Lifecycle
- Captured during governance actions; immutable once published.

#### Decision Record
##### Fields
- `decision_scope` (enum): tactical, strategic, architectural.
- `options_considered` (jsonb): alternatives with scoring.
- `rationale` (text): justification and trade-offs.
##### Relationships
- Linked to `TraceLink` edges (impacts/derives_from) and `InitiativeReference`.
- May produce new `RequiredCheck` obligations.
##### Lifecycle
- Drafted concurrently with review; finalized when decision ratified.

#### Model
##### Fields
- `model_language` (enum): ArchiMate, BPMN, UML, DMN, etc.
- `viewpoint` (string): e.g., Motivation, Application Usage.
- `storage_uri` (string): graph location of the model artifact.
- `render_metadata` (jsonb): thumbnails, diagram dimensions.
##### Relationships
- Linked to `TraceLink` (impacts/derives_from) and `ClassificationAssignment`.
- Referenced by transformations for scenario toggles.
##### Lifecycle
- Versioned through `ArtifactVersion`; re-render triggers new versions.

#### Template / Method Artifact
##### Fields
- `method_context` (enum): capability, governance, delivery.
- `applicable_roles` (array<string>): who uses the template.
- `artifact_uri` (string): document storage path.
##### Relationships
- Shared across transformations via `InitiativeReference` (reusable_pattern).
- May be cited by `GovernancePolicy`.
##### Lifecycle
- Updated as methodologies evolve; older versions retired but accessible.

#### Landscape Definition
##### Fields
- `landscape_role` (enum): baseline, target, transition.
- `scope` (string): system, capability, business unit.
- `time_horizon` (date_range): effective planning window.
##### Relationships
- Feeds `ArchitectureState` aggregates.
- Referenced in scenario toggles by transformations.
##### Lifecycle
- Aligns with planning cycles; transitions superseded or merged as plans evolve.

#### Evidence Artifact
##### Fields
- `evidence_type` (enum): audit_report, test_result, risk_assessment.
- `submitted_by` (FK): contributor providing evidence.
- `validity_window` (date_range): period evidence is considered current.
- `content_hash` (string): cryptographic hash of the evidence payload.
- `signature` (text, optional): detached signature or certificate reference.
- `signed_by` (FK, optional): principal who attested to the evidence.
- `signed_at` (timestamp, optional): time of attestation.
##### Relationships
- Attached to `RequiredCheck` runs and `GovernanceRecord`.
- Supports `ReleasePackage` readiness.
##### Lifecycle
- Issued → Reviewed → Accepted/Rejected; archived after expiration. Expired evidence triggers RequiredChecks to fail until refreshed.

#### Release Package
##### Fields
- `release_version` (string): semantic tag (e.g., v1.2).
- `release_notes` (text): summary of scope and decisions.
- `artifact_bundle_uri` (string): location of packaged artifacts.
- `readiness_status` (enum): planned, candidate, approved, released.
##### Relationships
- Aggregates approved `ArtifactVersion` entries and `Evidence Artifact`.
- Consumed by transformations for Go/No-Go reporting.
##### Lifecycle
- Planned → Candidate → Approved → Released → Retired.

## ArtifactVersion
### Fields
- `artifact_version_id` (UUID): primary key.
- `artifact_id` (FK): parent artifact.
- `version_number` (string/int): semantic version.
- `lifecycle_state` (enum): draft, in_review, rework, approved, superseded, retired.
- `visibility` (enum): public, restricted, confidential; defaults to artifact visibility but can tighten for specific versions.
- `governance_status` (enum): clean, pending_checks, blocked, waived.
- `effective_from` / `effective_to`: time-aware modeling windows.
- `scenario_tag` (enum): baseline, to_be, contingency, scenario label.
- `release_package_id` (FK, optional): associated release.
- `submitted_for_review_at` (timestamp, optional): most recent governance submission time.
- `submitted_by_id` (FK, optional): principal that initiated the current review cycle.
- `approved_at` (timestamp, optional): when the version achieved approved resolution.
- `approved_by_id` (FK, optional): approver recorded when governance quorum closed.
- `provenance` (jsonb): transformation, authoring tool, dataset references.
- `checksum` (string): integrity validation.
- `search_index_hash` (string): fingerprint used by search pipelines for incremental updates.
- `knowledge_graph_node_id` (string): unique node reference in graph projections.
- `sync_cursor` (string): high-water mark consumed by RepositorySyncJobs.
- `archived_at` (timestamp, optional): captured when retention policy removes the version from active queries.
### Relationships
- Drives `ClassificationAssignment`, `TraceLink`, `RequiredCheck` runs.
- Referenced by `InitiativeReference` when consumed.
- Tracked through `LifecycleEvent` history.
- Publishes index payloads to `RepositorySyncJob` executions through `ArtifactSyncPayload`.
- Emits `ArtifactRetentionEvent` entries when archived or restored.
- Links to `ReviewCase` records for submission/approval provenance via `submitted_*` and `approved_*` fields so audit trails can attribute governance decisions to specific principals.
- Visibility inherits from the parent artifact unless tightened; `confidential` versions require the artifact and any ClassificationAssignments to honour matching sensitivity rules.
### Lifecycle
- Inherits artifact lifecycle states; progression managed via governance workflows. `rework` captures change-request loops prior to resubmission, while `superseded` records versions replaced by newer approvals but retained for audit.
- Indexers update `search_index_hash`, `knowledge_graph_node_id`, and `sync_cursor` after successful `RepositorySyncJob` completion.
- Archival workflows move `archived_at` populated versions to cold storage per graph retention policy.
- Review submissions populate `submitted_for_review_at`/`submitted_by_id`; successful approvals stamp `approved_at`/`approved_by_id`, enabling downstream telemetry to reconcile cycle times without querying review queues.

## ClassificationAssignment
### Fields
- `assignment_id` (UUID): primary key.
- `artifact_version_id` (FK): associated version.
- `classification_id` (FK): graph classification value.
- `role` (enum): baseline, target, transition, standard, reference, requirement, governance.
- `scope` (enum): enterprise, segment, capability.
- `visibility` (enum): public, restricted, confidential (confidential entries require matching artifact sensitivity).
- `published_at` (timestamp): when assignment became effective.
- `placement_source` (enum): manual, auto_suggested, auto_enforced, system_migrated.
- `enforced_by_rule_id` (FK, optional): corresponding `PlacementRule`.
- `governance_violation_state` (enum): clean, warning, blocked.
- `last_validated_at` (timestamp): last time governance checks assessed the assignment.
- `last_validated_by` (FK, optional): principal or automation that performed validation.
### Relationships
- Bridges ArtifactVersion with RepositoryClassification.
- Constrained by `PlacementRule` and `GovernancePolicy`.
- Confidential assignments require the parent ArtifactVersion to hold `sensitivity_label=confidential`; compliance jobs validate that visibility never widens beyond graph allowances.
- Target for scenario toggles and transformation queries.
- Referenced by `CheckBinding` to determine required checks.
- Indexed by `RepositorySyncJob` for downstream sync payloads.
### Lifecycle
- Created upon classification; updated when scope/role shifts; retired with ArtifactVersion.
- Validation cycle triggers when PlacementRules or RequiredChecks change; violation states updated via governance jobs.

## ArchitectureState
### Fields
- `architecture_state_id` (UUID): computed view identifier.
- `scope` (string): domain/capability.
- `state_role` (enum): baseline, target, transition.
- `effective_window` (date_range): aggregated from contributing assignments.
- `composition_snapshot` (jsonb): list of contributing assignments/versions.
- `source_graph_id` (UUID): graph context for the aggregation.
- `generated_by_job_id` (FK, optional): originating analytics or sync job.
- `anomaly_flags` (jsonb): deviations detected during aggregation (e.g., missing baseline).
### Relationships
- Derived from `ClassificationAssignment`.
- Referenced by `TraceLink` (impacts, realises).
- Published through `RepositorySyncJob` to analytics feeds and knowledge graph views.
### Lifecycle
- Regenerated on demand or scheduled; versioned for audit.

## ArchitectureStateMaterialized
### Fields
- `materialized_state_id` (UUID): primary key.
- `architecture_state_id` (FK): logical state this snapshot represents.
- `state_hash` (string): hash of composition snapshot for tamper detection.
- `computed_at` (timestamp): when snapshot produced.
- `refresh_reason` (enum): scheduled, change_cascade, manual, anomaly.
- `diff_from_prior` (jsonb): summary of adds/removes/changes compared to previous snapshot.
- `generated_by_job_id` (FK, optional): originating analytics job.
### Relationships
- Provides fast lookup for analytics, dashboards, and APIs requiring stable state representations.
- Referenced by change detection tooling and Portfolio/Telemetry views to highlight capability shifts.
### Lifecycle
- **Active** snapshots retained per retention policy; older snapshots archived but queryable for historical comparisons.

## RepositoryElement
### Fields
- `element_id` (UUID): canonical identifier.
- `graph_id` (FK): owning graph.
- `element_type` (enum): ontology concept (e.g., archimate_capability, archimate_business_actor, bpmn_process, dcat_dataset).
- `ontology_source` (enum): archimate, bpmn, dmn, iso27001, custom_extension.
- `ontology_version` (string): semantic version of the source ontology catalogue.
- `name` / `description`: human-readable metadata.
- `lifecycle_state` (enum): draft, approved, deprecated, retired.
- `provenance` (jsonb): tool/workspace, author, import reference.
- `governance_owner_group_id` (FK, optional): default steward when element changes require review.
- `created_at` / `updated_at`: audit timestamps.
### Relationships
- Belongs to `Repository`; participates in `RepositoryElementRelationship` edges.
- Linked to `OntologyAssociation` for cross-ontology mappings.
- Referenced by `ElementAttachment` when artifacts annotate or discuss the element.
- Projects into `ElementProjection` viewpoints for diagram and impact analysis.
### Lifecycle
- **Draft**: created via modeling tool import or manual registration.
- **Approved**: validated against ontology constraints and governance rules.
- **Deprecated**: superseded but still referenced; raises alerts for dependent artifacts.
- **Retired**: removed from active use; retained for historical traceability.

## RepositoryElementRelationship
### Fields
- `relationship_id` (UUID)
- `graph_id` (FK)
- `source_element_id` / `target_element_id` (FK)
- `relationship_type` (enum): ontology-specific relation (serving, realization, specialization, association, sequence_flow, etc.).
- `ontology_source` (enum): archimate, bpmn, etc.
- `cardinality` (enum): one_to_one, one_to_many, many_to_many; derived from ontology constraints.
- `valid_from` / `valid_to` (timestamp, optional): effective window.
- `confidence` (decimal, optional): support for probabilistic associations when imported from analytics.
- `created_at` / `updated_at`
### Relationships
- Connects RepositoryElements per ontology graph.
- Referenced by `ElementProjection` and knowledge graph exports.
- Cardinality validations enforced against `OntologyCatalogue` relationship constraints to prevent invalid multiplicities.
- May trigger governance workflows when breaking constraints (e.g., invalid ArchiMate relationship).
### Lifecycle
- Created via modeling tooling or through governance-approved updates.
- Updated when ontology rules change; retired when elements are removed.

## OntologyAssociation
### Fields
- `association_id` (UUID)
- `graph_id` (FK)
- `source_element_id` / `target_element_id` (FK)
- `association_type` (enum): equivalent_to, refines, realises, constrained_by, aligns_with.
- `justification` (text): reason for association (standard mapping, manual decision, imported catalog).
- `authority_reference` (string): external standard or decision record backing the association.
- `governance_status` (enum): proposed, approved, deprecated.
- `created_at` / `updated_at`
### Relationships
- Links elements across ontologies and documents lineage to decision records via `TraceLink`.
- Referenced by `ElementProjection` and analytics to create integrated views.
- Surfaces to governance dashboards for ontology gap management.
### Lifecycle
- **Proposed**: association suggested by modeling tools or analysis.
- **Approved**: validated by governance group.
- **Deprecated**: replaced or invalidated; tracked for audit.

## ElementAttachment
### Fields
- `attachment_id` (UUID)
- `artifact_version_id` (FK) or `artifact_id` (FK)
- `element_id` (FK)
- `annotation_type` (enum): mentions, elaborates, constrains, documents, evidence_for.
- `context_location` (jsonb): fragment pointers into the artifact (page, heading, paragraph, diagram reference).
- `created_at` / `updated_at`
### Relationships
- Attaches RepositoryArtifacts/ArtifactVersions to RepositoryElements.
- Enables governance rules ensuring elements referenced in narratives align with latest approved definitions.
- Supports tooling that highlights elements directly within documents or diagrams.
- Allows strategy/change entities (e.g., `CommunicationPlan`, `TrainingPackage`) to cite specific `RepositoryElement` impacts without duplicating ontology data.
### Lifecycle
- Created during authoring/editing workflows; updated when artifact versions change; soft-deleted when artifact or element retires.

## ElementProjection
### Fields
- `projection_id` (UUID)
- `graph_id` (FK)
- `name` / `description`
- `projection_type` (enum): capability_map, value_stream, application_portfolio, data_flow, risk_view, custom.
- `element_ids` (array<UUID>): ordered list of included elements.
- `filters` (jsonb): derived rules for dynamic projections (e.g., include all capabilities in segment X).
- `render_metadata` (jsonb): layout hints, viewpoint, diagram coordinates.
- `generated_from_artifact_version_id` (FK, optional): source document or model if projection is generated.
- `created_at` / `updated_at`
### Relationships
- Builds on RepositoryElements and relationships; may generate RepositoryArtifacts (diagrams) for communication.
- Referenced by transformations and governance workflows to understand scope/impact.
- Exported through `RepositorySyncJob` to knowledge graph or visualization tooling.
### Lifecycle
- Generated or curated; versioned as underlying elements or layout changes; archived when superseded.

## TraceLink
### Fields
- `trace_link_id` (UUID): primary key.
- `source_artifact_version_id` (FK): origin of the link.
- `target_artifact_version_id` (FK): destination of the link.
- `trace_type` (enum): satisfies, implements, reuses, builds_on, duplicates, supersedes, constrains, depends_on, impacts, realises.
- `weight` (decimal): optional strength weighting between 0 and 1.
- `confidence` (decimal): optional confidence score between 0 and 1.
- `valid_from` / `valid_to` (timestamp, optional): window of applicability.
- `rationale` (text): explanation or citation backing the link.
- `created_at` / `updated_at` (timestamp).
### Relationships
- Connects ArtifactVersions (and, via attachments, RepositoryElements) for impact analysis, coverage reporting, and policies.
- Referenced by `TracePolicy`, Release readiness, and dependency overlays in transformations.
- TraceLinks always store the forward intent (e.g., Deliverable `satisfies` Requirement); inverse labels such as `satisfied_by` are derived at presentation time so reporting remains consistent across domains.
### Lifecycle
- Created during authoring or import; updated when rationale or scope changes; archived when superseded by new versions.

## TracePolicy
### Fields
- `trace_policy_id` (UUID): primary key.
- `graph_id` (FK)
- `scope_type` / `scope_id`: classification, artifact subtype, transformation, control, etc.
- `required_trace_type` (enum): link type that must exist (e.g., satisfies, implements).
- `minimum_links` (int): minimum number of links required.
- `enforcement_level` (enum): advisory, warning, blocking.
- `created_at` / `updated_at` (timestamp).
### Relationships
- Evaluated by RequiredChecks; violations produce governance alerts and block release when enforcement is blocking.
- Displayed in dashboards to show traceability coverage and gaps.
### Lifecycle
- Draft → Active → Suspended → Retired, with historical policies retained for audit.

## Visibility & Retention

### ArtifactVisibilityGrant
#### Fields
- `visibility_grant_id` (UUID)
- `artifact_id` (FK)
- `principal_id` (FK)
- `granted_scope` (enum): read, contribute, govern.
- `reason` (enum): default_graph, governance_role, explicit_share, policy_exception.
- `granted_at` / `expires_at` (timestamp)
- `granted_by` (FK)
#### Relationships
- Extends graph-level access governed by `RepositoryAccessRole`.
- Surfaces to governance dashboards for segregation-of-duties validation.
#### Lifecycle
- Auto-provisioned from graph defaults; manually extended for exceptions; revoked or expired per retention rules.

### ArtifactRetentionEvent
#### Fields
- `retention_event_id` (UUID)
- `artifact_id` / `artifact_version_id` (FK)
- `event_type` (enum): hold_applied, hold_released, purge_scheduled, purge_executed, restore_requested.
- `trigger_source` (enum): graph_policy, governance_policy, legal_hold, manual_override.
- `triggered_by` (FK)
- `event_payload` (jsonb): context such as legal case reference or policy id.
- `occurred_at` (timestamp)
#### Relationships
- References graph retention policy entries and governance policies.
- Drives notifications to governance owner groups and compliance teams.
#### Lifecycle
- Logged for every retention state change; immutable for audit fidelity.

## Synchronisation & Analytics

### ArtifactSyncPayload
#### Fields
- `payload_id` (UUID)
- `artifact_version_id` (FK)
- `sync_job_id` (FK)
- `target` (enum): knowledge_graph, search_index, analytics_snapshot, archive_snapshot.
- `payload_hash` (string)
- `payload_metadata` (jsonb): counts, languages, redaction flags.
- `dispatched_at` (timestamp)
#### Relationships
- Joins `RepositorySyncJob` with `ArtifactVersion` to provide replay ability and audit trail.
- Referenced by knowledge-graph and search services for troubleshooting.
#### Lifecycle
- Created during sync job execution; archived when retention window elapses.

### ArtifactSearchDocument
#### Fields
- `document_id` (UUID)
- `artifact_version_id` (FK)
- `language` (string)
- `tokenized_body` (tsvector/jsonb)
- `facets` (jsonb): classification, subtype, risk, owning unit, scenario tags.
- `boost_score` (decimal): ranking weight derived from governance signals.
- `indexed_at` (timestamp)
#### Relationships
- Produced from `ArtifactVersion` via `ArtifactSyncPayload` targeting search index.
- Consumed by search APIs and discovery dashboards.
#### Lifecycle
- Regenerated on artifact updates, classification changes, or governance events; removed when ArtifactVersion archived.

### ProcessMiningDataset
#### Fields
- `dataset_id` (UUID)
- `graph_id` (FK)
- `element_id` (FK): `RepositoryElement` (e.g., archimate_application_component) the telemetry describes.
- `source_system` (string): mining tool or pipeline identifier.
- `captured_range` (daterange): time window of the observation.
- `metrics` (jsonb): aggregated KPIs (throughput, conformance score, variant counts, etc.).
- `raw_artifact_uri` (string, optional): pointer to detailed export for audit.
- `ingested_at` (timestamp)
- `anomaly_flags` (jsonb, optional): deviations detected by analytics.
#### Relationships
- Complements RepositoryElements with operational telemetry without duplicating element semantics.
- Referenced by `OutcomeMetricSnapshot`, dashboards, and narrative artifacts (via `ElementAttachment`) that interpret the mining insights.
#### Lifecycle
- Ingested on schedule or event; superseded datasets retained per telemetry retention policy for trend analysis.


## Initiatives & Related Entities

### Initiative
#### Fields
- `transformation_id` (UUID): primary key.
- `graph_id` (FK): linked graph.
- `name` / `description`: planning metadata.
- `stage` (enum): Stage 1–5 (or ADM phase alignment).
- `health_status` (enum): green, amber, red.
- `cadence` (enum): ADM, agile, hybrid.
- `sponsor_id`, `product_owner_id`, `lead_architect_id` (FKs): key roles.
- `created_at` / `updated_at`.
- `time_zone` (string): defaults from graph but overridable for local execution rhythm.
- `locale` (string): inherits graph default locale for communications and templates.
#### Relationships
- Owns `InitiativeMilestone`, `InitiativeRoleAssignment`, `InitiativeMetricSnapshot`, and aligns to strategy/change constructs (`StrategicThemeAssignment`, `BenefitHypothesis`, `ChangeManagementProfile` defaults).
- Surfaces `InitiativeAlert` signals to highlight upstream governance or policy events that require transformation attention.
- Consumes artifacts via `InitiativeReference`.
- Feeds governance telemetry through blockers and readiness reporting.
- Mirrors graph default SLA expectations for review cycles; breaches emit `SlaBreachEvent`.
#### Lifecycle
- Proposed → Active → Closing → Completed → Archived.

### InitiativeMilestone
#### Fields
- `milestone_id` (UUID)
- `transformation_id` (FK)
- `milestone_type` (enum): discovery, design, build, launch, etc.
- `target_date` / `actual_date`
- `readiness_criteria` (jsonb)
- `status` (enum): planned, in_progress, achieved, slipped.
#### Relationships
- Linked to `PhaseDeliverableArtifact` (scope) and `ReleasePackage`.
- Surfaced on portfolio dashboards with `InitiativeMetricSnapshot`.
#### Lifecycle
- Planned → In Progress → Achieved or Slipped → Closed.

### InitiativeReference
#### Fields
- `reference_id` (UUID)
- `transformation_id` (FK)
- `artifact_version_id` (FK)
- `reference_type` (enum): baseline_reference, mandated_standard, reusable_pattern, dependency.
- `context` (jsonb): phase, sprint, release, rationale.
- `evidence_links` (array<string>)
#### Relationships
- Binds transformations to consumed artifacts.
- Drives blockers, readiness, and dependency overlays.
#### Lifecycle
- Created during planning; updated as scope changes; archived when transformation closes.

### InitiativeRole
#### Fields
- `role_id` (UUID)
- `name` (enum/string): sponsor, program_manager, product_owner, lead_architect, reviewer, contributor, ai_reviewer.
- `governance_responsibilities` (jsonb)
- `notification_defaults` (jsonb)
#### Relationships
- Referenced by `InitiativeRoleAssignment`.
- Mapped to directory roles or groups.
#### Lifecycle
- Managed centrally; evolves as methodology updates.

### InitiativeRoleAssignment
#### Fields
- `assignment_id` (UUID)
- `transformation_id` (FK)
- `role_id` (FK)
- `principal_id` (FK): person or service principal.
- `assignment_state` (enum): invited, accepted, declined, revoked.
- `start_date` / `end_date`
- `directory_sync_source` (string)
#### Relationships
- Feeds blockers funnel ownership and readiness reports.
- Linked to `GovernanceOwnerGroup` when roles overlap.
#### Lifecycle
- Awaiting Acceptance → Active → Reassigned/Ended.

### InitiativeMetricSnapshot
#### Fields
- `snapshot_id` (UUID)
- `transformation_id` (FK)
- `captured_at` (timestamp)
- `board_metrics` (jsonb): blockers outstanding, approvals pending, checks failing.
- `readiness_metrics` (jsonb): release completeness, evidence coverage.
- `scenario_metrics` (jsonb): toggle latency, impacted systems.
- `graph_statistic_snapshot_id` (FK, optional): synchronized graph KPI sample.
- `sla_state` (enum): on_track, warning, breach.
- `anomaly_flags` (jsonb): SLA breaches, capacity warnings, governance gaps.
#### Relationships
- Paired with `GovernanceMetricSnapshot` and `OutcomeMetricSnapshot` for portfolio reporting.
- Consumed by dashboards and analytics.
- Linked to `RepositoryStatisticSnapshot` for combined reporting and alerting.
- Provides inputs to funding guardrail and adoption assessments alongside `AdoptionSignal`.
#### Lifecycle
- Generated on schedule or event-driven; retained for trend analysis.

### InitiativeAlert
#### Fields
- `transformation_alert_id` (UUID)
- `transformation_id` (FK)
- `source_entity_type` / `source_entity_id`: policy, artifact_version, required_check, survey, etc.
- `alert_type` (enum): policy_change, recertification, compliance_violation, sla_breach, telemetry_anomaly.
- `severity` (enum): info, warning, critical.
- `message` (text): human-readable summary surfaced to transformation stakeholders.
- `triggered_at` (timestamp)
- `acknowledged_at` (timestamp, optional)
- `resolved_at` (timestamp, optional)
- `resolution_notes` (jsonb, optional): mitigation plan or linkage to corrective work items.
#### Relationships
- Raised by governance automation (e.g., `RecertificationJob`, `CheckRun`, `SlaBreachEvent`) to notify transformations consuming impacted artifacts.
- Referenced by `InitiativeMetricSnapshot` and blockers funnel dashboards to keep readiness scores aligned with outstanding risks.
- Syncs to `GovernanceNotification` to route alert acknowledgements and proof of response.
#### Lifecycle
- Created when upstream governance or telemetry events impact transformation scope → Acknowledged by transformation owners → Resolved once mitigation completes; retained for portfolio audits and outcome analysis.
