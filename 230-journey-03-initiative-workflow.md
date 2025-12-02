# Journey 233 — Initiative Consumes Repository Assets

## Overview
**Duration**: 10–15 minutes  
**Primary Personas**: Carol (Program Manager), Alice (Enterprise Architect), Bob (Reviewer)  
**Supporting Personas**: Sponsor (Frank), Automation Bot  
**Complexity**: High

**Business Value**: Validates how transformations source baseline architecture from the graph, plan new deliverables, and keep governance and telemetry aligned throughout execution.

---

## Prerequisites

### System Configuration
- Enterprise Repository configured with TOGAF classifications and governance profile v1.1.
- Initiative templates aligned to ADM phases with default milestones and required checks.
- ChangeManagementProfile and ChangePlaybook published for the graph.
- Strategy theme “Digital Transformation 2025” available with funding guardrails.

### User Setup
- **Carol**: Program Manager with transformation admin rights.
- **Alice**: Lead Architect assigned to transformations and graph contributor.
- **Bob**: Governance reviewer with visibility into transformation blockers.
- **Frank**: Sponsor with read access to dashboards.

### Seed Data
- Baseline asset “2025 Digital Transformation Vision” (approved, baseline).
- RepositoryElements representing key capabilities and value streams.
- Initiative backlog empty; metrics snapshots scheduled daily.

---

## User Story
**As** Carol (Program Manager),  
**I want** to launch an transformation that reuses existing graph assets and plans new deliverables,  
**So that** we can execute change with clear governance, telemetry, and adoption coverage.

---

## Journey Steps

### Phase 1: Initiative Launch (Carol)
1. Carol selects “ADM Program” template, providing name, description, target start/end dates.
2. She links the transformation to “Enterprise Repository” and aligns to StrategicTheme “Digital Transformation 2025”.
3. System seeds InitiativeMilestones (Vision, Architecture Definition, Roadmap, Implementation), default roles, and governance banner summarising required checks and SLA expectations.
4. Carol assigns roles: Sponsor (Frank), Product Owner (Carol), Lead Architect (Alice), Governance Reviewer (Bob).
5. Invitations sent; acceptance recorded in `InitiativeRoleAssignment`.

**Expected Outcome**
- Initiative status `proposed`, roles covered ≥ 80%.
- Governance banner displays policy coverage, default checks, and change profile.
- Initial InitiativeMetricSnapshot captured with baseline metrics.

### Phase 2: Scope from Repository (Alice)
1. Alice opens the graph browser inside the transformation workspace.
2. She references baseline asset “2025 Digital Transformation Vision” via `InitiativeReference` (type=`baseline_reference`) for milestone Vision.
3. She navigates capability map (ElementProjection) selecting impacted RepositoryElements; these feed scope heatmap and adoption planning.
4. Alice creates draft deliverable assets (e.g., “Target Application Landscape”) flagged as `draft` with scenario tag `to_be`.
5. `TraceLink` relationships established between new drafts and baseline assets; classification suggestions recorded.

**Expected Outcome**
- Initiative board shows baseline vs new work ratio.
- Scope coverage metrics update (capabilities referenced vs targeted).
- ChangeImpactProfile stubs created for affected stakeholder segments.

### Phase 3: Governance Alignment (Bob & Carol)
1. Bob reviews blockers funnel filtered to transformation scope; ensures required checks align with graph policies.
2. Carol configures readiness checklist (RequiredChecks subset) for upcoming milestones.
3. Automation bot simulates check coverage, highlighting missing evidence or classification gaps.
4. Carol updates milestone readiness criteria referencing RequiredChecks and TracePolicy expectations.

**Expected Outcome**
- Milestone readiness meter reflects configured checks.
- Governance notifications scheduled for milestone due dates.
- `GovernanceMetricSnapshot` notes transformation-specific coverage.

### Phase 4: Execution & Telemetry
1. Alice progresses draft deliverables through review (reusing Journey 231 mechanics).
2. As approvals land, `InitiativeMetricSnapshot` updates with readiness %, blockers resolved, asset references.
3. Carol monitors outcomes: blockers funnel, SLA timers, adoption signals seeded from ChangeManagementProfile.
4. Frank views scenario toggle (As-Is vs To-Be) to validate benefit trajectory.

**Expected Outcome**
- Approved deliverables feed graph; InitiativeReference updates to approved version IDs.
- Telemetry join (TJ3) reconciles transformation/graph metrics for reporting.
- Funding guardrails evaluated; no breaches recorded.

### Phase 5: Closeout & Handover
1. Carol generates Go/No-Go package summarising approvals, checks, outcome metrics, and change readiness.
2. Sponsor approves release; `OutcomeDecision` recorded as `continue`.
3. Initiative stage set to `completed`; cleanup tasks schedule retention and adoption reinforcement activities.

**Expected Outcome**
- Final metrics captured (readiness score, blockers resolved, adoption health).
- Governance and strategy dashboards updated with completion status.
- Lessons learned logged as `GovernanceRecord` for future iterations.

---

## Telemetry & KPIs
- **Scope Coverage %** (baseline assets referenced vs targeted).
- **Readiness Score** per milestone (target ≥ 80% before Go/No-Go).
- **Blocker Count & Resolution Time** (aligned with FR-13/14).
- **Sync Latency** for approved deliverables (target P95 < 30s).
- **Adoption Health Index** seeded from ChangeImpactProfiles.

---

## FR/NFR Coverage
- **FR-6**: Asset registration/resuse within transformation.
- **FR-11**: Asset lifecycle integration.
- **FR-13/14**: Governance queue and SLA monitoring.
- **FR-15**: Readiness reporting.
- **FR-18**: Classification and scenario tagging.
- **FI-01/FI-03**: Initiative management requirements.
- **NFR-Perf-02**: Sync latency.
- **NFR-Audit-01**: Traceability between transformation and graph artefacts.

---

## Related Journeys
- **Journey 231**: Source baseline asset for scope.
- **Journey 232**: Recertification path triggered by policy changes.
- **Journey 235**: Landscape toggle that consumes transformation deliverables.
- **Journey 246**: Tracks adoption aligned to transformation outputs.

---

## Revision History
- **2025-02-XX**: Initial documentation of transformation workflows journey.
