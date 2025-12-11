# Journey 243 — Cross-Repository Collaboration

## Overview
**Duration**: 8–10 minutes  
**Primary Personas**: Alice (Enterprise Architect), Carol (Program Manager)  
**Supporting Personas**: Governance Lead, Automation Bot  
**Complexity**: High

**Business Value**: Validates linking assets across repositories using `CrossRepositoryReference`, enforcing shared governance, and synchronising telemetry so multi-repo transformations stay coherent.

---

## Prerequisites
- Repository A: “Enterprise Repository”.
- Repository B: “Payments Repository”.
- Both repositories share compatible classifications and have cross-repo linking enabled.
- Target asset in Repository B (`paymentServiceBlueprint`) approved and visible to Repository A.
- Dual-approval routing rules configured for cross-repo references.

---

## Journey Steps

### Phase 1: Create Cross-Repository Reference (Alice)
1. From Repository A, Alice selects “Link External Asset”.
2. Searches Repository B catalogue, chooses `paymentServiceBlueprint`.
3. Provides linkage type `consumes` and justification.
4. System creates `CrossRepositoryReference` with status `draft` and triggers approval workflows in both repositories.

**Expected Outcome**
- Reference row stored with `relationship_type=consumes` and `sync_strategy=bidirectional`.
- `governance_policy_links` populated for both repositories.
- Review cases opened in Repository A and Repository B.

**Test Assertion**
```typescript
const reference = await db.findOne('cross_graph_references', {
  source_graph_id: enterpriseRepo.id,
  target_graph_id: paymentsRepo.id,
  target_asset_id: paymentServiceBlueprint.id
});
expect(reference.relationship_type).toBe('consumes');
expect(reference.status).toBe('draft');
```

### Phase 2: Dual Approval (Governance Leads)
1. Governance lead in Repository A approves the reference after verifying placement rules.
2. Governance lead in Repository B reviews and approves sharing of `paymentServiceBlueprint`.
3. System updates reference status to `active`, logs matching `GovernanceRecord` entries, and sends confirmation notifications.

**Expected Outcome**
- Reference status `active` with metadata noting both approvers.
- Review cases resolved in each graph.
- Notifications confirm activation (`notification_type=review_outcome`).

### Phase 3: Initiative Consumption (Carol)
1. Carol scopes transformation in Repository A and includes the linked asset.
2. Initiative workspace pulls metadata from Repository B in read-only mode.
3. `TraceLink` created from transformation deliverable to external asset via the cross-repo reference.

**Expected Outcome**
- Initiative board shows external asset with `read_only=true`.
- `TraceLink.trace_type=depends_on` referencing cross-repo asset.
- Initiative flagged `multi_repo=true` in analytics metadata.

### Phase 4: Telemetry Synchronisation
1. `RepositorySyncJob` publishes CrossRepositoryReference updates to knowledge graph and search.
2. Telemetry join merges metrics from both repositories for the transformation.
3. Dashboards display unified blockers, approvals, and readiness metrics across repos.

**Expected Outcome**
- Sync job success event logs `cross_graph_references_processed` count.
- Telemetry join row includes both graph IDs and transformation ID.
- Portfolio view flag `multi_repo=true` for the transformation.

**Test Assertion**
```typescript
const joinRow = await db.findOne('telemetry_join_view', {
  transformation_id: transformation.id,
  multi_repo: true
});
expect(joinRow.graph_ids.sort()).toEqual([enterpriseRepo.id, paymentsRepo.id].sort());
```

---

## Telemetry & KPIs
- Number of active cross-graph references.
- Approval cycle time for references.
- Initiative delivery health across repositories.
- Sync latency for cross-repo updates.

---

## FR/NFR Coverage
- **FR-11**: Asset consumption across repositories.
- **FR-18**: Traceability and classification alignment.
- **FR-20**: Governance audit trail across multiple repositories.
- **NFR-Obs-01**: Cross-repo telemetry visibility.

---

## Related Journeys
- **Journey 233**: Initiative workflows now referencing multi-repo assets.
- **Journey 235**: Landscape toggle includes cross-repo projections.
- **Journey 238**: Data quality survey coverage extends to linked assets.

---

## Revision History
- **2025-02-XX**: Initial draft describing cross-graph collaboration workflows.
