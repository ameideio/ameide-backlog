# Journey 240 — Governance Waiver Lifecycle

## Overview
**Duration**: 6–8 minutes  
**Primary Personas**: David (Governance Lead), Bob (Reviewer)  
**Supporting Personas**: Alice (Asset Owner), Compliance Officer  
**Complexity**: Medium

**Business Value**: Validates issuing, approving, tracking, and expiring a governance waiver so policy exceptions remain controlled, time-bound, and auditable.

---

## Prerequisites
- Governance profile requires Privacy Impact Assessment before approval.
- AssetVersion `customerPortal_v2` currently blocked by failing check (`CheckRun.status = failed`).
- Waiver template configured (`GovernanceRecord` subtype `waiver`) with default 14-day expiry.
- Governance routing rules assign waivers to Compliance Officer for ratification.

---

## Journey Steps

### Phase 1: Waiver Request (Alice)
1. Alice opens the ReviewCase, clicks “Request Waiver”.
2. Provides justification (“Need to deploy pilot before assessment completes”), proposed expiry (7 days), and mitigation plan.
3. System creates draft `GovernanceRecord` (`record_type=waiver`) linked to ReviewCase and asset.
4. Waiver request routed to Governance Lead and Compliance Officer via `GovernanceNotification` (`notification_type=decision_request`).

**Expected Outcome**
- Waiver record status `draft` with pending approvers.
- ReviewCase state remains `changes_requested` (no bypass yet).
- Notifications logged for approvers.

**Test Assertion**
```typescript
const waiverRecord = await db.findOne('governance_records', {
  review_case_id: reviewCase.id,
  record_type: 'waiver'
});
expect(waiverRecord.status).toBe('draft');

const waiverNotif = await db.find('governance_notifications', {
  related_entity_id: waiverRecord.governance_record_id,
  notification_type: 'decision_request'
});
expect(waiverNotif.map(n => n.recipient_id)).toContain(complianceOfficer.id);
```

### Phase 2: Waiver Approval (David & Compliance Officer)
1. David reviews mitigation, adds condition (“Assessment must complete within 7 days”).
2. Compliance Officer ratifies waiver, both submit approvals.
3. System updates `GovernanceRecord.status=approved`, sets `effective_from` and `effective_to` dates.
4. ReviewCase state transitions to `in_progress`; failing check is marked `waived`.

**Expected Outcome**
- Waiver approved with expiry date recorded.
- `CheckRun.status=waived` and `CheckRun.expiration_at` matches waiver expiry.
- Notifications (`notification_type=review_outcome`) inform asset owner.

**Test Assertion**
```typescript
const approvedWaiver = await db.findOne('governance_records', {
  governance_record_id: waiverRecord.governance_record_id
});
expect(approvedWaiver.status).toBe('approved');
const expiryDelta = new Date(approvedWaiver.effective_to).getTime() - new Date(approvedWaiver.effective_from).getTime();
expect(expiryDelta).toBe(7 * 24 * 60 * 60 * 1000);

const waivedCheck = await db.findOne('check_runs', {
  review_case_id: reviewCase.id,
  check_id: privacyImpactCheck.id
});
expect(waivedCheck.status).toBe('waived');
expect(waivedCheck.expiration_at).toEqual(approvedWaiver.effective_to);
```

### Phase 3: Waiver Monitoring
1. Daily job monitors waivers nearing expiry and issues reminders (`notification_type=review_feedback`).
2. Governance dashboard displays waiver badge with countdown.
3. `SlaBreachEvent` prepared to trigger if waiver expires without remediation.

**Expected Outcome**
- Reminder notifications logged.
- PromotionQueueEntry for the ReviewCase shows `escalation_state=warning` as expiry nears.
- Telemetry updates waiver metrics (active vs expiring).

### Phase 4: Waiver Expiry & Cleanup
1. Alice completes the outstanding Privacy Impact Assessment before expiry.
2. Check rerun passes; Governance Lead marks waiver `closed`.
3. System logs `GovernanceRecord.status=retired`, removes `waived` flag, and updates ReviewCase to `resolved`.

**Expected Outcome**
- Waiver retired, history preserved.
- Check status becomes `passed`.
- `SlaBreachEvent` never escalates (status stays `cancelled`).

**Test Assertion**
```typescript
await automationBot.rerunCheck(privacyImpactCheck.id, customerPortal_v2.id);
const latestCheck = await db.findOne('check_runs', {
  check_id: privacyImpactCheck.id,
  asset_version_id: customerPortal_v2.id
}, { orderBy: [['completed_at', 'DESC']] });
expect(latestCheck.status).toBe('passed');

const retiredWaiver = await db.findOne('governance_records', { governance_record_id: waiverRecord.governance_record_id });
expect(retiredWaiver.status).toBe('retired');

const breachEvent = await db.findOne('sla_breach_events', { related_entity_id: reviewCase.id });
expect(breachEvent.status).toBe('cancelled');
```

---

## Telemetry & KPIs
- Active vs expired waiver count.
- Average waiver duration.
- Percentage of waivers closed before expiry.
- Check compliance rate post-waiver.

---

## FR/NFR Coverage
- **FR-13**: Governance workflows handling waivers.
- **FR-14**: SLA monitoring via queue escalation.
- **FR-20**: Waiver audit trail.
- **NFR-Compliance-01**: Policy exception traceability.

---

## Related Journeys
- **Journey 231**: Base review cycle; waiver fits into change-request loop.
- **Journey 234**: Compliance remediation may trigger waivers.
- **Journey 244**: Negative path when escalations are ignored post-waiver.

---

## Revision History
- **2025-02-XX**: Initial draft documenting governance waiver lifecycle.
