# Strategy & Change Enablement Entities

## Domain Scope
This model captures portfolio strategy, benefit realisation, and change enablement constructs that complement the graph, artifact, and governance entity sets. Entities are designed to integrate with transformations, RepositoryElements, and telemetry snapshots so strategic intent, adoption plans, and realised value remain traceable across the Ameide platform.

## Strategic Portfolio Layer

### StrategicTheme
#### Fields
- `strategic_theme_id` (UUID): primary key.
- `name` / `description`: articulated vision statement and supporting narrative.
- `horizon` (enum): near_term, mid_term, long_term.
- `business_driver` (enum): growth, efficiency, compliance, resilience, innovation.
- `sponsor_id` (FK): accountable executive.
- `benefit_targets` (jsonb): planned quantitative outcomes with currency/unit metadata.
- `governance_owner_group_id` (FK, optional): steward for ongoing alignment.
- `status` (enum): proposed, approved, in_flight, completed, sunset.
- `created_at` / `updated_at`.
#### Relationships
- Links to `Initiative` via `StrategicThemeAssignment` for portfolio alignment.
- Mapped to RepositoryElements (e.g., capabilities, value streams) through `TraceLink`.
- Referenced by `BenefitHypothesis` to anchor metric expectations.
#### Lifecycle
- Proposed during strategy cycle → Approved by steering board → Tracked through execution → Sunset when objectives met or superseded.

### StrategicThemeAssignment
#### Fields
- `assignment_id` (UUID)
- `strategic_theme_id` (FK)
- `transformation_id` (FK)
- `alignment_role` (enum): primary, supporting, exploratory.
- `alignment_rationale` (text)
- `created_at` / `updated_at`
#### Relationships
- Bridges themes and transformations; surfaced in portfolio dashboards and transformation workspaces.
- Supports gating workflows by linking transformations to funding guardrails.
#### Lifecycle
- Created during transformation intake; updated when scope pivots; archived when transformation closes.

### BenefitHypothesis
#### Fields
- `benefit_hypothesis_id` (UUID)
- `strategic_theme_id` (FK, optional)
- `transformation_id` (FK, optional)
- `benefit_category` (enum): revenue, cost_avoidance, cost_reduction, customer_experience, risk_reduction, sustainability.
- `baseline_value` (decimal + unit metadata)
- `target_value` (decimal + unit metadata)
- `measurement_frequency` (enum): weekly, monthly, quarterly, annually.
- `assumptions` (text)
- `risks` (text)
- `owner_id` (FK): accountable benefits manager.
- `status` (enum): draft, approved, tracking, realised, retired.
- `created_at` / `updated_at`
#### Relationships
- Drives instrumentation for `OutcomeMetricSnapshot`.
- Linked to RepositoryElements (e.g., capability improvements) and artifacts (business cases, decision records) via `TraceLink` and `ElementAttachment`.
- Guides funding guardrails and Go/No-Go checkpoints surfaced in transformations.
#### Lifecycle
- Authored during planning → Approved alongside transformation → Monitored via telemetry → Marked realised or retired.

### FundingGuardrail
#### Fields
- `guardrail_id` (UUID)
- `strategic_theme_id` (FK)
- `transformation_id` (FK, optional)
- `guardrail_type` (enum): spend_cap, stage_gate, evidence_requirement, dependency_blocker.
- `threshold` (jsonb): specific limit or success criteria (currency, percentage, qualitative).
- `enforced_by_policy_id` (FK, optional): governance/change policy reference.
- `status` (enum): active, waived, retired.
- `justification` (text)
- `created_at` / `updated_at`
#### Relationships
- Tied to transformations and benefit hypotheses; displayed in readiness and funding dashboards.
- May trigger `GovernanceNotification` or change escalation when thresholds breached.
#### Lifecycle
- Defined during funding cycle → Active during execution → Waived or retired upon completion.

## Change Enablement Layer

### ChangeManagementProfile
#### Fields
- `change_profile_id` (UUID)
- `graph_id` (FK)
- `name` / `description`
- `default_playbook_id` (FK): reference to `ChangePlaybook`.
- `adoption_targets` (jsonb): baseline metrics for communication reach, training completion, adoption score.
- `governance_owner_group_id` (FK): change authority.
- `status` (enum): draft, published, superseded.
- `created_at` / `updated_at`
#### Relationships
- Provides default change scaffolding for transformations linked to a graph.
- Aligned with governance policies to ensure consistent escalation and reporting.
#### Lifecycle
- Authored by change authority → Published to graph consumers → Superseded as methodology evolves.

### ChangePlaybook
#### Fields
- `change_playbook_id` (UUID)
- `graph_id` (FK)
- `name` / `description`
- `steps` (jsonb): ordered tasks, triggers, personas, and expected outcomes.
- `cadence_guidance` (jsonb): recommended communication/training frequencies.
- `artifact_templates` (array<FK>): links to RepositoryArtifacts (communications, training packs, feedback forms).
- `active` (boolean)
- `version` (string)
- `created_at` / `updated_at`
#### Relationships
- Referenced by `ChangeManagementProfile` and `ChangePlan`.
- Integrated with governance playbooks for escalation alignment.
#### Lifecycle
- Draft → Approved → Published → Iterated; old versions retained for audit.

### ChangePlan
#### Fields
- `change_plan_id` (UUID)
- `transformation_id` (FK)
- `change_playbook_id` (FK, optional): base playbook applied.
- `name` / `description`
- `scope` (jsonb): impacted stakeholder segments, capabilities, and systems.
- `milestones` (jsonb): timeline with key communications/training events.
- `success_metrics` (jsonb): adoption KPIs, survey targets, sentiment thresholds.
- `status` (enum): draft, scheduled, in_progress, completed, retired.
- `owner_id` (FK): change manager accountable.
- `governance_owner_group_id` (FK, optional): escalation authority.
- `created_at` / `updated_at`
#### Relationships
- Seeds `CommunicationPlan`, `TrainingPackage`, and `ChangeTask` records.
- Linked to `ChangeManagementProfile` for default expectations.
- References RepositoryArtifacts and `ElementAttachment` for canonical guidance.
- Surfaces to `OutcomeMetricSnapshot` to correlate adoption progress with business outcomes.
#### Lifecycle
- Drafted during transformation planning → Activated as milestones commence → Updated as adoption feedback arrives → Closed with lessons learned and telemetry handoff.

### StakeholderSegment
#### Fields
- `segment_id` (UUID)
- `transformation_id` (FK)
- `segment_key` (string): label for the audience (e.g., Solution Architects, Field Ops, Stakeholders).
- `population_size` (int, optional)
- `impact_level` (enum): high, medium, low.
- `change_readiness` (enum): unaware, aware, ready, resistant.
- `primary_channels` (jsonb): preferred communication mechanisms.
- `owned_by` (FK): change manager responsible.
- `created_at` / `updated_at`
#### Relationships
- Tied to `ChangeImpactProfile`, `CommunicationPlan`, and `TrainingPackage`.
- Links to RepositoryElements (e.g., role definitions) via `ElementAttachment`.
#### Lifecycle
- Identified during change impact assessment → Updated with readiness surveys → Archived post-adoption.

### ChangeImpactProfile
#### Fields
- `impact_profile_id` (UUID)
- `transformation_id` (FK)
- `segment_id` (FK)
- `element_id` (FK, optional): RepositoryElement impacted.
- `impact_severity` (enum): negligible, moderate, significant, transformational.
- `impact_description` (text)
- `mitigation_actions` (jsonb): recommended responses.
- `effective_window` (daterange)
- `status` (enum): identified, mitigation_in_progress, mitigated, residual_risk.
- `created_at` / `updated_at`
#### Relationships
- Derived from transformation scope and RepositoryElement topology.
- Supports risk and adoption dashboards; informs `ChangeTask` backlog.
#### Lifecycle
- Created during assessment → Updated as mitigations executed → Closed when residual risk accepted.

### CommunicationPlan
#### Fields
- `communication_plan_id` (UUID)
- `transformation_id` (FK)
- `segment_id` (FK, optional): audience targeting.
- `cadence` (enum): weekly, biweekly, monthly, ad_hoc.
- `channels` (jsonb): email, townhall, intranet, threads, etc.
- `message_framework` (jsonb): key messages, call-to-action, links to RepositoryArtifacts.
- `schedule` (jsonb): send dates, owners.
- `status` (enum): planned, scheduled, sent, completed.
- `created_at` / `updated_at`
#### Relationships
- References artifact templates (FAQs, slide decks, release notes) via `TraceLink` or `ElementAttachment`.
- Feeds adoption telemetry by recording delivery metrics.
#### Lifecycle
- Authored during planning → Executed per schedule → Closed after metrics captured.

### TrainingPackage
#### Fields
- `training_package_id` (UUID)
- `transformation_id` (FK)
- `segment_id` (FK, optional)
- `training_type` (enum): instructor_led, self_paced, lab, webinar, microlearning.
- `delivery_mode` (enum): in_person, virtual, blended.
- `artifact_id` (FK, optional): RepositoryArtifact referencing the training materials.
- `prerequisites` (jsonb)
- `completion_criteria` (jsonb): quiz, attendance, practical assessment.
- `schedule` (jsonb)
- `status` (enum): planned, published, delivered, retired.
- `created_at` / `updated_at`
#### Relationships
- Linked to adoption metrics (training completion) and change tasks.
- Connects to RepositoryElements to show which capabilities/processes training covers.
#### Lifecycle
- Designed → Published → Delivered → Archived or refreshed.

### ChangeTask
#### Fields
- `change_task_id` (UUID)
- `transformation_id` (FK)
- `impact_profile_id` (FK, optional)
- `task_type` (enum): communication, training, feedback, mitigation, reinforcement.
- `description` (text)
- `owner_id` (FK)
- `due_date` (timestamp)
- `status` (enum): planned, in_progress, completed, blocked, cancelled.
- `linked_artifact_version_id` (FK, optional): deliverable evidence.
- `created_at` / `updated_at`
#### Relationships
- Integrated with transformation task boards; surfaced in blockers/adoption funnels.
- May trigger governance actions if overdue or critical.
#### Lifecycle
- Scheduled → In Progress → Completed; retains history for audit.

### AdoptionSignal
#### Fields
- `adoption_signal_id` (UUID)
- `transformation_id` (FK)
- `segment_id` (FK, optional)
- `signal_type` (enum): feature_usage, sentiment, support_ticket_volume, feedback_score, readiness_assessment.
- `captured_at` (timestamp)
- `metric_payload` (jsonb): raw measurement data with units/context.
- `source_system` (string): telemetry, survey tool, support desk.
- `trend_indicator` (enum): improving, stable, declining.
- `linked_benefit_hypothesis_id` (FK, optional)
- `created_at` / `updated_at`
#### Relationships
- Feeds `OutcomeMetricSnapshot` and change dashboards.
- Supports automated triggers for communication/training adjustments.
#### Lifecycle
- Recorded on cadence → Analysed alongside outcome metrics → Retained for trend analysis.

## Outcome & Telemetry Layer

### OutcomeMetricSnapshot
#### Fields
- `outcome_snapshot_id` (UUID)
- `transformation_id` (FK, optional)
- `strategic_theme_id` (FK, optional)
- `captured_at` (timestamp)
- `benefit_hypothesis_id` (FK, optional)
- `outcome_metrics` (jsonb): KPI name, actual, target, variance, unit.
- `adoption_metrics` (jsonb): communication reach, training completion, adoption score, sentiment.
- `financial_metrics` (jsonb): spend to date, forecast, variance.
- `linked_graph_statistic_snapshot_id` (FK, optional)
- `linked_governance_metric_snapshot_id` (FK, optional)
- `linked_transformation_metric_snapshot_id` (FK, optional)
- `confidence_score` (decimal, optional): quality of measurement.
- `anomaly_flags` (jsonb): breaches, trends requiring attention.
- `created_at` / `updated_at`
#### Relationships
- Harmonises strategic, governance, and transformation telemetry for SJ3/TJ4 reporting.
- Drives steering dashboards and quarterly architecture reviews.
- Provides input to `FundingGuardrail` evaluations and benefit realisation audits.
#### Lifecycle
- Generated on schedule or event-driven ingestion → Compared to targets → Archived for historical analysis.

### OutcomeDecision
#### Fields
- `outcome_decision_id` (UUID)
- `outcome_snapshot_id` (FK)
- `decision_type` (enum): continue, pivot, pause, stop, accelerate.
- `decision_date` (timestamp)
- `decision_body` (enum): steering_committee, sponsor, architecture_board.
- `rationale` (text)
- `follow_up_actions` (jsonb): tasks or change measures.
- `created_by` (FK)
#### Relationships
- Linked to `GovernanceRecord` or Go/No-Go artifacts for traceability.
- Updates transformation and funding guardrail status.
#### Lifecycle
- Recorded after review sessions → Follow-up tracked via change/governance workflows → Archived for audit.

## Integration & Traceability Notes
- All strategy and change entities leverage `TraceLink` to connect to RepositoryArtifacts, RepositoryElements, and transformations.
- Change constructs reuse governance routing mechanisms for escalations (e.g., overdue change tasks can trigger `GovernanceNotification`).
- Telemetry ingestion should feed `OutcomeMetricSnapshot` concurrently with `RepositoryStatisticSnapshot`, `GovernanceMetricSnapshot`, and `InitiativeMetricSnapshot` to maintain common cadence.

## Governance Considerations
- Strategy themes, benefit hypotheses, and guardrails require approval workflows; integrate with `GovernanceOwnerGroup` for accountability.
- Change profiles and playbooks should undergo yearly review; versioning stored on the entity and reinforced by graph governance policies.
- Adoption signals must respect data privacy and retention policies; ensure aggregation/sanitisation rules documented in change playbooks.
