# Journey 239 — Retention Hold & Purge Workflow

## Overview
**Duration**: 7–9 minutes  
**Primary Personas**: Carol (Compliance Officer), Alice (Asset Owner)  
**Supporting Personas**: Automation Bot, Legal Counsel  
**Complexity**: Medium

**Business Value**: Demonstrates how retention policies are enforced—applying a legal hold, auditing `AssetRetentionEvent`, expiring the hold, and executing purge—while ensuring governance notifications and telemetry remain accurate.

---

## Prerequisites
- Repository `Enterprise` configured with retention policy (`retention_policy.retention_days = 180`).
- Asset `securityAssessment` with approved `AssetVersion` older than 200 days.
- Legal case reference `LC-2025-04` requiring hold on the asset.
- Governance notifications and audit pipelines active.

---

## Journey Steps

### Phase 1: Apply Legal Hold (Carol)
1. Carol navigates to the asset retention panel and selects “Apply Legal Hold”.
2. Provides reason “LC-2025-04 discovery request” and assigns Legal Counsel as secondary reviewer.
3. System records `AssetRetentionEvent` (`event_type=hold_applied`, `trigger_source=legal_hold`).
4. Notifications (`notification_type=retention_event`) sent to asset owner and legal team.

**Expected Outcome**
- Asset retention state `retention_hold`.
- Retention hold metadata captured with legal reference.
- Notifications logged for the hold.

**Test Assertion**
```typescript
const holdEvent = await db.findOne('asset_retention_events', {
  asset_id: securityAssessment.id,
  event_type: 'hold_applied'
});
expect(holdEvent.trigger_source).toBe('legal_hold');
expect(holdEvent.event_payload.case_reference).toBe('LC-2025-04');

const retentionNotif = await db.findOne('governance_notifications', {
  related_entity_id: holdEvent.retention_event_id,
  notification_type: 'retention_event'
});
expect(retentionNotif.recipient_id).toBe(alice.id);
```

### Phase 2: Attempted Purge (Automation Guard)
1. Scheduled retention job evaluates assets past 180-day window.
2. System detects active hold and skips purge for `securityAssessment`.
3. `RepositoryStatisticSnapshot` logs skipped purge count; audit trail notes hold override.

**Expected Outcome**
- No purge executed while hold active.
- Snapshot metrics reflect skipped purge.
- Audit log entry `asset.purge_skipped` recorded.

### Phase 3: Release Hold (Legal Counsel)
1. Legal Counsel marks case as resolved and instructs Carol to release hold.
2. Carol records release reason and confirms purge window (7 days).
3. System creates `AssetRetentionEvent` (`event_type=hold_released`) and schedules purge (new event `purge_scheduled` with `occurred_at` = now + 7 days).

**Expected Outcome**
- Asset retention state `active`.
- Purge scheduled metadata recorded (`event_payload.purge_after`).
- Notifications generated for asset owner.

**Test Assertion**
```typescript
const releaseEvent = await db.findOne('asset_retention_events', {
  asset_id: securityAssessment.id,
  event_type: 'hold_released'
});
expect(releaseEvent.triggered_by).toBe(carol.id);

const purgeEvent = await db.findOne('asset_retention_events', {
  asset_id: securityAssessment.id,
  event_type: 'purge_scheduled'
});
expect(new Date(purgeEvent.event_payload.purge_after).getTime()).toBeGreaterThan(Date.now());
```

### Phase 4: Purge Execution
1. After 7 days, automated job purges the archived AssetVersion.
2. System removes search index document, knowledge graph node, and stores `AssetRetentionEvent` (`event_type=purge_executed`).
3. Telemetry updates `RepositoryStatisticSnapshot` (purge count) and `GovernanceMetricSnapshot` (retention compliance KPI).

**Expected Outcome**
- AssetVersion archived (`archived_at` populated).
- Purge event logged with evidence hash for audit.
- Notifications sent summarising purge completion.

**Test Assertion**
```typescript
const purgeExecuted = await db.findOne('asset_retention_events', {
  asset_id: securityAssessment.id,
  event_type: 'purge_executed'
});
expect(purgeExecuted.event_payload.evidence_hash).toBeDefined();

const archivedVersion = await db.findOne('asset_versions', { asset_version_id: securityAssessmentVersion.id });
expect(archivedVersion.archived_at).toBeDefined();
```

---

## Telemetry & KPIs
- Retention compliance (purge vs hold counts).
- Legal hold duration (actual vs target).
- Number of purge skips due to holds.
- Time from hold release to purge execution.

---

## FR/NFR Coverage
- **FR-20**: Retention policy governance and audit logs.
- **FR-13**: Notification workflows for retention events.
- **NFR-Compliance-01**: Legal hold auditable trail.
- **NFR-Perf-02**: Scheduled purge execution within SLA.

---

## Related Journeys
- **Journey 232**: Policy changes prompting re-certification after purge.
- **Journey 234**: Compliance remediation feeding retention decisions.
- **Journey 245**: Negative path for failed bulk import (potential purge rollback).

---

## Revision History
- **2025-02-XX**: Initial draft for retention hold and purge workflows.
