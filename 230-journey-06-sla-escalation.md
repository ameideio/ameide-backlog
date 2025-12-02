# Journey 236 — SLA Breach & Escalation

## Overview
**Duration**: 4–6 minutes  
**Primary Personas**: Bob (Reviewer), David (Governance Lead)  
**Supporting Personas**: Support Analyst, Automation Bot, Product Owner (Carol)  
**Complexity**: Medium

**Business Value**: Validates monitoring and escalation paths when review SLAs breach, ensuring governance queues remain healthy and stakeholders are informed promptly.

---

## Prerequisites

### System Configuration
- Governance profile v1.1 with default review SLA of 48 hours.
- `SlaBreachEvent` rules configured for review cases and promotion queue entries.
- Escalation matrix defined within `GovernanceRoutingRule` (primary owner group → escalation group → leadership).
- Notification channels (email, threads) integrated with delivery status tracking.

### User Setup
- **Bob**: Reviewer responsible for queued case.
- **David**: Governance Lead overseeing queue health.
- **Support Analyst**: Handles operational escalations.
- **Carol**: Product Owner awaiting approval.

### Seed Data
- ReviewCase in `in_progress` state for an AssetVersion, assigned to EA Board, started 50 hours ago (SLA about to breach).
- Automation monitors queue depth and SLA timers.
- Telemetry dashboards streaming queue metrics.

---

## User Story
**As** David (Governance Lead),  
**I want** to detect when approvals breach SLA and coordinate mitigation actions,  
**So that** governance throughput remains predictable and stakeholders stay informed.

---

## Journey Steps

### Phase 1: SLA Breach Detection
1. Scheduler evaluates ReviewCase timers; identifies case exceeding 48-hour SLA.
2. System emits `SlaBreachEvent` with severity `warning`, linking ReviewCase, owner group, and impacted RepositoryElements.
3. Notifications sent to primary reviewers (Bob, Bob2) and Governance Lead.
4. Telemetry dashboards flag breach; queue heatmap highlights item.

**Expected Outcome**
- Breach recorded in `GovernanceMetricSnapshot` and `RepositoryStatisticSnapshot`.
- PromotionQueueEntry state updated to `warning` with `sla_state=warning` and `escalation_state=normal`.
- Incident reference created for audit with start timestamp.

**Test Assertion**:
```typescript
const breachEvent = await db.findOne('sla_breach_events', { related_entity_id: reviewCase.id });
expect(breachEvent.status).toBe('open');

const queueEntry = await db.findOne('promotion_queue_entries', { review_case_id: reviewCase.id });
expect(queueEntry.sla_state).toBe('warning');
expect(queueEntry.escalation_state).toBe('normal');
```

### Phase 2: Governance Lead Mitigation (David)
1. David opens queue dashboard filtered to breached items.
2. He reviews context: required checks status, impacted transformations, adoption risk.
3. David reassigns reviewers (adds Support Analyst delegate) and adjusts priority to `escalated`.
4. Governance workflows logs reassignment; notifications reissued with new SLA (12-hour remediation window).

**Expected Outcome**
- PromotionQueueEntry shows updated owner and priority.
- PromotionQueueEntry.escalation_state set to `escalated` and `sla_due_at` recalculated for remediation window.
- `SlaBreachEvent` status `acknowledged`.
- Stakeholder notifications confirm escalated handling.

**Test Assertion**:
```typescript
await page.click(`[data-review-id="${reviewCase.id}"] [data-testid="escalate-btn"]`);
await page.selectOption('[data-testid="new-owner-select"]', supportAnalyst.id);
await page.click('[data-testid="confirm-escalation-btn"]');
const escalatedEntry = await db.findOne('promotion_queue_entries', { review_case_id: reviewCase.id });
expect(escalatedEntry.escalation_state).toBe('escalated');
expect(new Date(escalatedEntry.sla_due_at).getTime()).toBeGreaterThan(Date.now());

const acknowledgedEvent = await db.findOne('sla_breach_events', { related_entity_id: reviewCase.id });
expect(acknowledgedEvent.status).toBe('acknowledged');
```

### Phase 3: Reviewer Action (Bob)
1. Bob acknowledges escalation, reviews outstanding evidence, and completes approval with rationale.
2. If additional information needed, he requests changes instead, resetting SLA in controlled manner.
3. Upon approval, ReviewCase transitions to `resolved`, queue entry closed.
4. Automation marks `SlaBreachEvent` `resolved`, capturing time-to-recover.

**Expected Outcome**
- Queue depth and SLA metrics return to normal.
- LifecycleEvent recorded for approval.
- Governance dashboards show breach resolved within remediation target, `PromotionQueueEntry.sla_state=on_track`, and `SlaBreachEvent.status=resolved`.

**Test Assertion**:
```typescript
await waitFor(async () => {
  const resolvedCase = await db.findOne('review_cases', { review_case_id: reviewCase.id });
  const queueEntry = await db.findOne('promotion_queue_entries', { review_case_id: reviewCase.id });
  const breach = await db.findOne('sla_breach_events', { related_entity_id: reviewCase.id });
  return resolvedCase.state === 'resolved' && queueEntry.sla_state === 'on_track' && breach.status === 'resolved';
});
```

### Phase 4: Post-Mortem & Alerts
1. David logs mitigation notes (root cause, action taken) within `GovernanceRecord`.
2. Support Analyst reviews trend; if recurring, raises improvement task (e.g., add reviewer capacity).
3. Telemetry join updates to correlate breach with transformation readiness delays.

---

## Telemetry & KPIs
- **SLA Breach Count** (target zero by end of reporting window).
- **Mean Time to Recover (MTTR)** from breach (target < 12h).
- **Queue Wait P95** before and after escalation.
- **Notification Acknowledgement** latency.
- **Reassignment Frequency** (monitor for reviewer load issues).

---

## FR/NFR Coverage
- **FR-13**: Review queue management.
- **FR-14**: SLA monitoring and escalations.
- **FR-15**: Queue telemetry surfaced to dashboards.
- **FR-20**: Auditability of escalations.
- **NFR-Res-01**: Process resilience under breach conditions.
- **NFR-Obs-01**: Observability of governance operations.

---

## Related Journeys
- **Journey 231**: Baseline approval path subject to SLA.
- **Journey 232**: Recertification cases that might trigger escalations.
- **Journey 234**: Compliance remediation if breach stems from failed checks.
- **Journey 244**: Negative scenario handling absent escalation response.

---

## Revision History
- **2025-02-XX**: Initial documentation of SLA escalation journey.
