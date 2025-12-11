# Repository Entities

> Progress tracking lives in `220-graph-entity-model-progress.md` (updated 2025-10-20).

## Domain Summary
- Repositories anchor tenant-scoped architecture knowledge bases and determine the governance defaults inherited by transformations, artifacts, and dashboards.
- Each graph carries classification vocabularies, policy bundles, ontology catalogues, and telemetry snapshots that power placement validation, readiness analytics, and SLA monitoring.
- Foundation entities expose sync hooks into knowledge graph, search, archival, and strategy/change subsystems so downstream projections stay consistent with governance decisions and benefit tracking.

## Status Update (2025-10-21)
- ✅ Implemented graph protobuf surfaces for artifacts, transformations, governance roles, and assignments with Go/TS SDK parity.
- ✅ Repository service now persists artifacts, versions, and workspace assignments via typed commands surfaced through Connect handlers.
- ✅ Initiative pages consume the new proto contracts, exposing graph-aware workspaces and artifact listings in the platform UI.
- 🔄 Next steps: enrich `RepositoryAccessRole`/`Membership` projections and wire governance policy enforcement into the assignment pipeline.

## Repository
### Fields
- `graph_id` (UUID): tenant-unique identifier.
- `graph_slug` (string): URL-safe handle used for routing and API lookups.
- `tenant_id` (UUID): owning tenant.
- `name` / `description` (string): surfaced in headers, search, and integration payloads.
- `governance_profile_id` (FK): pointer to the active `GovernanceProfile` bundle.
- `visibility` (enum): public, restricted (legacy `private` aliases to `restricted`).
- `default_artifact_visibility` (enum): default visibility applied to newly registered artifacts (`public` or `restricted`).
- `default_time_zone` (string): scheduling context for SLAs and milestone templating.
- `default_locale` (string): localization baseline for graph-level content.
- `default_review_sla` (duration): governance turnaround expectation surfaced to queues.
- `sla_policy_refs` (array<FK>): linked SLA policies for approvals and queue latency.
- `auto_classification_mode` (enum): manual, suggested, enforced auto-placement behavior.
- `archival_policy` (jsonb): retention plan for retired artifacts and evidence.
- `retention_policy` (jsonb): graph-level data retention and purge configuration.
- `feature_toggles` (jsonb): graph-scoped feature flags and beta enablements.
- `knowledge_graph_sync_state` (enum): idle, scheduled, syncing, failed.
- `search_index_state` (enum): idle, stale, reindexing, failed.
- `statistics_cache` (jsonb): aggregate metrics (approval cycle, queue wait, data quality).
- `last_synced_at` (timestamp): most recent successful downstream synchronization.
 - `created_at` / `updated_at` (timestamp): auditing metadata.
### Relationships
- Owns many `RepositoryClassification`, `PlacementRule`, and `GovernanceOwnerGroup` records.
- Provides canonical source for `Artifact` registrations, `RepositoryElement` catalog entries, `Initiative` links, and graph-scoped `TraceLink` edges.
- Has many `RepositoryAccessRole` and `RepositoryMembership` entries governing access control.
- Linked to `RepositoryIntegrationEndpoint` and `RepositorySyncJob` entities for external system projections.
- Associates with `GovernanceProfile` via `governance_profile_id` to inherit policy bundles.
- Feeds `RepositoryStatisticSnapshot` records consumed by readiness dashboards and analytics.
- Provides defaults for `ChangeManagementProfile` and `ChangePlaybook` assignments defined in the strategy/change model.
### Lifecycle
- **Provisioning**: metadata captured, defaults seeded, awaiting governance profile activation.
- **Active**: fully configured, accepting artifacts, governance workflows, and sync jobs.
- **ReadOnly**: temporarily frozen (e.g., audit hold) but still discoverable.
- **Deprecated**: superseded by another graph; write operations blocked except archival jobs.
- **Archived**: retained for audit; classifications, artifacts, and integrations frozen.
### Operational Hooks
- Provisioning workflows seed default `RepositoryAccessRole`, classifications, and placement templates.
- Sync orchestration publishes graph deltas to knowledge graph, catalog exporters, and search indexers.
- Health monitors evaluate SLA adherence using `statistics_cache` and emit alerts when thresholds breached.

## RepositoryClassification
### Fields
- `classification_id` (UUID): primary key.
- `graph_id` (FK): owning graph.
- `code` / `label` (string): TOGAF partition identifier (e.g., `SIB`, `Landscape.Target`).
- `description` (string): usage guidance exposed in UX.
- `partition_type` (enum): capability, graph, landscape_baseline, landscape_target, governance_log, etc.
- `parent_classification_id` (UUID, FK self): parent classification for hierarchy.
- `hierarchy_path` (ltree/string): hierarchical encoding for nested navigation; maintained alongside parent links.
- `default_owner_group_id` (FK, optional): governance group automatically assigned on placement.
- `risk_tier` (enum): low, medium, high; influences evidence expectations.
- `ui_order` (int): ordering hint for navigation and reporting surfaces.
- `color_token` (string): design system token for consistent visual cues.
- `effective_from` / `effective_to` (timestamp): optional windows for temporary vocab entries.
- `synonyms` (array<string>): alternate labels exposed in search and mapping.
- `superseded_by_classification_id` (UUID, FK self, optional): records deprecation path.
- `metadata` (jsonb): organization-specific extensions (tags, tool integrations).
### Relationships
- Belongs to `Repository`; referenced by `ClassificationAssignment` records.
- Scoped by `PlacementRule` constraints and `RepositoryAccessRole` permissions.
- Tagged to `GovernanceOwnerGroup` for reviewer ownership and escalation.
- Indexed for knowledge graph exports and graph navigation menus.
- Supports hierarchy queries (parent/child traversal). Cycles are disallowed; leaf vs. interior nodes influence placement rules and policies.
### Lifecycle
- **Proposed**: new classifications pending governance approval and owner assignment.
- **Published**: available for artifact classification and surfaced in graph UI.
- **Suspended**: hidden from new placements while under review; existing assignments retained.
- **Retired**: removed from active use but preserved for historical traceability.

## OntologyCatalogue
### Fields
- `catalogue_id` (UUID): primary key.
- `graph_id` (FK): owning graph.
- `ontology_source` (enum): archimate, bpmn, dmn, dcat, iso27001, custom_extension.
- `version` (string): semantic version of the approved ontology package.
- `status` (enum): draft, approved, deprecated.
- `import_reference` (string): pointer to import artifact, tool export, or decision record.
- `governance_owner_group_id` (FK, optional): accountable authority for ontology stewardship.
- `extension_policy` (jsonb): rules for proposing custom types, relationships, or associations.
- `created_at` / `updated_at` (timestamp).
### Relationships
- Validates allowable values for `RepositoryElement.element_type` and `RepositoryElementRelationship.relationship_type`.
- Linked to `OntologyAssociation` entries documenting cross-ontology mappings.
- Referenced by strategy/change constructs requiring specific viewpoints (e.g., capability or value-stream projections).
### Lifecycle
- **Draft**: catalogue assembled and reviewed by governance/change owners.
- **Approved**: exposed to modeling tools and element registration workflows.
- **Deprecated**: replaced by newer packages while retaining backward compatibility for historical data.

## ClassificationMapping
### Fields
- `mapping_id` (UUID): primary key.
- `source_classification_id` (FK): classification within current graph.
- `target_graph_id` (FK, optional): external graph for the mapping.
- `target_classification_code` (string): external code or taxonomy identifier.
- `mapping_type` (enum): equivalence, broader, narrower, deprecated, custom.
- `confidence` (decimal, optional): degree of certainty for automated mapping.
- `effective_from` / `effective_to` (timestamp, optional).
- `created_at` / `updated_at` (timestamp).
### Relationships
- Enables federated reporting across repositories and external catalogs.
- Referenced during migration/import workflows and by governance policies enforcing cross-walk alignment.
### Lifecycle
- **Draft**: mapping proposed and under review.
- **Active**: mapping available to placement rules, reports, and synchronization jobs.
- **Deprecated**: mapping retained for historical traceability but excluded from new calculations.

## PlacementRule
### Fields
- `placement_rule_id` (UUID): primary key.
- `graph_id` (FK): owning graph.
- `classification_id` (FK): targeted classification constraint.
- `applies_to_descendants` (boolean): when true, rule applies to the entire classification subtree.
- `artifact_subtype` (enum/array): eligible artifact subtype(s).
- `min_lifecycle_state` (enum): minimum `ArtifactVersion` state (e.g., `approved`).
- `lifecycle_window` (daterange): optional effective window for placement enforcement.
- `scope` (enum): enterprise, segment, capability.
- `applies_to_roles` (array<FK>): graph roles required to perform placement.
- `required_policy_id` (FK, optional): linked `GovernancePolicy` enforcing evidence.
- `violation_handler` (enum): block_submission, warn_only, auto_route_to_review.
- `enforcement_level` (enum): advisory, blocking.
- `created_by` / `created_at` (FK/timestamp): audit metadata.
- `updated_at` (timestamp): last modification timestamp.
### Relationships
- Belongs to a `Repository` and references a `RepositoryClassification`.
- Consumed by governance engines when validating `ClassificationAssignment`.
- Linked to `RequiredCheck` definitions to enforce additional evidence gates.
- Evaluated during transformation readiness reviews and release gating workflows.
- Supports subtree application when `applies_to_descendants = true`, keeping policies consistent across classification hierarchies.
### Lifecycle
- **Draft**: authored and iterated by governance owners.
- **Effective**: actively enforced during classification, promotion, and sync.
- **Suspended**: temporarily bypassed (e.g., policy changes under review).
- **Retired**: no longer evaluated but preserved for audit history.

## RepositoryAccessRole
### Fields
- `role_id` (UUID): primary key.
- `graph_id` (FK): owning graph.
- `code` / `label` (string): unique handle for API and UX (e.g., `graph_admin`).
- `description` (string): surfaced in role selection dialogs.
- `permission_scope` (jsonb): fine-grained permissions (placement, review, integration).
- `is_default` (boolean): determines assignment for new members.
- `weight` (int): precedence when resolving conflicts across inherited roles.
- `grants_governance_membership` (boolean): auto-adds to `GovernanceOwnerGroup` if true.
- `created_at` / `updated_at` (timestamp): audit timestamps.
### Relationships
- Owned by `Repository`; referenced by `RepositoryMembership` and workflows engines.
- Mapped to identity provider groups or service principals for SSO provisioning.
- Influences visibility of classifications, placement actions, and integration controls.
### Lifecycle
- **Draft**: configured but not yet assignable.
- **Active**: available for membership grants and evaluated in authorization checks.
- **Deprecated**: replaced by a newer role; no new assignments permitted.
- **Retired**: removed from active use; historical memberships reference snapshot.

## RepositoryMembership
### Fields
- `membership_id` (UUID): primary key.
- `graph_id` (FK): scope of access.
- `principal_id` (FK): user or service principal receiving access.
- `role_id` (FK): granted `RepositoryAccessRole`.
- `assignment_source` (enum): direct, group, organization_policy, service_account.
- `state` (enum): pending, active, suspended, revoked, expired.
- `granted_at` / `granted_by` (timestamp/FK): issuance metadata.
- `expires_at` (timestamp, optional): scheduled access expiration.
- `revoked_at` / `revoked_by` (timestamp/FK, optional): deprovision metadata.
- `provisioning_ticket` (string): external IDM or audit reference.
### Relationships
- Belongs to `Repository`; joins `RepositoryAccessRole` and identity directory records.
- Surfaces to governance dashboards for accountability and quorum calculations.
- Drives inherited visibility for artifacts, transformations, and classifications.
### Lifecycle
- **Pending**: invitation sent or provisioning in progress.
- **Active**: access granted and visible in authorization cache.
- **Suspended**: temporarily disabled (e.g., compliance hold) but retention maintained.
- **Revoked**: access withdrawn; historical record retained for audit.
- **Expired**: automatically closed due to `expires_at` window.

## RepositoryIntegrationEndpoint
### Fields
- `endpoint_id` (UUID): primary key.
- `graph_id` (FK): owning graph.
- `integration_type` (enum): knowledge_graph, search_index, archive, analytics_feed, webhook.
- `display_name` (string): surfaced in integrations catalog.
- `configuration` (jsonb): connector-specific settings and routing details.
- `secrets_ref` (string): pointer to secret manager entry.
- `status` (enum): active, disabled, error, pending_validation.
- `last_sync_at` (timestamp): most recent successful dispatch.
- `error_state` (jsonb): last failure context and remediation guidance.
- `created_at` / `updated_at` (timestamp): audit metadata.
### Relationships
- Binds graph data pipelines to downstream systems and `RepositorySyncJob` executions.
- Referenced by alerting rules to monitor integration health.
- Supplies configuration to automation routines publishing graph deltas.
### Lifecycle
- **Registered**: connector created, waiting for validation.
- **Active**: validated and receiving sync jobs.
- **Disabled**: manually paused; sync jobs skipped.
- **Error**: health checks failing; remediation required.
- **Retired**: connector removed from service; historical syncs retained.

## RepositorySyncJob
### Fields
- `sync_job_id` (UUID): primary key.
- `graph_id` (FK): scope of synchronization.
- `integration_id` (FK, optional): target endpoint when applicable.
- `sync_target` (enum): knowledge_graph, search_index, analytics_snapshot, archive_snapshot.
- `trigger_type` (enum): scheduled, delta_event, manual, remediation.
- `status` (enum): queued, running, succeeded, failed, cancelled.
- `started_at` / `completed_at` (timestamp): execution timestamps.
- `duration_ms` (int): execution time used for SLA monitoring.
- `summary` (text): outcome description, counts, and warnings.
- `retry_count` (int): number of automatic retries attempted.
### Relationships
- Belongs to `Repository`; references `RepositoryIntegrationEndpoint` when applicable.
- Emits metrics feeding `RepositoryStatisticSnapshot` cadence.
- Surfaces to operations dashboards and audit logs for traceability.
- Generates `ArtifactSyncPayload` entries for artifact/version deltas and propagates `SlaBreachEvent` resolutions to downstream systems.
### Lifecycle
- **Queued**: awaiting execution by sync orchestrator.
- **Running**: active job processing.
- **Succeeded**: completed without errors; downstream systems updated.
- **Failed**: encountered error; may trigger remediation or support escalation.
- **Cancelled**: manually stopped or superseded by newer job.

## ImportJob
### Fields
- `import_job_id` (UUID): primary key.
- `graph_id` (FK)
- `initiated_by` (FK)
- `status` (enum): validating, running, quarantined, completed, failed, rolled_back.
- `manifest_uri` (string): storage location of the submitted manifest.
- `batch_size` (int): number of records processed per run.
- `success_count` / `failure_count` (int): aggregate row outcomes.
- `started_at` / `completed_at` (timestamp)
- `last_error` (jsonb, optional): summary of fatal failure context.
- `graph_statistic_snapshot_id` (FK, optional): metrics snapshot captured at completion.
- `sync_job_id` (FK, optional): downstream `RepositorySyncJob` triggered after import success.
### Relationships
- Owns many `ImportItem` rows representing per-record outcomes.
- Referenced by governance dashboards to track bulk operations and SLA adherence.
- Surfaces to telemetry so import throughput and error categories feed analytics.
### Lifecycle
- **Validating**: manifest schema and placement simulation in progress.
- **Running**: batches executing and ImportItems being created.
- **Quarantined**: failure rate exceeded threshold; awaiting remediation.
- **Completed**: all rows processed successfully (post-quarantine retries included).
- **Failed**: unrecoverable error; optional rollback executed.
- **Rolled_Back**: administrator reverted applied changes; audit retained.

## ImportItem
### Fields
- `import_item_id` (UUID): primary key.
- `import_job_id` (FK)
- `row_number` (int): manifest row reference.
- `target_type` (enum): artifact, artifact_version, classification_assignment, trace_link, graph_element.
- `target_id` (UUID, optional): entity created/updated when successful.
- `state` (enum): pending, validated, processed, quarantined, retried, failed.
- `error_code` (string, optional): machine-readable diagnosis (e.g., invalid_element, placement_violation).
- `error_details` (jsonb, optional)
- `processed_at` (timestamp, optional)
- `retry_count` (int)
### Relationships
- Belongs to `ImportJob` and, when successful, references created domain entities for traceability.
- Quarantined records trigger governance notifications and can be reprocessed after remediation.
- Exported to analytics to classify failure categories and remediation times.
### Lifecycle
- **Pending**: awaiting validation.
- **Validated**: ready for execution.
- **Processed**: successfully applied to graph records.
- **Quarantined**: held for manual remediation; may transition to `retried`.
- **Retried**: undergoing reprocessing after fixes.
- **Failed**: permanently rejected; retained for audit.

## RepositoryStatisticSnapshot
### Fields
- `snapshot_id` (UUID): primary key.
- `graph_id` (FK): source graph.
- `captured_at` (timestamp): sample time.
- `metric_payload` (jsonb): aggregated metrics (cycle time percentiles, backlog depth, SLA compliance).
- `source_job_id` (FK, optional): originating `RepositorySyncJob` or analytics pipeline.
- `trend_window` (daterange): period represented by the snapshot.
- `anomaly_flags` (jsonb): detected outliers or SLA breaches.
### Relationships
- Belongs to `Repository`; consumed by governance dashboards and portfolio analytics.
- Correlates with `GovernanceMetricSnapshot` for cross-tier reporting.
- Drives notifications when anomaly thresholds cross policy limits.
- Linked to `InitiativeMetricSnapshot` for synchronized transformation/graph health views.
### Lifecycle
- **Captured**: snapshot recorded and published to analytics consumers.
- **Superseded**: replaced by newer observation but retained for trend analysis.
- **Archived**: moved to long-term storage per retention policy.

## CrossRepositoryReference
### Fields
- `reference_id` (UUID): primary key.
- `source_graph_id` (FK): graph establishing the reference.
- `target_graph_id` (FK): graph providing the referenced artifact or element.
- `source_artifact_id` / `source_element_id` (FK, optional): local context.
- `target_artifact_id` / `target_element_id` (FK, optional): external object.
- `relationship_type` (enum): consumes, depends_on, informs, governs, shares_control.
- `governance_policy_links` (array<FK>): policies that apply across both repositories.
- `sync_strategy` (enum): notify-only, bidirectional, scheduled_snapshot.
- `status` (enum): draft, active, suspended, retired.
- `metadata` (jsonb): additional routing notes (e.g., external taxonomy codes).
- `created_at` / `updated_at` (timestamp).
### Relationships
- Enables multi-graph collaboration scenarios (Journey 238).
- Referenced by `RepositorySyncJob` to determine cross-tenant payload routing.
- Feeds governance dashboards so owner groups see shared responsibilities.
- Links to transformations to surface dependencies spanning repositories.
### Lifecycle
- **Draft**: proposed linkage awaiting approvals.
- **Active**: enforced by placement rules and sync pipelines.
- **Suspended**: temporarily disabled (e.g., incident response).
- **Retired**: archived for audit when collaboration ends.

## ChangeManagementProfile (Repository Scope)
> Detailed structure defined in `220-strategy-change-entity-model.md`.

### Purpose
- Provides graph-wide defaults for change playbooks, adoption targets, and change owner groups.
- Ensures every transformation launched within the graph inherits baseline communication/training expectations aligned with governance policies.

### Key Relationships
- Associated with `Repository` (one-to-many when multiple profiles exist for different segments).
- References `ChangePlaybook` templates and `GovernanceOwnerGroup` for escalation alignment.
- Surfaces metrics into `OutcomeMetricSnapshot` to compare actual adoption vs graph expectations.
