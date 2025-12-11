# Backlog 230 — E2E User Journeys Index

## Purpose
This backlog series documents end-to-end user journeys that validate AMEIDE's governance workflows from a user's perspective. Each journey exercises multiple features, crosses graph/transformation boundaries, and verifies the complete lifecycle from user intent through system state changes, notifications, audit trails, and metrics updates.

These journeys serve three purposes:
1. **Acceptance criteria** for Product Owners validating complete feature delivery
2. **E2E test scenarios** for QA teams automating user workflows validation
3. **Onboarding narratives** for new users understanding how AMEIDE operates

---

## Personas

### Alice — Enterprise Architect
**Role**: Lead Architect, Asset Creator
**Goals**: Create high-quality architecture artifacts, publish to graph, maintain compliance
**Pain Points**: Manual review processes, unclear approval status, lost work due to policy changes
**AMEIDE Access**: Contributor role across multiple repositories; Author role in transformations
**Key Workflows**: Draft assets → Submit for review → Address feedback → Publish to graph → Track consumption

### Bob — Governance Reviewer
**Role**: Architecture Review Board Member
**Goals**: Ensure artifacts meet standards, provide timely feedback, maintain quality gates
**Pain Points**: Review backlogs, unclear SLA deadlines, insufficient context for decisions
**AMEIDE Access**: Reviewer role in GovernanceOwnerGroup; Read access to all repositories
**Key Workflows**: Review queue triage → Execute required checks → Provide decisions → Escalate blockers

### Carol — Program Manager
**Role**: Transformation Initiative Lead
**Goals**: Deliver transformations on time, consume graph assets, track readiness
**Pain Points**: Asset discovery, dependency tracking, unclear approval timelines
**AMEIDE Access**: Program Manager role in transformations; Read access to enterprise graph
**Key Workflows**: Create transformations → Reference baseline assets → Publish deliverables → Monitor readiness

### David — Governance Administrator
**Role**: Chief Architect, Governance Lead
**Goals**: Define policies, monitor compliance, optimize review processes
**Pain Points**: Policy impact analysis, reviewer workload balancing, compliance drift
**AMEIDE Access**: Admin role across repositories; Owner of GovernanceOwnerGroups
**Key Workflows**: Configure policies → Monitor metrics → Handle escalations → Run compliance audits

### Eve — Standards Authority
**Role**: Enterprise Standards Lead
**Goals**: Define and maintain standards, ensure compliance, track adoption
**Pain Points**: Standard proliferation, compliance tracking, evidence validation
**AMEIDE Access**: Standards Owner role; Creator of Standard assets
**Key Workflows**: Publish standards → Run compliance audits → Validate evidence → Track adoption metrics

### Frank — Business Stakeholder
**Role**: VP of Business Unit, Executive Sponsor
**Goals**: Understand architecture landscape, validate alignment with strategy, track transformation progress
**Pain Points**: Technical complexity, lack of business-friendly views, delayed insights
**AMEIDE Access**: Read-only viewer; Access to landscape visualizations and dashboards
**Key Workflows**: View landscape toggles → Compare baseline/target → Export reports → Monitor transformation progress

---

## Journey Map Overview

| Journey ID | Journey Name | Duration | Primary Personas | Key Features |
|------------|-------------|----------|------------------|--------------|
| [231](230-journey-01-draft-to-baseline.md) | New Architecture Vision — Draft to Baseline | 5-10 min | Alice, Bob | Asset lifecycle, review workflows, classification, approvals |
| [232](230-journey-02-policy-recertification.md) | Policy Update Triggers Recertification | 8-12 min | David, Alice, Bob | Policy management, background jobs, bulk recertification |
| [233](230-journey-03-transformation-workflows.md) | Initiative Consumes Repository Assets | 10-15 min | Carol, Alice, Bob | Initiative management, asset references, deliverable publishing |
| [234](230-journey-04-compliance-violation.md) | Compliance Violation & Remediation | 6-8 min | Eve, Alice, Bob | Compliance audits, violation tracking, evidence validation |
| [235](230-journey-05-landscape-toggle.md) | Architecture Landscape Toggle (Baseline ↔ Target) | 3-5 min | Frank, Alice | Landscape views, traceability, scenario modeling |
| [236](230-journey-06-sla-escalation.md) | SLA Breach & Escalation | 4-6 min | Bob, David | SLA monitoring, escalation workflows, reassignment |
| [237](230-journey-07-bulk-import.md) | Bulk Asset Import & Classification | 8-10 min | Alice, David | Bulk operations, background jobs, validation |
| [238](230-journey-08-data-quality-survey.md) | Data Quality Survey & Follow-up | 6-8 min | Eve, Alice | Survey issuance, responses, recertification triggers |
| [239](230-journey-09-retention-hold.md) | Retention Hold & Purge Workflow | 7-9 min | Carol, Alice | Legal hold, retention events, purge execution |
| [240](230-journey-10-governance-waiver.md) | Governance Waiver Lifecycle | 6-8 min | David, Bob | Waiver request/approval, expiry monitoring |
| [241](230-journey-11-release-package-go-no-go.md) | Release Package & Go/No-Go | 10-12 min | Carol, Frank | Release readiness, steering decision, outcome logging |
| [242](230-journey-12-ontology-stewardship.md) | Ontology Update & Element Governance | 9-11 min | Eve, Bob | Catalogue upgrade, cardinality enforcement, associations |
| [243](230-journey-13-cross-graph-collaboration.md) | Cross-Repository Collaboration | 8-10 min | Alice, Carol | CrossRepositoryReference approvals, multi-repo telemetry |
| [244](230-journey-14-theme-outcome-review.md) | Theme-Level Outcome Review | 7-9 min | Frank, PMO Analyst | Strategic theme telemetry, outcome decisions |

---

## Cross-Journey Dependencies

### Prerequisite Setup
All journeys assume the following baseline configuration:
- **Repository**: "Enterprise Repository" with TOGAF classifications configured
- **Owner Groups**: "Enterprise Architecture Board" (2 approvals required)
- **Policies**: "Security & Privacy Policy", "Data Residency Standard"
- **Roles**: Contributor, Reviewer, Admin roles assigned appropriately
- **Checks**: Metadata completeness, TOGAF compliance, Security scan (blocking)

### Journey Sequencing
Recommended execution order for E2E validation:
1. **Journey 231** (Draft to Baseline) — Establishes foundation
2. **Journey 233** (Initiative Workflow) — Consumes baseline from Journey 231
3. **Journey 232** (Policy Recertification) — Affects assets from Journey 231
4. **Journey 234** (Compliance Violation) — Tests remediation workflows
5. **Journey 235** (Landscape Toggle) — Views baseline + target from Journeys 231 & 233
6. **Journey 236** (SLA Escalation) — Tests governance operations
7. **Journey 237** (Bulk Import) — Tests scale operations
8. **Journey 238** (Data Quality Survey) — Exercises metadata remediation loop
9. **Journey 239** (Retention Hold & Purge) — Validates retention controls
10. **Journey 240** (Governance Waiver) — Demonstrates policy exception handling
11. **Journey 241** (Release Package & Go/No-Go) — Confirms release governance
12. **Journey 242** (Ontology Stewardship) — Updates ontology catalogue safely
13. **Journey 243** (Cross-Repository Collaboration) — Shares assets across repositories
14. **Journey 244** (Theme Outcome Review) — Reviews strategic outcomes end-to-end

### Data Lineage Across Journeys
```
Journey 231 (Alice creates Baseline)
    ↓
Journey 233 (Carol references Baseline, creates Target)
    ↓
Journey 235 (Frank compares Baseline ↔ Target)

Journey 232 (David updates Policy)
    ↓
Journey 234 (Eve runs Compliance Audit on affected assets)
```

---

## Acceptance Test Matrix

| Journey | Repository FR | Initiative FR | Governance FR | Sync/Metrics FR |
|---------|---------------|---------------|---------------|-----------------|
| 231 | FR-11, FR-13 | — | FR-12, FR-14, FR-15 | FR-18 (sync) |
| 232 | FR-13, FR-20 | — | FR-16, FR-20 | FR-18 (metrics) |
| 233 | FR-11, FR-18 | FI-01, FI-03 | FR-15 | FR-18 (sync) |
| 234 | FR-13, FR-20 | — | FR-14, FR-16 | FR-18 (compliance) |
| 235 | FR-18 | FI-03 | — | FR-18 (views) |
| 236 | FR-13 | — | FR-14, FR-15 | FR-18 (SLA) |
| 237 | FR-11 | — | FR-13 | FR-18 (bulk sync) |
| 238 | FR-20 | — | FR-13 | FR-20 (quality telemetry) |
| 239 | FR-20 | — | FR-13 | FR-20 (retention) |
| 240 | FR-13 | — | FR-13, FR-14 | FR-20 (waiver audit) |
| 241 | FR-11, FR-18 | FI-03 | FR-15 | FR-18 (release telemetry) |
| 242 | FR-18 | — | FR-20 | FR-18 (ontology) |
| 243 | FR-11, FR-18 | FI-03 | FR-13 | FR-18 (multi-repo telemetry) |
| 244 | FR-15 | FI-03 | FR-20 | FR-20 (theme telemetry) |

---

## KPIs & Telemetry Coverage

### Governance KPIs (captured across journeys)
- **Approval Cycle Time** (P50/P95) — Journeys 231, 232, 234, 236
- **Queue Wait Time** (P95) — Journeys 231, 236
- **Check Pass Rate** — Journeys 231, 232, 234
- **SLA Breach Count** — Journey 236
- **Data Quality Score** — Journey 232, 234

### Repository KPIs (captured across journeys)
- **Search Performance** (P95 < 2.5s) — Journey 237
- **Sync Latency** (P95) — Journeys 231, 232, 233, 237
- **Release Adoption Rate** — Journey 233
- **Classification Accuracy** — Journey 231, 237

### Initiative KPIs (captured across journeys)
- **Reference Count** — Journey 233
- **Deliverable Completion** — Journey 233
- **Toggle Latency** (baseline ↔ target) — Journey 235
- **Readiness Score** — Journey 233

---

## Test Environment Requirements

### Minimum Data Setup
- **Users**: 6 (Alice, Bob, Carol, David, Eve, Frank)
- **Repositories**: 1 (Enterprise Repository)
- **Owner Groups**: 2 (EA Board, Standards Team)
- **Policies**: 2 (Security Policy, Data Residency Standard)
- **Checks**: 3 (Metadata, TOGAF, Security)
- **Classifications**: 7 (SIB, Landscape.Baseline, Landscape.Target, etc.)

### Infrastructure Requirements
- **Background Jobs**: Worker pool for recertification, check execution, sync jobs
- **Notifications**: Email + in-app notification delivery
- **Search Index**: Full-text search capability
- **Audit Log**: Time-series storage for 30+ days
- **Metrics Store**: Time-series database for KPI tracking

### Test Data Reset
Each journey should support:
- **Isolation**: Independent test database/tenant
- **Cleanup**: Automatic teardown after test completion
- **Repeatability**: Idempotent setup via test fixtures
- **Debugging**: Snapshot capability for failure analysis

---

## Usage Notes

### For Product Owners
- Each journey maps to specific FR/FI requirements (see Acceptance Test Matrix)
- KPI targets are specified in each journey document
- Use journeys to validate Definition of Done for implementation stages

### For QA Engineers
- Journey documents provide step-by-step test scripts
- Test assertions are specified at end of each journey
- Playwright/Cypress code examples can be generated from steps
- See [CI automation patterns](220-ci-automation-patterns.md) for test infrastructure guidance

### For Architects
- Journeys demonstrate integration points between graph/transformation contexts
- State transitions and audit trails are explicitly documented
- Notification and sync workflows are validated end-to-end

### For New Users
- Read journeys in sequence (231 → 237) for comprehensive onboarding
- Each journey includes "Why this matters" context
- Personas provide relatable user perspectives

---

## Related Documentation
- [220 — AMEIDE Information Architecture](220-ameide-information-architecture.md) — Entity model
- [220 — Asset Entity Model](220-asset-entity-model.md) — Asset lifecycle details
- [220 — Governance Entity Model](220-governance-entity-model.md) — Governance workflows
- [220 — Repository Entity Model](220-graph-entity-model.md) — Repository structure
- [150 — Enterprise Repository Business Scenarios](150-enterprise-graph-business-scenario.md) — PO-first scenarios
- [151 — Enterprise Repository Functional](151-enterprise-graph-functional.md) — Functional requirements
- [220 — Continuous Delivery Automation Patterns](220-ci-automation-patterns.md) — Testing patterns

---

## Revision History
- **2025-02-XX**: Added journey documents 232–244, CI automation reference, and extended coverage (surveys, retention, waivers, release, ontology, cross-repo, theme review).
- **2025-01-XX**: Initial creation — 7 journeys documented
- **Future**: Extend coverage for API-first automation and negative paths as needed.
