# Journey 238 — Data Quality Survey & Follow-up

## Overview
**Duration**: 6–8 minutes  
**Primary Personas**: Eve (Governance Lead), Alice (Asset Owner)  
**Supporting Personas**: Automation Bot, David (Governance Lead backup)  
**Complexity**: Medium

**Business Value**: Exercises the metadata quality loop—issuing a targeted `DataQualitySurvey`, collecting `SurveyResponse`, triggering remediation, and confirming telemetry updates—so the organisation can systematically close metadata gaps before they turn into audit findings.

---

## Prerequisites

- Repository `Enterprise` with `GovernanceProfile` v1.1 active.
- Data quality survey template “Baseline Completeness” published.
- Classification `Landscape.Baseline` mapped to GovernanceOwnerGroup “Standards Team”.
- AssetVersion `baselineVision_v1` (approved) missing required metadata to trigger survey.
- Notification channels configured (email + in-app).

---

## Journey Steps

### Phase 1: Launch Survey (Eve)
1. Eve opens the governance dashboard, filters the quality widget to `Landscape.Baseline`.
2. She selects affected asset cohort (10 baselines) and clicks “Launch Data Quality Survey”.
3. Eve sets due date (7 days), escalation fallback (David), and adds reminder cadence (48h).
4. System creates `DataQualitySurvey` rows, links targeted assets, and dispatches `GovernanceNotification` (`notification_type=survey_due`) to asset owners.

**Expected Outcome**
- Survey status `issued`.
- `survey_target_count = 10`.
- Notifications recorded for each asset owner.
- `GovernanceMetricSnapshot` records survey issuance.

**Test Assertion**
```typescript
const survey = await db.findOne('data_quality_surveys', {
  graph_id: enterpriseRepo.id,
  classification_scope: 'Landscape.Baseline',
  status: 'issued'
});
expect(survey.target_count).toBe(10);

const notices = await db.find('governance_notifications', {
  related_entity_id: survey.survey_id,
  notification_type: 'survey_due'
});
expect(notices.length).toBe(10);
```

### Phase 2: Owner Responds (Alice)
1. Alice opens in-app notification, reviews outstanding metadata items.
2. Updates asset metadata fields (e.g., adds data steward and data sensitivity tags).
3. Submits `SurveyResponse` with status `completed` and attaches evidence link.
4. Automation bot re-runs metadata checks; marks asset `clean`.

**Expected Outcome**
- `SurveyResponse` stored (status `completed`).
- Associated asset metadata updated timestamps.
- `CheckRun` entry for metadata completeness re-evaluated to `passed`.

**Test Assertion**
```typescript
const response = await db.findOne('survey_responses', {
  survey_id: survey.survey_id,
  responder_id: alice.id
});
expect(response.status).toBe('completed');

const latestCheck = await db.findOne('check_runs', {
  asset_version_id: baselineVision_v1.id,
  check_id: metadataCheck.id
}, { orderBy: [['completed_at', 'DESC']] });
expect(latestCheck.status).toBe('passed');
```

### Phase 3: Reminder & Escalation (Automation Bot)
1. Reminder job runs after 48h for non-responders.
2. Notifications `survey_due` resent; escalation to David for items nearing due date.
3. `PromotionQueueEntry` created for surveys still pending, referencing `graph_statistic_snapshot_id`.

**Expected Outcome**
- Reminder notifications logged.
- `PromotionQueueEntry.sla_state=warning` for pending items.
- Escalation notifications use `notification_type=review_assignment` with context `survey`.

### Phase 4: Survey Closure & Telemetry
1. Survey deadline reached; Eve closes survey.
2. System marks outstanding responses `expired`, raises `SlaBreachEvent` if any remain incomplete.
3. Telemetry updates `GovernanceMetricSnapshot` (data quality score) and creates `RepositoryStatisticSnapshot` entry.
4. If failure rate > threshold, automation triggers `RecertificationJob` for impacted assets.

**Expected Outcome**
- Survey status `closed`.
- Data quality score improved compared to previous snapshot.
- Optional `RecertificationJob` rows created when responses missing.

**Test Assertion**
```typescript
await governanceApi.closeSurvey(survey.survey_id);
const closedSurvey = await db.findOne('data_quality_surveys', { survey_id: survey.survey_id });
expect(closedSurvey.status).toBe('closed');

const latestMetrics = await db.findOne('governance_metric_snapshots', {
  graph_id: enterpriseRepo.id
}, { orderBy: [['captured_at', 'DESC']] });
expect(latestMetrics.data_quality.score).toBeGreaterThan(previousMetrics.data_quality.score);
```

---

## Telemetry & KPIs
- Survey completion rate (target ≥ 90%).
- Average turnaround time per response.
- Post-survey data quality score uplift.
- Recertification follow-up count.

---

## FR/NFR Coverage
- **FR-20**: Data quality surveys and telemetry.
- **FR-13**: Workflow management for escalations (via PromotionQueueEntry).
- **NFR-Audit-01**: Survey issuance and response trail recorded.
- **NFR-Obs-01**: Telemetry snapshots updated in near real-time.

---

## Related Journeys
- **Journey 231**: Source assets whose metadata must remain high quality.
- **Journey 236**: Escalation mechanics reused for overdue surveys.
- **Journey 244**: Negative path when escalations are ignored.

---

## Revision History
- **2025-02-XX**: Initial draft covering data-quality survey lifecycle.
