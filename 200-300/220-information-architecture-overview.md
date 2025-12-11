# 220 — Information Architecture Overview

## Scope & Intent
- Frame the core domains (Repository, Governance, Artifacts & Initiatives, CI Automation) that compose Ameide's tenant-scoped architecture platform.
- Highlight how policy bundles, catalog metadata, and operational evidence circulate across domains to keep governance, delivery, and observability aligned.
- Provide a reference layout for downstream UX, integration, and analytics work that sits above the detailed entity models.

## Domain Pillars

### Repository Foundation
- Anchors the tenant knowledge base and seeds defaults (visibility, SLA, locale, timezone) consumed throughout the stack.
- Curates vocabularies and placement logic via `RepositoryClassification`, `PlacementRule`, and ontology-backed `RepositoryElement` catalogues.
- Coordinates downstream consistency using `RepositorySyncJob`, `RepositoryIntegrationEndpoint`, `ImportJob`, and `CrossRepositoryReference`.
- Captures operational posture in `RepositoryStatisticSnapshot` and sync states (knowledge graph, search, telemetry caches).
- Hosts governance baseline by binding `GovernanceProfile`, `SlaPolicy`, and `ChangeManagementProfile` bundles that transformations inherit.

### Governance Control Plane
- Manages ownership (`GovernanceOwnerGroup`), guardrails (`GovernancePolicy`, `PlacementRule`), and workflows playbooks that drive review execution.
- Orchestrates decision pipelines through `ReviewCase`, `ApprovalDecision`, `PromotionQueueEntry`, and `GovernanceWorkflowPlaybook`.
- Enforces checks and evidence via `RequiredCheck`, `CheckBinding`, `CheckRun`, and `DataQualitySurvey`, feeding recertification triggers.
- Routes signals and escalations across repositories using `GovernanceRoutingRule`, `SlaBreachEvent`, `GovernanceNotification`, and `GovernanceEscalation`.
- Provides telemetry snapshots (`GovernanceMetricSnapshot`, `RecertificationJob` logs) that synchronize with graph and transformation analytics.

### Artifact & Initiative Execution Layer
- Registers canonical architecture knowledge through `Artifact`, `ArtifactVersion`, subtype-specific schemas (requirement, standard, governance_record), and supporting references.
- Maintains ontology-backed `Element` entities (e.g., ArchiMate Business Actor, BPMN Process) that belong to repositories and provide reusable semantic anchors.
- Governs artifact placement with `ClassificationAssignment`, `TraceLink`, `ArtifactSyncPayload`, and retention/evidence events—artifacts reference elements to ground provenance and modelling detail.
- Connects delivery outcomes to portfolio efforts via `Initiative`, `InitiativeMilestone`, `InitiativeReference`, `InitiativeAlert`, and `InitiativeMetricSnapshot`.
- Ensures transformations share graph defaults (visibility, SLA, locale) while layering transformation-specific cadence, role assignments, and telemetry.
- Aligns change enablement by pairing transformations to `ChangeManagementProfile`, adoption signals, and outcome tracking.

### Automation & Telemetry Fabric
- Applies the CI blueprint (`Lint/Type`, `Unit/Contract`, `Journey E2E`, `Compliance Gates`, `Ephemeral Deploy`, `Promotion Hooks`) across repositories and transformations.
- Publishes pipeline evidence (SBOMs, scan reports, test summaries) as `TraceLink` attachments to `ArtifactVersion` records and governance records.
- Mirrors governance SLAs inside pipelines so breaches raise `SlaBreachEvent` and can trigger `RecertificationJob`.
- Streams observability metrics (build/test duration, gate outcomes, impacted journeys) into telemetry dashboards alongside graph/governance snapshots.
- Maintains synthetic monitors and automation debt registers to sustain coverage and regression defense.

## Cross-Domain Flows

### Artifact Onboarding & Governance
1. Repository defaults (visibility, SLA, owner groups) seed new `Artifact` records while surfacing relevant ontology-backed `Element` anchors for linkage.
2. Auto-placement/placement rules propose `ClassificationAssignment`; governance owner groups review via `ReviewCase` and confirm element references match modelling standards.
3. Required checks execute (`CheckRun`), with evidence stored and linked for audit; failures escalate through routing rules and notifications, and element coverage gaps raise targeted follow-ups.
4. Approved `ArtifactVersion` publishes via `RepositorySyncJob`, updating search, knowledge graph, and analytics caches alongside element relationship projections.

### Policy Change & Recertification
1. Governance teams update `GovernancePolicy` bundles inside `GovernanceProfile`.
2. Activated changes propagate to repositories and transformations; `CheckBinding` enforces new evidence expectations.
3. Recertification automation (`RecertificationJob`) reopens impacted artifacts and transformations, ensuring linked elements still comply with updated ontology and policy guidance; element-only impacts trigger element-focused recertifications, and alerts surface through `InitiativeAlert` and governance notifications.
4. Resulting decisions and evidence refresh graph and governance metric snapshots for telemetry parity.

### Continuous Delivery Evidence Loop
1. CI pipelines execute the automation blueprint, tagging runs with graph and transformation identifiers.
2. Compliance gates reuse governance-required checks, emitting results into `SlaBreachEvent` or `GovernanceNotification` when thresholds miss.
3. Successful runs deposit artefacts linked to `ArtifactVersion` or `GovernanceRecord`, closing the audit trail.
4. Telemetry streams align with `RepositoryStatisticSnapshot` and `InitiativeMetricSnapshot` so dashboards reflect delivery health in near real-time.

## Shared Services & Data Products
- **Knowledge propagation:** Repository sync orchestration publishes deltas to knowledge graph, search index, and archival stores to keep external consumers aligned.
- **Telemetry mesh:** Repository, governance, and transformation snapshots converge with CI observability metrics for SLA analytics, anomaly detection, and readiness reporting.
- **Access & identity:** Repository membership and governance owner groups coordinate role mapping, ensuring decision actors have consistent access across domains.
- **Retention & archival:** Repository-level retention policies drive artifact archival, governance record immutability, and import/export audit trails.
- **Ontology services:** Repository-managed element catalogues keep semantic anchors (e.g., ArchiMate actors, BPMN processes) versioned and reusable across artifacts and analytics.

## Experience Surfaces
- **Repository workspace:** Classification explorers, ontology-backed element libraries, policy bundles, and telemetry health cards sourced from graph and governance entities.
- **Governance workbench:** Review queues, check orchestration, routing configuration, and escalation tooling referencing owner groups, policies, and cases.
- **Initiative cockpit:** Backlog, milestone, and alert views enriched with artifact and element references, metric snapshots, and governance status.
- **Automation console:** Pipeline dashboards, synthetic monitors, and evidence ledgers tied back to governance SLAs and graph sync status.

## Entity Glossary
- **Repository:** Tenant-scoped workspace that anchors defaults (visibility, SLA, locale) and houses classifications, sync states, and integration orchestration.
- **RepositoryClassification:** Vocabulary node structuring placement hierarchies, governance ownership, and navigation ordering.
- **PlacementRule:** Constraint record that validates artifact placement, required evidence, and auto-classification behaviour.
- **GovernanceProfile:** Bundled policy set that activates governance defaults, workflows, and required checks for a graph.
- **GovernanceOwnerGroup:** Steward cohort responsible for reviews, approvals, and escalations across defined coverage scopes.
- **GovernancePolicy:** Guardrail definition capturing compliance expectations, required evidence, and linked workflows playbooks.
- **ReviewCase:** Execution record for governance reviews tracking decisions, SLA timers, and evidence attachments.
- **RequiredCheck:** Automated or manual control that validates policy compliance and produces evidence for review cases and CI gates.
- **RepositorySyncJob:** Orchestrated export pipeline that propagates graph deltas to knowledge graph, search, analytics, and archival systems.
- **Artifact:** Canonical architecture artefact storing graph knowledge, subtype metadata, and governance ownership.
- **ArtifactVersion:** Approved snapshot of an artifact that carries evidence, trace links, and sync payloads for downstream consumers.
- **Element:** Ontology-backed entity owned by a graph, specialised by modelling vocabulary (e.g., ArchiMate Business Actor, BPMN Process) and referenced by artifacts for semantic grounding.
- **TraceLink:** Relationship edge connecting artifacts, elements, transformations, evidence artefacts, and policy records for audit traceability.
- **Initiative:** Execution container for portfolio work that consumes artifacts, tracks milestones, and inherits graph defaults.
- **InitiativeMetricSnapshot:** Periodic telemetry sample summarizing transformation readiness, blockers, and SLA posture.
- **SlaBreachEvent:** Alert event emitted when review, check, or pipeline timers exceed thresholds, driving notifications and escalations.
- **RecertificationJob:** Automation run that revalidates artifacts and elements (and associated transformations) after policy changes or telemetry anomalies.
- **GovernanceNotification:** Communication record that routes decisions, breaches, and survey requests to stakeholders and tools.
- **CheckRun:** Concrete execution of a required check, capturing outcomes, evidence references, and SLA compliance.
- **RepositoryStatisticSnapshot:** Aggregated graph metrics aligning operational health with governance and transformation telemetry.
- **GovernanceEscalation:** Managed escalation record coordinating mitigation plans when governance workflows stall or breach SLAs.

## Implementation Considerations
- Sequence enabling work by hardening graph provisioning (defaults, classifications) before layering governance enforcement and transformation telemetry.
- Ensure data contracts between sync jobs, telemetry snapshots, and CI evidence share consistent identifiers (`graph_id`, `artifact_version_id`, `element_id`, `transformation_id`).
- Prioritize automation of audit trails (TraceLink, evidence artefacts) to keep policy, delivery, and telemetry narratives synchronized.
- Align UX and integration roadmaps to these domain pillars so teams can reason about responsibilities and data flows without diving into low-level entity specs.
