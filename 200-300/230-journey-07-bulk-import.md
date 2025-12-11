# Journey 237 — Bulk Asset Import & Classification

## Overview
**Duration**: 8–10 minutes  
**Primary Personas**: Alice (Asset Owner), David (Repository Admin)  
**Supporting Personas**: Governance Lead, Automation Bot, Platform Analyst  
**Complexity**: High

**Business Value**: Validates large-scale asset onboarding with schema validation, placement rules, telemetry, and rollback safety, ensuring graph quality at scale.

---

## Prerequisites

### System Configuration
- Bulk Import API enabled with manifest schema (CSV/JSON) covering asset metadata, classification hints, RepositoryElement references.
- PlacementRules defined for target classifications, including subtree enforcement.
- Governance owner groups ready to review validation exceptions.
- Analytics pipeline prepared to ingest import telemetry (failure categories, counts).

### User Setup
- **David**: Repository Admin initiating import.
- **Alice**: Asset Owner reviewing results for her domain.
- Governance Lead to approve waivers if thresholds exceeded.
- Automation bot processing validations and sync jobs.

### Test Data
- Manifest file containing 100 assets (mix of new and updates).
- Sample includes RepositoryElement mappings, scenario tags, and tags.
- Some rows intentionally invalid (missing element references, policy violations) to test quarantine.

---

## User Story
**As** David (Repository Admin),  
**I want** to import a batch of assets with automated validation and clear remediation paths,  
**So that** large migrations maintain governance integrity without manual re-entry.

---

## Journey Steps

### Phase 1: Preflight Validation
1. David uploads manifest via Bulk Import UI/API.
2. System performs schema validation (data types, required fields) and simulates classification placement against PlacementRules.
3. Validation report summarises successes, warnings (e.g., missing optional tags), and blocking errors (invalid RepositoryElement).
4. David downloads error report, fixes manifest rows, and re-uploads.

**Expected Outcome**
- `ImportJob` created with status `validating` and manifest metadata recorded.
- Preflight must reach ≥95% success before import allowed.
- Warning categories documented with justification.
- Blocking errors quarantined with actionable messages via `ImportItem.state=quarantined`.

**Test Assertion**:
```typescript
const importJob = await db.findOne('import_jobs', { graph_id: enterpriseRepo.id, status: 'validating' });
expect(importJob.manifest_uri).toContain('bulk-manifests/2025-02');

const pendingItems = await db.find('import_items', { import_job_id: importJob.import_job_id });
expect(pendingItems.every(item => item.state === 'pending' || item.state === 'quarantined')).toBe(true);
```

### Phase 2: Execute Import
1. David confirms import; system processes in batches (e.g., 25 assets per batch).
2. For each batch:
   - New assets created with default governance owner/states.
   - Existing assets updated; new AssetVersions generated when content changed.
   - `ElementAttachment` and `TraceLink` relationships refreshed.
   - ClassificationAssignments applied respecting PlacementRules.
3. Successful rows emit `AssetSyncPayload` to downstream systems.

**Expected Outcome**
- Import job status transitions `running` → `completed` with `success_count` incremented.
- Successful `ImportItem` rows marked `processed` and backfilled with target IDs.
- Repository sync jobs triggered (knowledge graph, search) and linked via `sync_job_id` on the `ImportJob`.
- Telemetry records processed count, duration, anomaly flags.

**Test Assertion**:
```typescript
await waitFor(async () => {
  const job = await db.findOne('import_jobs', { import_job_id: importJob.import_job_id });
  return job.status === 'completed';
});

const processedItems = await db.count('import_items', { import_job_id: importJob.import_job_id, state: 'processed' });
expect(processedItems).toBeGreaterThan(0);

const syncJob = await db.findOne('graph_sync_jobs', { sync_job_id: importJob.sync_job_id });
expect(syncJob.status).toBe('queued');
```

### Phase 3: Quarantine Handling
1. Invalid rows stored in quarantine table with failure codes (e.g., `invalid_element`, `placement_violation`).
2. Automation bot notifies Governance Lead if failure rate exceeds threshold.
3. Governance Lead decides to retry, roll back, or escalate (ties into negative Journey 245 if threshold breached).
4. David corrects quarantined rows, reprocesses partial manifest (only failed rows).

**Expected Outcome**
- Quarantined `ImportItem` rows resolved (state `retried` → `processed`).
- Import job updates `failure_count` downward and records remediation in `last_error` history.
- GovernanceLead acknowledges incident; `GovernanceRecord` captures actions if thresholds exceeded.
- Metrics show improved success rate on retry and `ImportJob.status` stays `completed`.

**Test Assertion**:
```typescript
const quarantinedItems = await db.find('import_items', { import_job_id: importJob.import_job_id, state: 'quarantined' });
expect(quarantinedItems).toHaveLength(0);

const latestJob = await db.findOne('import_jobs', { import_job_id: importJob.import_job_id });
expect(latestJob.failure_count).toBe(0);
```

### Phase 4: Post-Import Review
1. Alice filters graph to newly imported assets, validates metadata, classifications, and RepositoryElement coverage.
2. Governance dashboards show placement compliance and traceability coverage.
3. Platform Analyst confirms sync latency within SLA and telemetry ingestion complete.

**Expected Outcome**
- New assets ready for authoring or review workflows.
- Data-quality score improved.
- Audit trail records manifest ID, actor, timestamps, counts via `ImportJob` and associated `ImportItem` entries.
- `RepositoryStatisticSnapshot` references the completed import job in its `source_job_id`.

---

## Telemetry & KPIs
- **Import Success Rate** (target ≥ 95% on final run).
- **Validation Error Categories** (track top causes).
- **Sync Latency** post-import (target < 2 min for downstream projections).
- **Classification Accuracy** (no PlacementRule violations post-import).
- **Quarantine Resolution Time** (target < 24h).

---

## FR/NFR Coverage
- **FR-6**: Asset creation/update at scale.
- **FR-8**: Bulk operations with validation.
- **FR-11**: Asset lifecycle versioning through import.
- **FR-18**: Classification enforcement.
- **FR-20**: Telemetry + audit reporting.
- **NFR-Perf-02**: Import duration and sync performance.
- **NFR-Rel-01**: Rollback/quarantine mechanisms.

---

## Related Journeys
- **Journey 231**: Validates single-asset lifecycle before bulk.
- **Journey 232**: Policy updates that may require re-import or recertification.
- **Journey 245**: Negative path when validation failures exceed tolerance.
- **Journey 235 (TJ1)**: Telemetry operations verifying sync success.

---

## Revision History
- **2025-02-XX**: Initial documentation of bulk import and classification journey.
