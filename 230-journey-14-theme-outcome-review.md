# Journey 244 — Theme-Level Outcome Review

## Overview
**Duration**: 7–9 minutes  
**Primary Personas**: Frank (Sponsor), PMO Analyst  
**Supporting Personas**: Carol (Product Owner), Finance Partner  
**Complexity**: Medium

**Business Value**: Reviews business outcomes at the strategic theme level, correlating `OutcomeMetricSnapshot` entries with graph/transformation telemetry so leadership can steer investments without drilling into each transformation individually.

---

## Prerequisites
- Strategic theme `Digital Efficiency` with associated transformations (3 active, 1 completed).
- Theme-level `OutcomeMetricSnapshot` data ingested (financial variance, adoption index).
- Telemetry join supports `strategic_theme_id` key.
- Dashboards configured with theme filter.

---

## Journey Steps

### Phase 1: Load Theme Dashboard (PMO Analyst)
1. Analyst opens Portfolio dashboard, selects theme `Digital Efficiency`.
2. Dashboard loads aggregated metrics from `OutcomeMetricSnapshot`, `GovernanceMetricSnapshot`, `RepositoryStatisticSnapshot`, and `InitiativeMetricSnapshot` keyed by `strategic_theme_id`.

**Expected Outcome**
- Telemetry join row retrieved with theme ID, aggregated deltas.
- Baseline vs current variance displayed for financial, adoption, governance KPIs.

**Test Assertion**
```typescript
const joinRow = await db.findOne('telemetry_join_view', {
  strategic_theme_id: digitalEfficiencyTheme.id,
  captured_at: currentWindow
});
expect(joinRow.outcome_metrics.benefit_variance).toBeDefined();
expect(joinRow.transformation_ids.length).toBeGreaterThan(0);
```

### Phase 2: Drill into Theme Variances (Frank)
1. Frank reviews benefit variance chart and selects month with negative variance.
2. Dashboard reveals contributing transformations and key blockers (governance breaches, adoption lag).
3. Sponsor raises questions assigned to respective Product Owners via `OutcomeDecision` follow-up.

**Expected Outcome**
- Drill-down shows transformations sorted by variance contribution.
- Follow-up actions captured in theme-level `OutcomeDecision` records.

### Phase 3: Approve Adjustments
1. Frank approves mitigation plan (e.g., extend training program, accelerate automation).
2. `OutcomeDecision` stored (`decision_type=accelerate`) linked to strategic theme.
3. Change tasks generated for impacted transformations and change managers notified.

**Expected Outcome**
- Decision recorded with `strategic_theme_id` and follow-up tasks.
- Notifications dispatched (`notification_type=review_outcome`) to transformation owners.
- Governance dashboard updates action tracker.

### Phase 4: Verify Post-Decision Metrics
1. Analyst schedules follow-up snapshot to measure impact.
2. After cadence, new `OutcomeMetricSnapshot` shows variance improvement.
3. Telemetry join updates trend charts for theme-level reporting.

**Expected Outcome**
- Updated snapshot with improved variance.
- Trend line shows positive trajectory.
- Portfolio summary exports highlight decision impact.

---

## Telemetry & KPIs
- Theme-level benefit variance (target within ±5%).
- Adoption index across theme.
- Number of outstanding actions per theme.
- Time to close theme-level follow-up tasks.

---

## FR/NFR Coverage
- **FR-20**: Theme-level telemetry alignment.
- **FR-15**: Readiness/variance reporting aggregated by theme.
- **NFR-Obs-01**: Multi-dimensional telemetry joins.
- **NFR-Res-01**: Ability to track follow-up actions and verify effect.

---

## Related Journeys
- **Journey 235**: Initiative-level scenario toggling feeding theme metrics.
- **Journey 241**: Release package decisions feed theme outcome review.
- **Journey 246**: Strategy & adoption journeys leverage theme dashboards.

---

## Revision History
- **2025-02-XX**: Initial draft capturing theme-level outcome review.
