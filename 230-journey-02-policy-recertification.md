# Journey 232 — Policy Update Triggers Recertification

## Overview
**Duration**: 8–12 minutes  
**Primary Personas**: David (Governance Lead), Alice (Asset Owner), Bob (Reviewer)  
**Supporting Personas**: Automation Bot, Carol (Program Manager)  
**Complexity**: Medium

**Business Value**: Demonstrates how a governance policy change cascades to existing assets, automatically reopens approvals, and ensures evidence is refreshed without losing traceability or SLA discipline.

---

## Prerequisites

### System Configuration
- Repository “Enterprise Repository” active with `GovernanceProfile` v1.0 (bundles security/privacy policies, playbooks, required check catalogue).
- GovernanceOwnerGroup “Enterprise Architecture Board” with quorum 2.
- GovernancePolicy “Security & Privacy Policy” (v1.0) linked to classification `Landscape.Baseline`.
- RequiredChecks:
  - Metadata completeness (blocking).
  - Security classification (blocking).
  - Data residency attestation (warning).
- Recertification job template configured for “Security & Privacy Policy”.

### User Setup
- **David**: Governance Lead, admin of governance profile.
- **Alice**: Asset owner of “2025 Digital Transformation Vision” (approved baseline).
- **Bob**: Reviewer in the EA Board.
- Automation bot account with permission to run RequiredChecks.

### Test Data
- Approved baseline asset version (`asset_version_id = av-001`) classified as `Landscape.Baseline`.
- Audit evidence stored as `EvidenceArtifact` linked through `TraceLink`.
- RepositoryStatisticSnapshot and GovernanceMetricSnapshot jobs scheduled hourly.

---

## User Story
**As** David (Governance Lead),  
**I want** to tighten the Security & Privacy Policy and force impacted assets back through review,  
**So that** previously approved baselines remain compliant with the new requirements.

---

## Journey Steps

### Phase 1: Publish Updated Policy (David)
1. David clones governance profile v1.0 to draft v1.1, adding a new blocking RequiredCheck “Privacy Impact Assessment”.
2. He updates `GovernancePolicy` effective date to immediate and links the new check via `CheckBinding` to the `Landscape.Baseline` classification.
3. System version increments `GovernancePolicy`, persists change log, and surfaces preview of impacted assets and transformations.
4. David activates profile v1.1. Activation emits `GovernanceRecord` entry and `GovernanceMetricSnapshot` with policy delta annotation.

**Expected Outcome**
- Governance profile v1.1 active.
- `RecertificationJob` generated for every approved AssetVersion under affected classification.
- Notifications dispatched to asset owners and governance queue.

**Test Assertion**:
```typescript
const profile = await db.findOne('governance_profiles', {
  graph_id: enterpriseRepo.id,
  version: '1.1.0'
});
expect(profile.status).toBe('active');

const mappings = await db.find('governance_profile_policies', {
  governance_profile_id: profile.governance_profile_id
});
expect(mappings.map(m => m.policy_id)).toContain(updatedPolicy.policy_id);
```

### Phase 2: Automation Bot Initiates Recertification
1. Recertification engine transitions impacted AssetVersions from `approved` to `in_review`.
2. Automation bot creates new `ReviewCase` per asset with RequiredChecks recalculated (including Privacy Impact Assessment).
3. `PromotionQueueEntry` appended with priority=`policy_change`.
4. SLA timers reset based on policy configuration; `RepositoryStatisticSnapshot` marks impacted assets as “Recertifying”.

**Expected Outcome**
- Review queues updated with new cases.
- `GovernanceMetricSnapshot` reflects recertification backlog count.
- Initiatives referencing the asset receive `InitiativeAlert` about pending recertification.

**Test Assertion**:
```typescript
const cases = await db.find('review_cases', {
  asset_version_id: baselineVersion.id
}, { orderBy: [['opened_at', 'ASC']] });
const firstCase = cases[0];
const newCase = cases[cases.length - 1];
expect(newCase.review_case_id).not.toBe(firstCase.review_case_id);
expect(new Date(newCase.opened_at).getTime()).toBeGreaterThan(new Date(firstCase.closed_at).getTime());
expect(new Date(newCase.sla_due_at).getTime() - new Date(newCase.opened_at).getTime()).toBe(newCase.sla_target * 60 * 1000);
```

### Phase 3: Asset Owner Responds (Alice)
1. Alice receives notification and opens draft. Asset state displays `in_review (recertification)`.
2. She completes new Privacy Impact Assessment questionnaire and uploads supporting evidence (`EvidenceArtifact` with signature).
3. Metadata updated; `TraceLink` ensures evidence references the correct RepositoryElements.
4. Alice resubmits for review; system reruns blocking checks via automation bot.

**Expected Outcome**
- RequiredChecks pass; `CheckRun` records show rerun status `succeeded`.
- ReviewCase status transitions to `open` and ready for approvals.
- LifecycleEvent appended noting recertification submission.
- Notification with `notification_type=review_resubmission` delivered to reviewers; SLA uses the new case’s `sla_due_at`.

**Test Assertion**:
```typescript
const recertCase = await db.findOne('review_cases', { asset_version_id: baselineVersion.id, state: 'open' });
const recertNotif = await db.findOne('governance_notifications', {
  related_entity_id: recertCase.review_case_id,
  notification_type: 'review_resubmission'
});
expect(recertNotif).toBeDefined();
```

### Phase 4: Governance Board Approves (Bob & Bob2)
1. Bob reviews case, inspections new evidence, confirms check history. Records decision `approve` with rationale referencing updated policy.
2. Bob2 completes quorum approval. SLA satisfied (<24h).
3. ReviewCase resolves to `approved`; AssetVersion returns to `approved`.
4. `ClassificationAssignment` validated against updated PlacementRules; `RepositorySyncJob` publishes deltas (knowledge graph & search).

**Expected Outcome**
- `RecertificationJob` marked `completed`.
- `GovernanceMetricSnapshot` shows reduced recert backlog and updated check coverage.
- `RepositoryStatisticSnapshot` logs recertification cycle time for dashboards.
- Initiative views clear blocker chips; Outcome telemetry flagged as stable.

### Phase 5: Post-Recertification Analytics
1. Automation bot writes `GovernanceRecord` summarising policy change impact and approvals.
2. Telemetry join updates to correlate policy version with asset compliance state.
3. Carol verifies transformation readiness scoreboard and documents absence of risk.

---

## Telemetry & KPIs
- **Recertification Cycle Time** (target P95 < 48h).
- **Check Pass Rate** for new check (target 100%).
- **Recertification Backlog Size** (expected drops to zero).
- **Notification Acknowledgement Latency** (target < 2h).
- **Sync Latency** after approval (target < 30s).

---

## FR/NFR Coverage
- **FR-13**: Review workflows with approvals.
- **FR-14**: SLA monitoring and queue management.
- **FR-16**: Policy-driven recertification.
- **FR-20**: Governance telemetry and audit trail.
- **NFR-Audit-01**: Recertification events logged immutably.
- **NFR-Perf-02**: Sync latency adherence.

---

## Related Journeys
- **Journey 231** (Draft to Baseline): Source asset that undergoes recertification.
- **Journey 234** (Compliance Violation): Escalation path if recertification fails.
- **Journey 236** (SLA Escalation): Handles overdue recertification cases.
- **Journey 244** (Negative path): Fallback when escalations stall.

---

## Revision History
- **2025-02-XX**: Initial documentation of policy-driven recertification flow.
