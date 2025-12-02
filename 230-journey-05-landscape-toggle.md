# Journey 235 — Architecture Landscape Toggle (Baseline ↔ Target)

## Overview
**Duration**: 3–5 minutes  
**Primary Personas**: Frank (Business Stakeholder), Alice (Enterprise Architect)  
**Supporting Personas**: Carol (Program Manager), Platform Analyst  
**Complexity**: Low

**Business Value**: Ensures stakeholders can easily compare baseline and target states, understand capability impacts, and rely on consistent telemetry sourced from graph assignments and transformation outputs.

---

## Prerequisites

### System Configuration
- Enterprise Repository populated with approved baseline and target AssetVersions.
- `ClassificationAssignment` records tagged with roles `baseline` and `target`, including effective windows.
- `ElementProjection` capability map published for both scenarios.
- Telemetry synchronization (knowledge graph + analytics) up to date.

### User Setup
- **Frank**: Sponsor with read-only access to dashboards.
- **Alice**: Architect providing context for scenario changes.
- **Carol**: Program Manager monitoring transformation metrics.
- Platform Analyst ensures dashboards functional.

### Seed Data
- Baseline asset “2025 Digital Transformation Vision”.
- Target asset “2026 Digital Operating Model” approved and classified.
- Initiative deliverables linked to target scenario, readiness data available.

---

## User Story
**As** Frank (Sponsor),  
**I want** to compare the current baseline with the proposed target architecture,  
**So that** I can understand scope, risk, and expected benefit before approving funding milestones.

---

## Journey Steps

### Phase 1: Dashboard Entry
1. Frank navigates to Strategy & Change dashboard.
2. Dashboard loads Telemetry Join view combining Repository, Governance, Initiative, and Outcome snapshots for selected transformation.
3. Default view shows baseline capability map (ElementProjection) with KPIs (readiness, adoption, benefit variance).

**Expected Outcome**
- Baseline scenario visible with metrics (e.g., capability health, blockers).
- Scenario toggle control enabled (Baseline, Target, Scenario X).

### Phase 2: Toggle to Target (Frank & Alice)
1. Frank switches toggle to “Target”. Dashboard queries either `ClassificationAssignment` entries where `role=target` and active window or consumes precomputed `ArchitectureState` snapshots for the target scenario.
2. Visualization updates to highlight capability deltas, new systems, and associated transformations.
3. Alice explains changes using overlay: new components, decommissioned systems, adoption signals.

**Expected Outcome**
- Delta view emphasises adds/removes/changes via color coding sourced from `ArchitectureState` diffs.
- Initiative readiness badges displayed next to impacted capabilities.
- Outcome metrics update to show expected vs actual benefit trend.
- Telemetry join includes `strategic_theme_id` when the selected view aggregates at theme scope.

### Phase 3: Drill-down & Evidence
1. Frank selects a capability to drill into detail panel showing:
   - Linked AssetVersions (baseline/target).
   - Associated transformations and milestones.
   - Governance status (checks, approvals, waivers).
   - Outcome metrics (benefit hypothesis progress, adoption signals).
2. Carol confirms milestone readiness; provides evidence of approvals via `GovernanceRecord` links.
3. Platform Analyst ensures telemetry timestamps within SLA (≤ 30 min).

**Expected Outcome**
- Stakeholder obtains confidence in data accuracy and freshness.
- Capability drill-down references canonical entities (AssetVersion, InitiativeReference, OutcomeMetricSnapshot, ArchitectureState).
- Any open blockers surfaced immediately.
- Telemetry panel shows both transformation-level and theme-level KPIs when available.

**Test Assertion**:
```typescript
const joinRow = await db.findOne('telemetry_join_view', {
  graph_id: enterpriseRepo.id,
  captured_at: currentWindow,
  transformation_id: transformation.id
});
expect(joinRow.strategic_theme_id).toBe(strategicTheme.id);
expect(joinRow.architecture_state_hash).toBe(targetState.state_hash);
```

### Phase 4: Export & Follow-up
1. Frank exports scenario comparison report (PDF/CSV) capturing deltas, KPIs, adoption summary.
2. Report attaches to Go/No-Go package (Journey 233 Phase 5).
3. Sponsor logs decision notes and questions for next steering meeting.

**Expected Outcome**
- Export stored as `EvidenceArtifact` for audit.
- OutcomeDecision ready with contextual data.
- Telemetry alert set if adoption/benefit variance drifts beyond tolerance.

---

## Telemetry & KPIs
- **Toggle Latency** (target < 300 ms cached, < 2 s uncached).
- **Capability Coverage** (target 100% of targeted capabilities visible).
- **Outcome Variance** (highlighted for each capability/theme combination).
- **Telemetry Freshness** (timestamps within SLA).
- **Adoption Signal Trend** (improving/stable/declining).

---

## FR/NFR Coverage
- **FR-18**: Time-aware metadata, scenario tagging, and projections.
- **FI-03**: Initiative metrics presented in context.
- **NFR-Perf-03**: Dashboard performance tolerances.
- **NFR-UX-01**: Readable stakeholder overlays with evidence.
- **NFR-Audit-01**: Export stored for traceability.

---

## Related Journeys
- **Journey 231**: Provides baseline assets referenced in view.
- **Journey 233**: Supplies target deliverables and readiness data.
- **Journey 235 (TJ3/TJ4)**: Telemetry operations ensuring data accuracy.
- **Journey 246**: Change and adoption storytelling for stakeholders.

---

## Revision History
- **2025-02-XX**: Initial documentation of baseline ↔ target toggle experience.
