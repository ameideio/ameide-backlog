# Journey 241 — Release Package & Go/No-Go Decision

## Overview
**Duration**: 10–12 minutes  
**Primary Personas**: Carol (Product Owner), Frank (Sponsor)  
**Supporting Personas**: Bob (Reviewer), Platform Analyst  
**Complexity**: High

**Business Value**: Bundles approved deliverables into a `ReleasePackage`, performs readiness checks, presents evidence to the steering committee, and records an `OutcomeDecision`, proving Ameide’s ability to orchestrate release governance end-to-end.

---

## Prerequisites
- Initiative `Digital Transformation 2025` with milestones approaching release.
- Approved AssetVersions flagged `release_candidate=true`.
- Required release checklist defined (governance checks, change tasks, adoption metrics).
- Dashboard templates and export service configured.

---

## Journey Steps

### Phase 1: Assemble Release Package (Carol)
1. Carol opens the Release workspace and selects approved scope (3 deliverables).
2. System creates `ReleasePackage` with version `R2025.1`, storing bundle URI and initial status `candidate`.
3. Automated job hydrates package with linked evidence (checks, waivers, training assets).

**Expected Outcome**
- `ReleasePackage` row created (`readiness_status=candidate`).
- Linked AssetVersions stored in `release_package_assets` join table.
- `RequiredCheck` bindings cloned to package scope.

**Test Assertion**
```typescript
const release = await db.findOne('release_packages', { transformation_id: transformation.id, release_version: 'R2025.1' });
expect(release.readiness_status).toBe('candidate');

const packageAssets = await db.find('release_package_assets', { release_package_id: release.release_package_id });
expect(packageAssets.length).toBe(3);
```

### Phase 2: Readiness Verification (Automation & Bob)
1. Automation bot executes release checks (regression tests, security scan, change readiness) via `CheckRun` targeting the package.
2. Bob reviews outstanding findings; marks one advisory as accepted.
3. Release dashboard updates readiness score and surfaces outstanding blockers (none remaining).

**Expected Outcome**
- All blocking `CheckRun` entries status `passed`.
- Any advisory recorded with rationale.
- `ReleasePackage.readiness_status` updated to `approved`.

**Test Assertion**
```typescript
const releaseChecks = await db.find('check_runs', {
  release_package_id: release.release_package_id
});
expect(releaseChecks.every(check => check.enforcement_level === 'advisory' || check.status === 'passed')).toBe(true);

const refreshedRelease = await db.findOne('release_packages', { release_package_id: release.release_package_id });
expect(refreshedRelease.readiness_status).toBe('approved');
```

### Phase 3: Steering Committee Decision (Frank)
1. Carol exports Go/No-Go briefing (pulls metrics from telemetry join + release evidence).
2. Steering committee meets; Frank records decision `OutcomeDecision.decision_type=continue` with follow-up actions.
3. Outcome stored, linked to release package and transformation.

**Expected Outcome**
- `OutcomeDecision` row stored with `decision_body=steering_committee` and `follow_up_actions` JSON.
- `GovernanceRecord` (meeting minutes) captures evidence.
- Telemetry dashboards log decision timestamp.

**Test Assertion**
```typescript
const decision = await db.findOne('outcome_decisions', { release_package_id: release.release_package_id });
expect(decision.decision_type).toBe('continue');
expect(decision.follow_up_actions).toContain('Track adoption for Finance segment');

const minutes = await db.findOne('governance_records', {
  related_entity_type: 'release_package',
  related_entity_id: release.release_package_id
});
expect(minutes.record_type).toBe('meeting_minutes');
```

### Phase 4: Post-Decision Execution
1. Carol triggers deployment tasks; automation kicks off `RepositorySyncJob` for release artifacts.
2. Change manager schedules communications; adoption tasks linked to release package.
3. Telemetry snapshots capture `OutcomeMetricSnapshot` variance after go-live.

**Expected Outcome**
- Sync jobs succeed; release artifacts published.
- Adoption/change tasks created referencing release package.
- Outcome metrics show baseline vs post-go decision delta.

---

## Telemetry & KPIs
- Release readiness score (target ≥ 90%).
- Go/No-Go cycle time.
- Post-decision adoption variance.
- Number of outstanding actions post approval.

---

## FR/NFR Coverage
- **FR-15**: Release readiness reporting.
- **FR-18**: Scenario/tag propagation through release bundle.
- **FR-20**: Governance decision audit trail.
- **NFR-Res-01**: Post-go tracking of follow-up actions.

---

## Related Journeys
- **Journey 233**: Initiative workflows feeding release scope.
- **Journey 235**: Landscape toggle consumes baseline vs target after release.
- **Journey 246**: Adoption telemetry follows go-live outcomes.

---

## Revision History
- **2025-02-XX**: Initial draft covering release packaging and steering decision.
