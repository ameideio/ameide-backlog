# Governance Entities

## GovernanceOwnerGroup
### Fields
- `owner_group_id` (UUID): primary key.
- `graph_id` (FK): scope of authority.
- `name` / `description`: surfaced in governance profile banners.
- `coverage_scope` (jsonb): classifications, artifact subtypes, transformations covered.
- `membership` (array<FK>): user/group identifiers with roles (lead, reviewer, delegate).
- `escalation_chain` (array<FK>): ordered backup contacts.
- `quorum_rules` (jsonb): required approvals, abstain handling.
- `linked_graph_role_ids` (array<FK>): associated `RepositoryAccessRole` entries auto-granting governance duties.
- `default_review_sla` (duration): overrides graph default for this coverage scope.
- `created_at` / `updated_at`.
### Relationships
- Assigned to repositories, classifications, and transformations.
- Provides default reviewers on `ReviewCase` creation.
- Linked to `GovernancePolicy` ownership metadata.
- Syncs with `RepositoryAccessRole` grants via `linked_graph_role_ids` for automatic quorum coverage.
### Lifecycle
- Draft → Active → Suspended (coverage gap) → Retired (superseded).

## GovernanceProfile
### Fields
- `governance_profile_id` (UUID): primary key.
- `graph_id` (FK): owning graph.
- `name` / `description`.
- `version` (string): semantic version of the profile bundle.
- `status` (enum): draft, active, deprecated.
- `policy_ids` (array<FK>): cached ordering of `GovernancePolicy` versions included in the bundle (canonical mapping stored in `GovernanceProfilePolicy`).
- `playbook_ids` (array<FK>): associated `GovernanceWorkflowPlaybook` templates.
- `required_check_catalog_id` (FK, optional): default catalogue grouping `RequiredCheck` definitions.
- `default_owner_group_ids` (array<FK>): owner groups applied at activation.
- `change_window` (jsonb): effectiveness period and rollout guidance.
- `activation_notes` (text): rationale, rollout instructions, audit references.
- `created_at` / `updated_at` (timestamp).
### Relationships
- Referenced by `Repository` (`governance_profile_id`) during provisioning.
- Surfaces in governance dashboards so transformations inherit guardrails.
- Coordinates with `RecertificationJob` templates to schedule policy rollouts.
- Published via `RepositorySyncJob` payloads for downstream audit replicas.
- Joined to policies through `GovernanceProfilePolicy` so versions remain auditable and reusable.
### Lifecycle
- **Draft**: assembled by governance leads.
- **Active**: enforced for placement, review, and transformation workflows.
- **Deprecated**: retained for historical traceability once superseded.

## GovernanceProfilePolicy
### Fields
- `governance_profile_id` (FK)
- `policy_id` (FK)
- `sequence_order` (int, optional): ordering guidance when multiple policies apply.
- `created_at` / `updated_at` (timestamp).
### Relationships
- Join table supporting many-to-many between `GovernanceProfile` and `GovernancePolicy`.
- Enables policy bundles to be re-used across repositories.
### Lifecycle
- Entries created when policies are attached to a profile and retired when the profile version is deprecated.

## GovernancePolicy
### Fields
- `policy_id` (UUID)
- `graph_id` (FK)
- `name` / `summary`
- `policy_type` (enum): governance, quality, compliance, risk.
- `effective_from` / `effective_to`
- `required_evidence` (jsonb): checklist items.
- `linked_checks` (array<FK>): default RequiredCheck IDs.
- `status` (enum): draft, ratified, superseded.
- `default_review_playbook_id` (FK, optional): default workflows template for ReviewCases spawned by this policy.
### Relationships
- Owned by `GovernanceOwnerGroup`.
- Drives `PlacementRule` and `RequiredCheck` configuration.
- Triggers `RecertificationJob` when updated.
- Bundled into `GovernanceProfile` records for graph activation.
- References `OntologyCatalogue` entries to enforce approved modeling standards for `RepositoryElement`.
- Coordinates with `ChangeManagementProfile` to ensure change escalations follow governed playbooks.
### Lifecycle
- Authored → Reviewed → Ratified → Monitored → Superseded/Expired.

## ReviewCase
### Fields
- `review_case_id` (UUID)
- `artifact_version_id` (FK)
- `artifact_id` (FK)
- `graph_id` (FK)
- `classification_id` (FK, optional)
- `initiated_by` (FK): submitter
- `assigned_owner_group_id` (FK)
- `sla_target` (duration)
- `sla_due_at` (timestamp): computed deadline derived from `opened_at + sla_target`.
- `required_evidence` (jsonb)
- `state` (enum): open, in_progress, changes_requested, on_hold, resolved, escalated.
- `resolution` (enum, optional): approved, rejected, withdrawn, superseded.
- `resolved_at` (timestamp, optional)
- `resolved_by_id` (FK, optional): final approver/closer recorded when resolution set.
- `opened_at` / `closed_at`
- `last_status_synced_at` (timestamp): last time downstream systems consumed case status.
- `sync_cursor` (string): emitted with `RepositorySyncJob` payloads for downstream replay.
- `graph_statistic_snapshot_id` (FK, optional): linkage to `RepositoryStatisticSnapshot` at creation time.
### Relationships
- Enqueues `PromotionQueueEntry`.
- Collects `ApprovalDecision` and `CheckRun` records.
- Resolves into `GovernanceRecord`.
- Published via `RepositorySyncJob` payloads to knowledge graph and analytics consumers.
- Links to `ElementAttachment` entries when reviews cover specific `RepositoryElement` impacts (e.g., capability changes).
- Provides `sla_due_at` to downstream queues and notifications so SLA monitoring is consistent across touchpoints.
### Lifecycle
- Opened upon review request → In Progress with approvals → Resolved (approved/rejected or other resolution) → Archived.
- SLA monitors produce `SlaBreachEvent` entries when `sla_target`/`sla_due_at` thresholds are exceeded prior to Resolution.

## PromotionQueueEntry
### Fields
- `queue_entry_id` (UUID)
- `review_case_id` (FK)
- `position` (int)
- `priority` (enum/int)
- `escalation_state` (enum): normal, escalated, waived.
- `entered_at` / `processed_at`
- `graph_statistic_snapshot_id` (FK, optional): reference to `RepositoryStatisticSnapshot`.
- `sla_due_at` (timestamp, optional): computed deadline for queue processing.
- `sla_state` (enum): on_track, warning, breach.
### Relationships
- Belongs to a `ReviewCase`.
- Referenced by dashboards and readiness reports.
- Updated by queue management workflows.
- Contributes to `RepositoryStatisticSnapshot` queue depth calculations via `graph_statistic_snapshot_id` and exposes `sla_due_at` for escalation monitoring.
### Lifecycle
- Created when case enters promotion queue → Reordered / Escalated as needed → Processed on approval → Closed.

## ApprovalDecision
### Fields
- `approval_id` (UUID)
- `review_case_id` (FK)
- `approver_id` (FK)
- `decision` (enum): approve, request_changes, abstain.
- `rationale` (text)
- `evidence_links` (array<string>)
- `decided_at` (timestamp)
- `delegated_from_id` (FK, optional): original approver if decision delegated.
- `sla_state` (enum): on_time, late, escalated.
- `notification_id` (FK, optional): associated governance notification record.
### Relationships
- Belongs to `ReviewCase`.
- Summarized in transformation blockers funnel.
- Feeds audit logs and readiness reports.
- Tied to `GovernanceNotification` entries to confirm delivery of decision requests or outcomes.
### Lifecycle
- Pending → Recorded → Immutable (only superseded by new cycle).

## ControlCatalogue
### Fields
- `catalogue_id` (UUID): primary key.
- `graph_id` (FK): scope of catalogue.
- `name` / `description`: displayed in governance configuration UI.
- `frameworks` (array<string>): frameworks included (ISO27001, NIST CSF, SOC2, custom).
- `control_ids` (array<FK>): controls curated within the catalogue.
- `default_recert_interval` (interval, optional): recommended cadence applied when control-specific values absent.
- `evidence_template_ids` (array<FK>): default evidence artefact templates.
- `status` (enum): draft, published, retired.
- `created_at` / `updated_at` (timestamp).
### Relationships
- Referenced by `GovernancePolicy` and `GovernanceProfile` to seed control coverage.
- Tied to `RequiredCheck` definitions ensuring evidence aligns with control expectations.
- Syncs to analytics so dashboards show control coverage by framework.
### Lifecycle
- **Draft**: curated by compliance teams.
- **Published**: available for policy binding and auditing.
- **Retired**: retained for historical traceability when replaced.

## Control
### Fields
- `control_id` (UUID): primary key.
- `framework` (enum/string): control framework identifier (ISO27001, NIST, CIS, custom).
- `control_code` (string): framework-specific code or reference.
- `requirement_text` (text): canonical statement of the control.
- `severity` (enum): low, medium, high, critical.
- `recert_interval` (interval, optional): recommended recertification cadence.
- `evidence_requirements` (jsonb): structured evidence checklist/remediation notes.
- `created_at` / `updated_at` (timestamp).
### Relationships
- Linked to `ControlCatalogue` membership for curated governance bundles.
- Linked to `ControlMapping` which ties controls to RequiredChecks.
- Surfaced in compliance dashboards to show coverage and gaps per framework.
### Lifecycle
- Draft → Approved → Monitored → Superseded, retaining historical versions for audit.

## ControlMapping
### Fields
- `control_id` (FK)
- `check_id` (FK)
- `mapping_type` (enum): verifies, provides_evidence_for, compensating, manual_override.
- `created_at` / `updated_at` (timestamp).
### Relationships
- Many-to-many bridge between `Control` and `RequiredCheck`, enabling coverage reporting and automated attestation.
- Referenced by governance dashboards and policy editors when evaluating control efficacy.
### Lifecycle
- Active mappings adjusted as controls evolve; historical mappings retained.

## RequiredCheck
### Fields
- `check_id` (UUID)
- `name` / `description`
- `check_type` (enum): automated, manual, external.
- `enforcement_level` (enum): blocking, advisory.
- `owner_group_id` (FK)
- `policy_id` (FK, optional)
- `retry_policy` (jsonb)
- `timeout` (duration)
- `evidence_template_id` (FK, optional): structured evidence form required on pass/fail.
- `supports_async_execution` (boolean): indicates whether check runs over long-running external jobs.
- `last_reviewed_at` (timestamp): governance review of check configuration.
### Relationships
- Scoped via `CheckBinding`.
- Executed through `CheckRun`.
- Displayed on readiness dashboards and blockers funnel.
- May evaluate `RepositoryElement` or `ElementProjection` conformance (e.g., capability model standards) when associated via `CheckBinding`.
- Linked through `ControlMapping` to external control frameworks, enabling compliance coverage reporting.
### Lifecycle
- Defined → Published → Iterated → Deprecated.

## CheckBinding
### Fields
- `binding_id` (UUID)
- `check_id` (FK)
- `scope_type` / `scope_id`: classification, policy, transformation, release, etc.
- `activation_state` (enum): active, paused, retired.
- `conditions` (jsonb): conditional expressions (e.g., artifact subtype, risk score).
- `placement_rule_id` (FK, optional): ties binding to a specific placement rule constraint.
- `effective_from` / `effective_to` (timestamp, optional): bounds for temporary enforcement.
- `priority` (int): evaluation order when multiple bindings apply.
### Relationships
- Links RequiredCheck to applicable scope.
- Evaluated during review and promotion.
- When tied to `PlacementRule`, ensures classification enforcement and evidence alignment.
- Supports additional scope types for `RepositoryElement`, `ElementProjection`, or change constructs (e.g., `ChangePlan`) so ontology and adoption rules are governed uniformly.
### Lifecycle
- Configured → Active → Paused → Retired.

## CheckRun
### Fields
- `check_run_id` (UUID)
- `check_id` (FK)
- `artifact_version_id` (FK) or `release_package_id` (FK)
- `status` (enum): pending, running, warning, passed, failed, waived, expired.
- `started_at` / `completed_at`
- `executor` (enum): system, human, external_service.
- `evidence_uri` (string)
- `failure_details` (jsonb)
- `sync_payload_id` (FK, optional): link to `ArtifactSyncPayload` used for the run.
- `requested_by` (FK, optional): initiator of the run or rerun.
- `requested_at` (timestamp)
- `rerun_requested_at` (timestamp, optional)
### Relationships
- Belongs to a RequiredCheck and associated artifact/release.
- Referenced by `ReviewCase` state and readiness reports.
- May trigger `RecertificationJob` if failure severity high.
- Sync payload linkage enables replay during RepositorySyncJob troubleshooting.
- When `supports_async_execution=true`, checks remain in `running` until external completion; SLA monitors compare `requested_at` and `completed_at` against the check timeout to detect latent breaches.
### Lifecycle
- Scheduled → Running → Completed (pass/fail/waived) → Archived.

## DataQualitySurvey
### Fields
- `survey_id` (UUID)
- `policy_id` (FK)
- `target_scope` (jsonb): owner group, classification, artifact subtype.
- `questionnaire_template` (jsonb)
- `due_date` / `reminder_schedule`
- `status` (enum): drafted, issued, closed.
- `graph_statistic_snapshot_id` (FK, optional): contextual metrics captured at issuance.
- `linked_sla_policy_id` (FK, optional): aligns survey obligations with SLA targets.
### Relationships
- Issues `SurveyResponse` requests to artifact owners.
- Outcomes influence `GovernanceMetricSnapshot`.
- May trigger `RecertificationJob`.
- Shares contextual metrics with `RepositoryStatisticSnapshot` for holistic reporting.
- Can target `StakeholderSegment` or `ChangeImpactProfile` cohorts to validate adoption data before key decisions.
### Lifecycle
- Authored → Issued → Responses Collected → Closed → Archived.

## SurveyResponse
### Fields
- `response_id` (UUID)
- `survey_id` (FK)
- `respondent_id` (FK)
- `submitted_at`
- `answers` (jsonb)
- `score` (decimal)
- `flags` (jsonb): follow-up items.
- `linked_review_case_id` (FK, optional): connects response to active governance case.
- `synced_at` (timestamp): when response propagated to analytics and reporting.
### Relationships
- Belongs to `DataQualitySurvey`.
- Updates `GovernanceMetricSnapshot`.
- Alerts GovernanceOwnerGroup when thresholds breached.
- Linked to `ReviewCase` when responses inform active governance decisions.
### Lifecycle
- Pending → Submitted → Validated → Archived.

## RecertificationJob
### Fields
- `recertification_job_id` (UUID)
- `policy_id` (FK)
- `trigger_reason` (enum): policy_change, sla_breach, data_quality_drop.
- `initiated_at` / `completed_at`
- `target_artifacts` (array<FK>)
- `status` (enum): pending, running, completed, aborted.
- `actions_taken` (jsonb): e.g., artifact pushed to in_review.
- `graph_statistic_snapshot_id` (FK, optional): metrics context when job initiated.
- `sync_job_id` (FK, optional): downstream sync publishing recertification outcomes.
### Relationships
- References `ArtifactVersion`, `RequiredCheck`, `ReviewCase`.
- Generates alerts for transformations consuming impacted artifacts.
- Feeds `RepositorySyncJob` when recertification outcomes alter artifact availability.
### Lifecycle
- Triggered → Assessed → Actions Executed → Closed.

## GovernanceMetricSnapshot
### Fields
- `metric_snapshot_id` (UUID)
- `graph_id` (FK)
- `transformation_id` (FK, optional)
- `captured_at` (timestamp)
- `approval_cycle_time` (object): P50/P95 stats.
- `approval_metrics` (jsonb): counts of approvals, rejections, resubmissions, quorum completions.
- `queue_wait_time` (object)
- `check_pass_rate` (decimal)
- `data_quality_score` (decimal)
- `recertification_events` (int)
- `graph_statistic_snapshot_id` (FK, optional): direct link to underlying graph metrics.
- `anomaly_flags` (jsonb): SLA breaches, backlog spikes, data quality regressions.
### Relationships
- Aggregates inputs from `CheckRun`, `SurveyResponse`, `PromotionQueueEntry`.
- Joined with `InitiativeMetricSnapshot` for portfolio analytics.
- References `RepositoryStatisticSnapshot` to align platform-level reporting.
- Correlated with `OutcomeMetricSnapshot` to reconcile governance health with benefit and adoption telemetry.
### Lifecycle
- Captured on schedule → Stored for trending → Archived per retention policy.

## GovernanceBackgroundJob
### Fields
- `background_job_id` (UUID)
- `graph_id` (FK)
- `job_type` (enum): review_sync, evidence_digest, recertification_dispatch, notification_replay, telemetry_harvest, custom.
- `related_entity_type` / `related_entity_id` (optional): review_case, artifact_version, policy, transformation, etc.
- `priority` (enum/int)
- `status` (enum): queued, running, succeeded, failed, cancelled.
- `arguments` (jsonb): serialized payload driving the job.
- `created_at` / `queued_at`
- `started_at` / `completed_at`
- `retry_count` (int)
- `last_error` (jsonb, optional)
- `triggered_by_id` (FK, optional): principal or automation that enqueued the job.
### Relationships
- Orchestrates deferred governance automation, including dispatching `RepositorySyncJob` runs after approvals, scheduling `RecertificationJob`s, and replaying notifications for `ReviewCase` updates.
- Emits telemetry consumed by `GovernanceMetricSnapshot` and `InitiativeAlert` so stakeholders see automation throughput and failures.
- Integrates with `SlaBreachEvent` to escalate stuck jobs and with `GovernanceNotification` to confirm completion to owner groups.
### Lifecycle
- Created when automation enqueues work → Running while processors execute → Succeeded/Failed/Canceled with audit history retained for compliance.

## LifecycleEvent
### Fields
- `lifecycle_event_id` (UUID)
- `artifact_version_id` (FK)
- `actor_id` (FK)
- `from_state` / `to_state`
- `occurred_at`
- `rationale` (text)
- `evidence_links` (array<string>)
- `sync_cursor` (string): emitted for downstream analytics replication.
- `published_at` (timestamp): propagation time to graph sync targets.
### Relationships
- Audit trail for state changes consumed by governance dashboards.
- Referenced by `ReviewCase` and `GovernanceRecord`.
- Published via `RepositorySyncJob` for replication to knowledge graph and analytics stores.
### Lifecycle
- Immutable append-only log; no state transitions beyond recording.

## Routing & Alerts

### GovernanceRoutingRule
#### Fields
- `routing_rule_id` (UUID)
- `graph_id` (FK)
- `trigger_type` (enum): classification_assignment, required_check_failure, survey_flagged, transformation_blocker, manual_submission.
- `condition` (jsonb): expression evaluating scope, risk tier, lifecycle state.
- `owner_group_id` (FK): governance group receiving routed work.
- `fallback_owner_group_id` (FK, optional): backup group if primary lacks quorum.
- `priority` (enum/int)
- `created_at` / `updated_at`
#### Relationships
- Consumed when creating `ReviewCase`, `RecertificationJob`, or `SlaBreachEvent`.
- Aligns with `PlacementRule` and `CheckBinding` to ensure consistent enforcement coverage.
- Extends to change triggers (e.g., adoption decline, change task breach) by integrating `AdoptionSignal` and `ChangeTask` escalations.
#### Lifecycle
- Authored → Validated → Active → Suspended → Retired.

### SlaBreachEvent
#### Fields
- `sla_breach_event_id` (UUID)
- `graph_id` (FK)
- `related_entity_type` / `related_entity_id`: review_case, check_run, survey, transformation.
- `breach_type` (enum): review_sla, queue_wait, check_runtime, survey_response.
- `breach_detected_at` (timestamp)
- `threshold_minutes` (int)
- `actual_minutes` (int)
- `owner_group_id` (FK)
- `severity` (enum): info, warning, critical.
- `resolved_at` (timestamp, optional)
#### Relationships
- Triggers `GovernanceNotification` and may spawn `RecertificationJob`.
- Aggregated into `RepositoryStatisticSnapshot` and `GovernanceMetricSnapshot`.
#### Lifecycle
- Detected → Routed → Mitigated → Closed.

### GovernanceNotification
#### Fields
- `notification_id` (UUID)
- `graph_id` (FK)
- `recipient_id` (FK) or `recipient_channel` (email, webhook, slack, teams)
- `notification_type` (enum): review_assignment, decision_request, review_feedback, review_resubmission, review_outcome, sla_breach, check_failure, survey_due, recertification, retention_event.
- `payload` (jsonb)
- `sent_at` (timestamp)
- `delivery_status` (enum): pending, sent, failed, acknowledged.
- `acknowledged_at` (timestamp, optional)
- `related_entity_type` / `related_entity_id`
#### Relationships
- Linked to `ApprovalDecision`, `ReviewCase`, `SlaBreachEvent`, or `RecertificationJob`.
- Feeds audit trails and activity feeds for accountability.
- Supports change enablement triggers by notifying responsible parties when `AdoptionSignal` trends decline or `ChangeTask` milestones slip, and captures review loops via `review_feedback` / `review_resubmission` plus outcome broadcasts via `review_outcome`.
#### Lifecycle
- Enqueued → Delivered → Acked/Failed; retries logged on failure.

### GovernanceEscalation
#### Fields
- `escalation_id` (UUID)
- `related_entity_type` / `related_entity_id`
- `owner_group_id` (FK)
- `initiated_at` (timestamp)
- `escalation_reason` (enum): quorum_gap, sla_breach, decision_deadlock, evidence_missing.
- `status` (enum): open, acknowledged, mitigated, closed.
- `mitigation_plan` (jsonb)
- `resolved_at` (timestamp, optional)
- `resolved_by` (FK, optional)
#### Relationships
- References `GovernanceNotification` records to ensure stakeholders are informed.
- Updates `PromotionQueueEntry` and `ReviewCase` states upon mitigation.
- Can reference change constructs (e.g., `ChangeTask`, `CommunicationPlan`) when escalations span governance and adoption responsibilities.
#### Lifecycle
- Opened → Routed → Mitigated → Closed, with audit trail retained.

### GovernanceWorkflowPlaybook
#### Fields
- `playbook_id` (UUID)
- `graph_id` (FK)
- `name` / `description`
- `applicability_scope` (jsonb): classification, artifact subtype, policy type.
- `steps` (jsonb): ordered tasks, decision points, automation hooks.
- `default_owner_group_id` (FK)
- `sla_guidance` (jsonb): recommended timelines per step.
- `active` (boolean)
- `version` (string)
#### Relationships
- Referenced by `GovernancePolicy` (`default_review_playbook_id`) and `ReviewCase`.
- Supplies templates for `GovernanceNotification` sequencing and `CheckBinding` orchestration.
#### Lifecycle
- Authored → Approved → Published → Iterated; previous versions retained for audit.

### GovernanceSupportTicket
#### Fields
- `support_ticket_id` (UUID)
- `graph_id` (FK)
- `source` (enum): automation, user_report, integration_monitoring.
- `subject` / `description`
- `severity` (enum): low, medium, high, critical.
- `status` (enum): open, investigating, awaiting_response, resolved, closed.
- `opened_at` / `updated_at` / `closed_at`
- `related_entity_type` / `related_entity_id`
- `assigned_to` (FK, optional)
#### Relationships
- Interfaces with operations/helpdesk tooling; may spawn `GovernanceEscalation`.
- Links back to `RepositoryIntegrationEndpoint` or `RepositorySyncJob` when integration failures occur.
#### Lifecycle
- Opened → Investigated → Resolved → Closed; knowledge base entry created when applicable.
