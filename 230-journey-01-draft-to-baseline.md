# Journey 231 — New Architecture Vision: Draft to Baseline

## Overview
**Duration**: 5-10 minutes
**Primary Personas**: Alice (Architect), Bob (Reviewer)
**Secondary Personas**: Frank (Stakeholder)
**Complexity**: Medium

**Business Value**: Validates the complete asset lifecycle from initial creation through governance approval to graph classification, ensuring architecture artifacts follow proper review workflows and become authoritative baseline references for the enterprise.

---

## Prerequisites

### System Configuration
- Repository: "Enterprise Repository" (TOGAF classifications configured)
- GovernanceOwnerGroup: "Enterprise Architecture Board" (quorum: 2 approvals)
- GovernancePolicy: "Architecture Governance Policy v1.0"
- PlacementRules: Only `approved` assets can be classified as `Landscape.Baseline`
- RequiredChecks:
  - Metadata completeness (blocking)
  - TOGAF compliance validation (blocking)
  - Security classification check (advisory)

### User Setup
- **Alice**: Contributor role in Enterprise Repository
- **Bob**: Reviewer in "Enterprise Architecture Board" GovernanceOwnerGroup
- **Bob2**: Second reviewer in same group (for quorum)

### Test Data
- None required (journey creates from scratch)

---

## User Story

**As** Alice (Enterprise Architect),
**I want to** create an Architecture Vision document, submit it for governance review, address feedback, and have it approved as an enterprise baseline,
**So that** transformation transformations can reference authoritative current-state architecture.

---

## Journey Steps

### Phase 1: Asset Creation & Enrichment (Alice)

#### Step 1.1: Login & Navigation
**Actor**: Alice
**Action**:
- Navigate to `https://platform.dev.ameide.io/repositories/enterprise`
- Click "Create Asset" button

**Expected Outcome**:
- Asset creation modal appears
- Asset subtype dropdown shows all TOGAF-aligned types

**Test Assertion**:
```typescript
await page.goto('/repositories/enterprise');
await page.click('[data-testid="create-asset-btn"]');
await expect(page.locator('[data-testid="asset-modal"]')).toBeVisible();
```

#### Step 1.2: Asset Basic Information
**Actor**: Alice
**Action**: Fill asset form
- Name: "2025 Digital Transformation Vision"
- Asset Subtype: "Phase Deliverable"
- ADM Phase: "A (Architecture Vision)"
- Description: "Strategic vision for enterprise-wide digital transformation, establishing principles, capabilities, and roadmap for 2025-2027 planning horizon"
- Owning Unit: "Enterprise Architecture"

**Expected Outcome**:
- Form validates successfully
- "Save Draft" button becomes enabled

**Test Assertion**:
```typescript
await page.fill('[data-testid="asset-name"]', '2025 Digital Transformation Vision');
await page.selectOption('[data-testid="asset-subtype"]', 'phase_deliverable');
await page.selectOption('[data-testid="adm-phase"]', 'A');
await expect(page.locator('[data-testid="save-draft-btn"]')).toBeEnabled();
```

#### Step 1.3: Artifact Upload
**Actor**: Alice
**Action**: Upload architecture documents
- Architecture Vision document (PDF, 2.5 MB)
- Executive summary slides (PPTX, 1.2 MB)
- Business capability map (ArchiMate model, 450 KB)

**Expected Outcome**:
- Files upload successfully
- File checksums calculated
- Upload artifacts displayed in file list

**Test Assertion**:
```typescript
await page.setInputFiles('[data-testid="file-upload"]', [
  'fixtures/architecture-vision.pdf',
  'fixtures/executive-summary.pptx',
  'fixtures/capability-map.archimate'
]);
await expect(page.locator('[data-testid="upload-status"]')).toContainText('3 files uploaded');
```

#### Step 1.4: Save as Draft
**Actor**: Alice
**Action**: Click "Save Draft"

**Expected Outcome**:
- Asset created with unique `asset_id`
- AssetVersion created (version "1.0")
- Lifecycle state: `draft`
- Success notification displayed
- Redirected to asset detail page

**Test Assertion**:
```typescript
await page.click('[data-testid="save-draft-btn"]');
await expect(page.locator('[data-testid="lifecycle-state"]')).toContainText('Draft');
const assetId = await page.getAttribute('[data-testid="asset-id"]', 'data-id');
expect(assetId).toMatch(/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/);

// Verify in database
const asset = await db.findOne('assets', { asset_id: assetId });
expect(asset.name).toBe('2025 Digital Transformation Vision');
expect(asset.asset_subtype).toBe('phase_deliverable');

const version = await db.findOne('asset_versions', { asset_id: assetId });
expect(version.lifecycle_state).toBe('draft');
expect(version.version_number).toBe('1.0');
```

#### Step 1.5: Metadata Enrichment
**Actor**: Alice
**Action**: Add metadata
- Tags: `digital-transformation`, `cloud-first`, `customer-experience`, `2025-roadmap`
- Stakeholders: CEO, CIO, VP Customer Experience
- Effective window: 2025-01-01 to 2027-12-31
- Continuum attributes: `{"abstraction_level": "strategic", "detail_level": "vision"}`

**Expected Outcome**:
- Metadata saved to asset record
- Search index updated (background job)

**Test Assertion**:
```typescript
await page.click('[data-testid="edit-metadata-btn"]');
await page.fill('[data-testid="tags-input"]', 'digital-transformation, cloud-first');
await page.click('[data-testid="save-metadata-btn"]');

const updatedAsset = await db.findOne('assets', { asset_id: assetId });
expect(updatedAsset.tags).toEqual(expect.arrayContaining(['digital-transformation', 'cloud-first']));
```

---

### Phase 2: Pre-Submission Validation (Alice)

#### Step 2.1: Run Automated Checks
**Actor**: Alice
**Action**: Click "Run Checks" button

**Expected Outcome**:
- System executes RequiredChecks:
  - Metadata completeness: ✓ Passed
  - TOGAF compliance: ✓ Passed
  - Security classification: ⚠ Advisory warning (no security classification tag)

**Test Assertion**:
```typescript
await page.click('[data-testid="run-checks-btn"]');
await page.waitForSelector('[data-testid="checks-complete"]');

const checks = await db.find('check_runs', { asset_version_id: version.id });
expect(checks).toHaveLength(3);
expect(checks.filter(c => c.status === 'passed')).toHaveLength(2);
expect(checks.filter(c => c.status === 'warning')).toHaveLength(1);
```

#### Step 2.2: Address Advisory Warning
**Actor**: Alice
**Action**: Add security classification tag: `internal`

**Expected Outcome**:
- Metadata updated
- Advisory warning cleared on re-check

---

### Phase 3: Review Submission (Alice)

#### Step 3.1: Submit for Review
**Actor**: Alice
**Action**:
- Click "Submit for Review"
- Select target classification: "Landscape.Baseline"
- Enter justification: "Establishes strategic baseline for all transformation programs. Required for Q1 2025 planning cycle."
- Set effective window start: `2025-01-15` (future-dated baseline activation)
- Confirm submission

**Expected Outcome**:
- ReviewCase created
- Asset lifecycle state: `draft` → `in_review`
- GovernanceOwnerGroup "EA Board" assigned
- Notifications sent to Bob and Bob2 with `notification_type=review_assignment`
- SLA timer starts (2 business days) and `sla_due_at` recorded
- LifecycleEvent logged with metadata noting classification placement
- AssetVersion.effective_from populated for the submitted version

**Test Assertion**:
```typescript
await page.click('[data-testid="submit-for-review-btn"]');
await page.selectOption('[data-testid="classification"]', 'landscape-baseline');
await page.fill('[data-testid="justification"]', 'Establishes strategic baseline...');
await page.fill('[data-testid="effective-from"]', '2025-01-15');
await page.click('[data-testid="confirm-submit-btn"]');

await expect(page.locator('[data-testid="review-status"]')).toContainText('In Review');

// Database verification
const reviewCase = await db.findOne('review_cases', { asset_version_id: version.id });
expect(reviewCase.state).toBe('open');
expect(reviewCase.assigned_owner_group_id).toBe(eaBoardGroup.id);
expect(reviewCase.sla_target).toBeGreaterThan(0);
const openedAt = new Date(reviewCase.opened_at).getTime();
const dueAt = new Date(reviewCase.sla_due_at).getTime();
expect(dueAt - openedAt).toBe(reviewCase.sla_target * 60 * 1000);

const updatedVersion = await db.findOne('asset_versions', { asset_version_id: version.id });
expect(updatedVersion.lifecycle_state).toBe('in_review');
expect(updatedVersion.submitted_for_review_at).toBeDefined();
expect(updatedVersion.effective_from.startsWith('2025-01-15')).toBe(true);

// Notification verification
const notifications = await db.find('governance_notifications', {
  related_entity_id: reviewCase.id,
  notification_type: 'review_assignment'
});
expect(notifications.map(n => n.recipient_id)).toContain(bob.id);

// Audit trail
const lifecycleEvent = await db.findOne('lifecycle_events', {
  asset_version_id: version.id,
  to_state: 'in_review'
});
expect(lifecycleEvent.actor_id).toBe(alice.id);
expect(lifecycleEvent.metadata.classification_code).toBe('Landscape.Baseline');
expect(lifecycleEvent.rationale).toContain('Establishes strategic baseline');
```

---

### Phase 4: Initial Review & Feedback (Bob)

#### Step 4.1: Receive & Open Review
**Actor**: Bob
**Action**:
- Receive email notification: "New review assigned: 2025 Digital Transformation Vision"
- Click email link → Opens AMEIDE
- Navigate to review queue `/governance/reviews`
- See review case in queue with SLA countdown

**Expected Outcome**:
- Review case visible in queue
- SLA indicator shows time remaining
- Asset metadata summary displayed

**Test Assertion**:
```typescript
// Simulate Bob login
await loginAs(bob);
await page.goto('/governance/reviews');

const reviewCard = page.locator(`[data-review-id="${reviewCase.id}"]`);
await expect(reviewCard).toBeVisible();
await expect(reviewCard.locator('[data-testid="sla-remaining"]')).toContainText('48 hours');
```

#### Step 4.2: Review Asset Details
**Actor**: Bob
**Action**:
- Click review case
- Download Architecture Vision PDF
- View ArchiMate model in embedded viewer
- Check metadata completeness
- Verify check results

**Expected Outcome**:
- All artifacts accessible
- Check status visible
- Review decision form displayed

#### Step 4.3: Request Changes
**Actor**: Bob
**Action**:
- Click "Request Changes"
- Add comment: "Missing financial impact section in Architecture Vision. Please add cost-benefit analysis and budget estimates for 2025-2027 horizon."
- Submit decision

**Expected Outcome**:
- ApprovalDecision created (decision: `request_changes`)
- ReviewCase state: `open` → `changes_requested`
- AssetVersion state: `in_review` → `rework`
- Notification sent to Alice
- LifecycleEvent logged

**Test Assertion**:
```typescript
await page.click(`[data-review-id="${reviewCase.id}"]`);
await page.click('[data-testid="request-changes-btn"]');
await page.fill('[data-testid="rationale"]', 'Missing financial impact section...');
await page.click('[data-testid="submit-decision-btn"]');

// Database verification
const approvalDecision = await db.findOne('approval_decisions', {
  review_case_id: reviewCase.id,
  approver_id: bob.id
});
expect(approvalDecision.decision).toBe('request_changes');

const updatedReview = await db.findOne('review_cases', { review_case_id: reviewCase.id });
expect(updatedReview.state).toBe('changes_requested');

const reworkVersion = await db.findOne('asset_versions', { asset_version_id: version.id });
expect(reworkVersion.lifecycle_state).toBe('rework');

// Notification to Alice
const aliceNotif = await db.findOne('governance_notifications', {
  recipient_id: alice.id,
  related_entity_id: reviewCase.id,
  notification_type: 'review_feedback'
});
expect(aliceNotif).toBeDefined();
```

---

### Phase 5: Rework & Resubmission (Alice)

#### Step 5.1: Receive Feedback Notification
**Actor**: Alice
**Action**:
- Receive notification: "Changes requested for 2025 Digital Transformation Vision"
- Open asset in AMEIDE
- Read Bob's feedback

**Expected Outcome**:
- Review comments visible
- Asset still in `rework` state
- Edit capability enabled

#### Step 5.2: Update Document
**Actor**: Alice
**Action**:
- Download original PDF
- Add financial impact section (pages 15-18)
- Re-upload updated PDF
- Update version notes: "Added financial impact analysis per reviewer feedback"

**Expected Outcome**:
- New AssetVersion created (version "1.1")
- Previous version retained for audit
- File checksum updated

**Test Assertion**:
```typescript
await loginAs(alice);
await page.goto(`/assets/${assetId}`);
await page.click('[data-testid="upload-new-version-btn"]');
await page.setInputFiles('[data-testid="file-upload"]', 'fixtures/architecture-vision-v1.1.pdf');
await page.fill('[data-testid="version-notes"]', 'Added financial impact analysis');
await page.click('[data-testid="save-version-btn"]');

const newVersion = await db.findOne('asset_versions', {
  asset_id: assetId,
  version_number: '1.1'
});
expect(newVersion).toBeDefined();
expect(newVersion.lifecycle_state).toBe('rework');
```

#### Step 5.3: Resubmit for Review
**Actor**: Alice
**Action**: Click "Resubmit for Review"

**Expected Outcome**:
- Same ReviewCase updated (not new case created)
- AssetVersion state: `rework` → `in_review`
- Bob notified of resubmission
- SLA timer does NOT reset (continues from original submission)

**Test Assertion**:
```typescript
await page.click('[data-testid="resubmit-review-btn"]');

const resubmittedVersion = await db.findOne('asset_versions', { asset_version_id: newVersion.id });
expect(resubmittedVersion.lifecycle_state).toBe('in_review');

// Same review case
const sameReview = await db.findOne('review_cases', { review_case_id: reviewCase.id });
expect(sameReview.state).toBe('in_progress'); // Back to in progress
expect(sameReview.asset_version_id).toBe(newVersion.id); // Updated to new version
expect(new Date(sameReview.sla_due_at).getTime()).toBe(new Date(reviewCase.sla_due_at).getTime());

// Notification
const resubmitNotif = await db.findOne('governance_notifications', {
  recipient_id: bob.id,
  related_entity_id: reviewCase.id,
  notification_type: 'review_resubmission'
});
expect(resubmitNotif).toBeDefined();
```

---

### Phase 6: Approval & Classification (Bob, Bob2)

#### Step 6.1: Bob Reviews Updated Version
**Actor**: Bob
**Action**:
- Receive resubmission notification
- Open review case
- Verify financial section added
- All checks still passing

**Expected Outcome**:
- Updated version visible
- Previous feedback marked as addressed

#### Step 6.2: Bob Approves
**Actor**: Bob
**Action**:
- Click "Approve"
- Rationale: "Financial impact section addresses previous concern. Vision is comprehensive and aligned with enterprise strategy. Ready for baseline classification."
- Submit approval

**Expected Outcome**:
- ApprovalDecision created (decision: `approve`)
- ReviewCase state: `in_progress` (quorum not yet met, needs 2 approvals)
- Quorum tracker: 1/2 approvals

**Test Assertion**:
```typescript
await loginAs(bob);
await page.goto(`/governance/reviews/${reviewCase.id}`);
await page.click('[data-testid="approve-btn"]');
await page.fill('[data-testid="rationale"]', 'Financial impact section addresses...');
await page.click('[data-testid="submit-approval-btn"]');

const approvals = await db.find('approval_decisions', {
  review_case_id: reviewCase.id,
  decision: 'approve'
});
expect(approvals).toHaveLength(1);

const inProgressReview = await db.findOne('review_cases', { review_case_id: reviewCase.id });
expect(inProgressReview.state).toBe('in_progress'); // Not resolved yet
```

#### Step 6.3: Bob2 Approves (Quorum Met)
**Actor**: Bob2 (second reviewer)
**Action**:
- Reviews asset
- Approves with rationale: "Concur with first reviewer. Approved."

**Expected Outcome**:
- Second ApprovalDecision created
- Quorum threshold met (2/2)
- ReviewCase state: `in_progress` → `resolved`
- ReviewCase resolution: `approved`
- AssetVersion state: `in_review` → `approved`
- ClassificationAssignment created: `Landscape.Baseline`
- Timestamp fields populated: `approved_at`, `approved_by_id`
- LifecycleEvent logged
- GovernanceMetricSnapshot updated
- RepositorySyncJob queued
- Notification sent to Alice

**Test Assertion**:
```typescript
await loginAs(bob2);
await page.goto(`/governance/reviews/${reviewCase.id}`);
await page.click('[data-testid="approve-btn"]');
await page.fill('[data-testid="rationale"]', 'Concur with first reviewer. Approved.');
await page.click('[data-testid="submit-approval-btn"]');

// Quorum met
const allApprovals = await db.find('approval_decisions', {
  review_case_id: reviewCase.id,
  decision: 'approve'
});
expect(allApprovals).toHaveLength(2);

// Review resolved
const resolvedReview = await db.findOne('review_cases', { review_case_id: reviewCase.id });
expect(resolvedReview.state).toBe('resolved');
expect(resolvedReview.resolution).toBe('approved');
expect(resolvedReview.resolved_at).toBeDefined();
expect(resolvedReview.resolved_by_id).toBe(bob2.id);

// Asset approved
const approvedVersion = await db.findOne('asset_versions', { asset_version_id: newVersion.id });
expect(approvedVersion.lifecycle_state).toBe('approved');
expect(approvedVersion.approved_at).toBeDefined();
expect(approvedVersion.approved_by_id).toBe(bob2.id);

// Classification assigned
const classification = await db.findOne('classification_assignments', {
  asset_version_id: newVersion.id
});
expect(classification).toBeDefined();
expect(classification.role).toBe('baseline');
expect(classification.scope).toBe('enterprise');
expect(classification.published_at).toBeDefined();

// Lifecycle event
const approvalEvent = await db.findOne('lifecycle_events', {
  asset_version_id: newVersion.id,
  to_state: 'approved'
});
expect(approvalEvent.actor_id).toBe(bob2.id);
expect(approvalEvent.evidence_links).toContain(reviewCase.id);

// Notification to Alice
const approvalNotif = await db.findOne('governance_notifications', {
  recipient_id: alice.id,
  related_entity_id: reviewCase.id,
  notification_type: 'review_outcome'
});
expect(approvalNotif).toBeDefined();

// Metrics updated
const metrics = await db.findOne('governance_metric_snapshots', {
  graph_id: enterpriseRepo.id
});
expect(metrics.approval_metrics.count).toBeGreaterThan(0);

// Sync job queued
const syncJob = await db.findOne('governance_background_jobs', {
  job_type: 'graph_sync',
  'arguments.asset_version_id': newVersion.id
});
expect(syncJob).toBeDefined();
```

---

### Phase 7: Verification & Consumption (Alice, Frank)

#### Step 7.1: Alice Views Approved Asset
**Actor**: Alice
**Action**:
- Receive approval notification
- Navigate to asset detail page

**Expected Outcome**:
- Lifecycle state badge: "Approved" (green)
- Classification badge: "Landscape.Baseline"
- Approval timestamp visible
- Approved by: Bob2
- Audit trail complete

**Test Assertion**:
```typescript
await loginAs(alice);
await page.goto(`/assets/${assetId}`);

await expect(page.locator('[data-testid="lifecycle-state"]')).toContainText('Approved');
await expect(page.locator('[data-testid="classification"]')).toContainText('Landscape.Baseline');
await expect(page.locator('[data-testid="approved-at"]')).toBeVisible();
```

#### Step 7.2: View in Repository Classification View
**Actor**: Alice
**Action**: Navigate to `/repositories/enterprise/classifications/landscape-baseline`

**Expected Outcome**:
- Asset appears in Landscape.Baseline smart folder
- Metadata visible
- Downloadable

**Test Assertion**:
```typescript
await page.goto('/repositories/enterprise/classifications/landscape-baseline');
const assetCard = page.locator(`[data-asset-id="${assetId}"]`);
await expect(assetCard).toBeVisible();
await expect(assetCard.locator('[data-testid="asset-name"]')).toContainText('2025 Digital Transformation Vision');
```

#### Step 7.3: Frank Views Baseline Landscape
**Actor**: Frank (Business Stakeholder)
**Action**:
- Login as Frank (read-only)
- Navigate to `/repositories/enterprise/landscape?view=baseline`

**Expected Outcome**:
- Baseline landscape view displayed
- "2025 Digital Transformation Vision" visible
- Can download artifacts
- Cannot edit or submit for review

**Test Assertion**:
```typescript
await loginAs(frank);
await page.goto('/repositories/enterprise/landscape?view=baseline');

await expect(page.locator('[data-testid="landscape-view"]')).toContainText('Baseline (Current State)');
await expect(page.locator(`[data-asset-id="${assetId}"]`)).toBeVisible();

// Verify read-only
await expect(page.locator('[data-testid="edit-btn"]')).not.toBeVisible();
await expect(page.locator('[data-testid="submit-review-btn"]')).not.toBeVisible();
```

---

## Complete Audit Trail

### Lifecycle Events (in order)
1. Asset created (actor: Alice)
2. Draft saved (actor: Alice)
3. Metadata updated (actor: Alice)
4. Submitted for review (actor: Alice, from_state: draft, to_state: in_review)
5. Changes requested (actor: Bob, from_state: in_review, to_state: rework)
6. Version updated (actor: Alice)
7. Resubmitted for review (actor: Alice, from_state: rework, to_state: in_review)
8. Approved (actor: Bob2, from_state: in_review, to_state: approved)
9. Classification assignment published (actor: System, metadata.classification_code: Landscape.Baseline)

### Repository Audit Events
- `asset.created`
- `asset.submitted_for_review`
- `review_case.opened`
- `approval_decision.changes_requested`
- `asset.version_updated`
- `asset.resubmitted`
- `approval_decision.approved` (x2)
- `review_case.resolved`
- `asset.approved`
- `classification_assignment.published`

---

## KPIs Captured

### Governance KPIs
- **Approval Cycle Time**: Time from submission to approval
  - Target: P50 < 24 hours, P95 < 48 hours
  - Actual: (measured during test execution)
- **Rework Rate**: % of reviews requiring changes
  - Baseline data point: 1 rework in this journey
- **Check Pass Rate**: % of required checks passing
  - Target: 100% blocking checks
  - Actual: 100% (all blocking checks passed)

### Repository KPIs
- **Classification Accuracy**: % of assets correctly classified
  - Target: 100%
  - Actual: 100% (PlacementRule enforced)
- **Sync Latency**: Time from approval to search index update
  - Target: P95 < 30 seconds
  - Actual: (measured via background job completion)

---

## FR/NFR Coverage

### Functional Requirements
- **FR-11**: Asset lifecycle management (draft → approved)
- **FR-12**: Governance owner group assignment
- **FR-13**: Review workflows with approval decisions
- **FR-14**: Required checks execution
- **FR-15**: SLA monitoring
- **FR-18**: Time-aware metadata (effective windows)
- **FR-20**: Audit trail completeness

### Non-Functional Requirements
- **NFR-Perf-01**: Asset creation < 2s
- **NFR-Perf-02**: Search index update < 30s
- **NFR-Sec-01**: Role-based access control enforced
- **NFR-Audit-01**: Complete audit trail logged

---

## Edge Cases Tested

### Implicit Coverage
1. **Concurrent edits**: Alice cannot edit while in review
2. **Version history**: Previous version (1.0) retained after creating 1.1
3. **Notification delivery**: All notifications successfully sent
4. **Quorum validation**: Cannot approve with only 1/2 approvals
5. **PlacementRule enforcement**: Cannot classify as Baseline unless approved

### Future Test Variations
- Asset rejected (not approved)
- SLA breach before approval
- Third reviewer abstains
- Blocking check fails
- Alice withdraws submission

---

## Related Journeys
- **Journey 233** (Initiative Workflow): Carol will reference this baseline asset
- **Journey 232** (Policy Recertification): This asset may require recertification if policy changes
- **Journey 235** (Landscape Toggle): Frank will compare this baseline to target state

---

## Why This Journey Matters

### For Product Owners
Validates the complete "happy path" for asset governance, ensuring:
- Users can create and enrich assets intuitively
- Review workflows are transparent and traceable
- Approvals enforce quality gates before publication
- Approved assets become discoverable baseline references

### For QA Engineers
Establishes baseline E2E test covering:
- Full asset lifecycle state machine
- Governance workflows with quorum
- Notification delivery
- Audit logging
- Metrics capture

### For Architects
Demonstrates:
- Integration between Asset, ReviewCase, and Classification entities
- State transition triggers and side effects
- Background job orchestration (sync, metrics)
- Event-driven notification system

---

## Test Data Cleanup

### Teardown Steps
```typescript
afterAll(async () => {
  // Delete in reverse dependency order
  await db.delete('classification_assignments', { asset_version_id: newVersion.id });
  await db.delete('approval_decisions', { review_case_id: reviewCase.id });
  await db.delete('review_cases', { review_case_id: reviewCase.id });
  await db.delete('lifecycle_events', { asset_version_id: { in: [version.id, newVersion.id] } });
  await db.delete('check_runs', { asset_version_id: { in: [version.id, newVersion.id] } });
  await db.delete('asset_versions', { asset_id: assetId });
  await db.delete('assets', { asset_id: assetId });
  await db.delete('governance_notifications', { related_entity_id: reviewCase.id });
  await db.delete('graph_audit_events', { entity_id: assetId });
});
```

---

## Revision History
- **2025-01-XX**: Initial creation
