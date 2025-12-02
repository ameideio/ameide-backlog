# Journey 234 — Compliance Violation & Remediation

## Overview
**Duration**: 6–8 minutes  
**Primary Personas**: Eve (Compliance Lead), Alice (Asset Owner), Bob (Reviewer)  
**Supporting Personas**: Automation Bot, David (Governance Lead)  
**Complexity**: Medium

**Business Value**: Exercises how compliance checks surface violations, orchestrate remediation, and feed telemetry so governance risk can be resolved with full traceability.

---

## Prerequisites

### System Configuration
- Governance profile v1.1 active with Security & Privacy Policy requiring quarterly attestation.
- Automated RequiredCheck “Data Residency Scan” configured as blocking for `Landscape.Baseline`.
- GovernanceRoutingRule directs compliance breaches to “Standards Team” owner group.
- Incident response playbook stored as `GovernanceWorkflowPlaybook`.

### User Setup
- **Eve**: Compliance Lead, member of Standards Team owner group.
- **Alice**: Asset owner for baseline asset.
- **Bob**: Reviewer in EA Board.
- **David**: Governance Lead supervising escalations.

### Test Data
- Approved baseline asset in graph.
- Latest RequiredCheck run older than 90 days, ready to trigger compliance scan.
- Telemetry dashboards active with anomaly detection.

---

## User Story
**As** Eve (Compliance Lead),  
**I want** to triage a compliance violation, coordinate remediation with the asset owner, and close out the incident,  
**So that** risk is mitigated and dashboards reflect up-to-date compliance posture.

---

## Journey Steps

### Phase 1: Violation Detection
1. Scheduled `CheckRun` for Data Residency Scan executes and fails (status `failed`, severity `critical`).
2. Automation bot logs failure with evidence details, impacted RepositoryElements, and attaches to `ReviewCase`.
3. `GovernanceRoutingRule` triggers: Standards Team assigned, `SlaBreachEvent` prepared with 24-hour remediation clock.
4. Eve receives notification; Governance dashboard flags compliance risk.

**Expected Outcome**
- ReviewCase state `on_hold` with blocking check failure.
- Violation recorded in `GovernanceMetricSnapshot` and `RepositoryStatisticSnapshot`.
- Initiatives consuming the asset show risk chip in blockers funnel.
- Fallback owner group engaged when primary quorum unavailable.

**Test Assertion**:
```typescript
await db.update('governance_owner_groups', { owner_group_id: standardsTeam.id }, { quorum_available: false });
await waitFor(async () => {
  const routedCase = await db.findOne('review_cases', { review_case_id: reviewCase.id });
  return routedCase.assigned_owner_group_id === complianceEscalationGroup.id;
});
```

### Phase 2: Remediation Planning (Eve & Alice)
1. Eve opens compliance workspace summarising failing evidence, impacted controls, and affected RepositoryElements.
2. She adds remediation tasks (update privacy model, refresh evidence) using `GovernanceWorkflowPlaybook` guidance.
3. Alice acknowledges assignment, opens asset draft, and reviews comments.
4. Alice updates documentation, regenerates privacy data mapping, and uploads new `EvidenceArtifact` with signature.

**Expected Outcome**
- Remediation tasks tracked with due dates; notifications sent to stakeholders.
- Asset enters temporary `draft` state for edits; LifecycleEvents recorded.
- New evidence linked via `TraceLink`; automation ready for rerun.

### Phase 3: Re-run Checks & Approval (Bob)
1. Alice re-requests review. Automation bot reruns Data Residency check; this time status `passed`.
2. ReviewCase transitions back to `open`; `SlaBreachEvent` timer stops before breach.
3. Bob verifies remediation, reviews evidence, and approves with rationale referencing violation ticket.
4. Eve marks incident resolved; `GovernanceRecord` generated summarising remediation path.

**Expected Outcome**
- ReviewCase closed with state `resolved`.
- Control coverage metrics updated to reflect restored compliance.
- `GovernanceMetricSnapshot` registers violation as resolved within SLA.

### Phase 4: Telemetry & Follow-up
1. Telemetry join updates to show compliance risk cleared; dashboards drop risk badge.
2. David schedules follow-up survey to monitor recurring issues; `DataQualitySurvey` targeting same classification.
3. Automation bot archives incident details for post-mortem analysis.

---

## Telemetry & KPIs
- **Compliance Violation Count** (target 0 after remediation).
- **Time to Remediate** (target < 24h).
- **Check Pass Rate** (should return to 100%).
- **SLA Breach Events** (none expected).
- **Incident Post-Mortem Completion** (tracked through GovernanceRecord).

---

## FR/NFR Coverage
- **FR-13**: Review workflows with required checks.
- **FR-14**: SLA monitoring and escalations.
- **FR-16**: Policy-driven remediation workflows.
- **FR-20**: Telemetry reporting of compliance.
- **NFR-Audit-01**: Incident fully traceable with evidence.
- **NFR-Res-01**: Remediation playbook executed without manual overrides.

---

## Related Journeys
- **Journey 232**: Policy update that can trigger recertification if violations persist.
- **Journey 236**: SLA escalation path if remediation exceeds thresholds.
- **Journey 244**: Negative scenario when escalation fails.
- **Journey 246**: Adoption and communication updates triggered by compliance changes.

---

## Revision History
- **2025-02-XX**: Initial documentation of compliance violation remediation journey.
