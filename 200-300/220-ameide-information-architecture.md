# Backlog 220 — Ameide Information Architecture

## Domain Overview
Ameide is an enterprise architecture platform composed of three cooperating surfaces: applications where operational work is executed, repositories that curate reusable architecture knowledge, and artifacts that narrate that knowledge. Repositories are one feature of the broader platform—they manage curated collections of RepositoryElements, classifications, policies, and telemetry. Applications remain the system of record for operational data, while artifacts translate that data into architectural context that can be governed, shared, and published.

## Platform Layers
- **Application layer**: Business applications manage transformations, controls, portfolios, and operational workflows. They expose record-centric UI, own transactional telemetry, and stay independent from ontology identifiers.
- **Repository feature**: Repositories capture intentionally curated knowledge—RepositoryElements, classifications, governance bundles, retention policy references, and sync status. They sit alongside other platform capabilities such as transformation workspaces and governance queues.
- **Artifact library**: Artifacts and ArtifactVersions provide reusable narratives, documents, and presentation bundles. They connect applications to repositories without binding the two data models directly.
- **Ontology & diagrams**: A lightweight ontology service registers element types and relationships that diagrams render. Diagram panels show RepositoryElements with backlinks to the artifacts that present them.
- **Governance service**: Policies, queues, review cases, and notification rules operate across artifact versions, graph elements, and release packages while maintaining an application-first reviewer experience.
- **Telemetry & analytics**: Snapshot tables and join views align application metrics, graph cadence, and diagram overlays using consistent keys so analytics reflect the same architecture story that artifacts present.

## Interaction Contracts
1. Applications emit or reference artifacts whenever architecture context is required.
2. Artifacts attach to RepositoryElements via `ElementAttachment`, enriching diagrams and knowledge views.
3. Repositories synchronise elements, classifications, governance state, and telemetry snapshots for downstream publishing to search, graph, and analytics consumers.
4. Diagrams read RepositoryElements from repositories and surface artifacts as the doorway back into applications.
5. Governance and telemetry services treat artifact versions and graph elements as review subjects while keeping application records in their native flows.

## Repository Feature
Repositories organise curated architecture knowledge.
- **Collections**: RepositoryElements (ontology-backed nodes), RepositoryElementRelationships, classifications, and OntologyAssociations live inside the graph feature.
- **Policies**: Each graph references a `GovernanceProfile`, retention policies, and placement rules that guide artifact lifecycle and diagram eligibility.
- **Visibility & access**: Repository-level visibility flags and membership roles define who can publish, review, or synchronise content.
- **Sync & publishing**: Repositories manage downstream sync to knowledge graph, search, and analytics services, ensuring diagrams and dashboards display the latest approved state.

RepositoryElements themselves are governed objects contained within the graph feature. They carry type, provenance, lifecycle flags, and relationship connectivity. Creation flows include manual curation, internal setup automations, and external publishers. Auto-mirror setups are opt-in per application record type; they map source fields, create or update RepositoryElements, and optionally generate presentation artifacts without altering application UI contracts.

## Artifact Library
Artifacts encapsulate human-readable context, documentation, evidence, and presentation bundles. ArtifactVersions manage lifecycle (`draft → in_review → rework → approved → retired`), classification placement, checks, and evidence attachments. Subtypes—roadmap, decision record, requirement spec, transformation brief, and others—tailor UI layouts and governance rules.

Key behaviours:
- Application records point to artifacts, never directly to RepositoryElements.
- ArtifactVersions attach to RepositoryElements using `ElementAttachment`, declaring presentation intent (primary, supporting, evidence).
- `TraceLink` connects ArtifactVersions to RepositoryElements and to application records for coverage, satisfaction, and accountability chains.

## Ontology & Diagram Service
- **OntologyCatalogue** registers element types, relationship verbs, and cardinality rules (for example `archimate_work_package`, `archimate_requirement`, platform extensions).
- **RepositoryElementRelationship** stores directed, typed connections with cardinality for validation and knowledge graph exports.
- **OntologyAssociation** captures semantic links such as `equivalent_to`, `realises`, or `constrained_by`.
- **Diagrams** render RepositoryElements only; element panels list referencing artifacts which in turn link back to application records.
- **Search** offers record and artifact discovery by default, while ontology search surfaces elements that route through artifacts for deeper exploration.

## Navigation & UX
- Application pages remain record-centric and do not embed raw element chips. When architecture context is needed, the UI links to the relevant artifact detail view.
- Artifact detail views display related RepositoryElements, evidence, and downstream impacts, acting as the bridge between applications and repositories.
- Diagram viewers display element topology, health overlays sourced from artifact telemetry, and backlinks to artifacts for deeper inspection.
- Global navigation provides entry points for applications, repositories, diagrams, and governance queues, keeping the user aware of the current surface.

## Governance & Policy Layer
- `GovernanceProfile` is a versioned bundle of policies, check catalogues, and playbooks; each graph references one active profile.
- Review subjects include `artifact_version`, `graph_element`, and `release_package`, with states `open`, `in_progress`, `on_hold`, `changes_requested`, `escalated`, and `resolved`.
- Notification vocabulary (`review_assignment`, `decision_request`, `resubmission`, `recertification`, `sla_breach`, etc.) is applied consistently across channels.
- Governance queues and dashboards key metrics to `graph_statistic_snapshot_id` for alignment with telemetry snapshots.
- Element-targeted checks surface through the artifacts referencing those elements so reviewers stay in the artifact context.

## Telemetry & Analytics
- Application telemetry keeps its native keys; repositories and diagrams rely on `TelemetryJoinView`, aligning snapshots on `(graph_id, work_package_element_id, captured_at)` with optional strategic theme dimensions.
- Diagram overlays summarise element health using artifact-sourced metrics; raw application records do not feed diagrams directly.
- Outcome, benefit, and theme analytics align to strategic themes and work-package elements using the same joins.

## Journeys & Workflows
- **Draft to Baseline (231)**: Authoring occurs in applications; artifacts capture the narrative, route through review, and promote into graph classifications once approved.
- **Recertification (232)**: GovernanceProfile updates reopen relevant artifact versions and graph elements with refreshed checks while dashboards report backlog and clearance trends.
- **Initiative Workflow (233)**: Boards operate on transformation records. When graph elements exist, diagrams offer architecture views linked back through transformation artifacts.
- **Compliance Violation (234)**: Violations originate in governance queues, remediation evidence is recorded in artifacts, and successful closure updates graph telemetry.
- **Landscape Toggle (235)**: Baseline/target/transition views rely on classification assignments and ArchitectureState aggregates. Element drill-downs rely on artifacts for presentation and navigation.
- **SLA Escalation (236)**: Queue metrics capture breaches, reassignment, and MTTR; telemetry snapshots provide consistent reporting identifiers.
- **Bulk Import (237)**: Import pipelines accept artifact packages that reference graph elements when provided, emitting artifacts, attachments, and trace links that follow graph placement rules.

## Optional Automations
- **Auto-mirror setup**: Configured per record type, specifying ontology type, identity mapping, and optional artifact template. Synchronises RepositoryElements and optional presentation artifacts on eligible record changes.
- **External publishing**: Third-party tools may publish elements or artifacts through curated APIs with provenance metadata that passes OntologyCatalogue validation.
- **Internal curation**: Administrators can seed repositories by scripting element creation, relationship wiring, and artifact scaffolding without altering application schemas.

## Implementation Considerations
- Maintain OntologyCatalogue entries and documentation for supported element types and relationships.
- Ensure telemetry pipelines honour the shared join keys and feed diagram overlays through artifact-derived metrics.
- Keep governance templates current with notification vocabulary, state transitions, and subject polymorphism.
- Provide admin tooling and audit trails for auto-mirror mappings, element provenance, and artifact publication.
- Update meta-model diagrams to depict applications, repositories, and artifacts as peer layers with well-defined interaction contracts.
