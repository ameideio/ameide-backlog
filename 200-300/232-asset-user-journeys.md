# Asset Lifecycle & Knowledge Graph Journeys

| Journey ID | Name | Epic | Primary Personas | Goal | Linked Specs |
| --- | --- | --- | --- | --- | --- |
| AJ1 (Journey 231) | Register & Seed New Asset | E1 Asset Lifecycle Management | Enterprise Architect, Contributor | Capture canonical metadata, subtype specifics, and governance steward for a new asset. | FR‑6, FR‑11; backlog/220-asset-entity-model.md |
| AJ2 | Co-create Draft & Evidence | E1 Asset Lifecycle Management | Contributor, Reviewer, Automation Bot | Collaboratively edit draft artifact, attach evidence, and prep for governance submission. | FR‑11, FR‑13, backlog/120 editors |
| AJ3 | Classify & Relate Asset | E4 Landscape & Reporting | Enterprise Architect, Solution Architect | Assign classifications, trace links, and scenario tags to position the asset in the graph. | FR‑18; backlog/220-asset-entity-model.md |
| AJ4 | Sync to Graph, Search, & Initiatives | E4 Landscape & Reporting | Platform Analyst, Product Owner | Ensure approved assets propagate to knowledge graph, search index, and transformation boards. | RepositorySyncJob; backlog/231 JR2/JR3 |
| AJ5 | Manage Retention & Sunset | E2 Governance Workflows | Governance Lead, Compliance Officer | Apply retention holds, purge schedule, and release archival notices while preserving audit trail. | AssetRetentionEvent; backlog/220-asset-entity-model.md |
| AJ6 (Journey 237) | Bulk Asset Operations & Validation | E1 Asset Lifecycle Management | Repository Admin, Asset Owner | Import/update assets in bulk with validation, rollback, and governance oversight. | FR‑6, FR‑8, FR‑18; backlog/238-extended-journeys.md#journey-245-bulk-import-with-validation-failures |

---

## AJ1 Register & Seed New Asset

**Epic**: E1 Asset Lifecycle Management  
**User Stories**:
- US-AJ1.1 As an EA, I can create a new asset with subtype-specific defaults.
- US-AJ1.2 As a Contributor, I can enrich subtype fields and tags before saving.
- US-AJ1.3 As a Governance Lead, I can see new assets automatically linked to owner groups.

**Personas**: Enterprise Architect (EA), Contributor (CT)  
**Steps**
1. EA selects “Create Asset”, chooses subtype (Requirement, Standard, Model, etc.), enters core metadata (name, description, originating phase, owning unit).
2. System hydrates defaults from graph (visibility, governance owner group, retention state) and loads relevant `RepositoryElement` templates from the ontology catalogue.
3. CT fills subtype-specific fields (e.g., requirement type, priority) and tags; optionally links existing RepositoryElements or creates element placeholders for modelling later.
4. EA saves asset; `Asset` row created with `retention_state=active`, `AssetVersion` draft seeded, default `TraceLink` stubs prepared, and `ElementAttachment` placeholders recorded for future updates.

**Outcomes**: Asset visible in graph drafts; governance owner assigned; initial RepositoryElement attachments captured; telemetry recorded for registration time.

---

## AJ2 Co-create Draft & Evidence

**Epic**: E1 Asset Lifecycle Management  
**User Stories**:
- US-AJ2.1 As a Contributor, I can edit drafts collaboratively with presence indicators.
- US-AJ2.2 As a Contributor, I can attach evidence artifacts and link external references.
- US-AJ2.3 As a Reviewer, I can leave comments and resolve them before formal submission.
- US-AJ2.4 As an Automation Bot, I can suggest required checks based on evidence attachments.

**Personas**: Contributor, Reviewer, Automation Bot  
**Steps**
1. CT opens collaborative editor (BPMN/ArchiMate/Markdown) within Artifact Host, edits content; changes synced via Yjs.
2. CT attaches evidence artifacts (test logs, diagrams) and references external systems; automation bot parses attachments to auto-suggest checks and highlight affected `RepositoryElement` nodes.
3. CT links model elements to Document sections via `ElementAttachment` updates so reviewers can navigate from diagram fragments to canonical elements.
4. CT requests peer review (non-governance) to refine before formal submission; inline comments resolved.
5. Draft timeline records edits and comments; readiness meter indicates metadata completeness and element coverage.

**Outcomes**: Draft prepared with evidence, gating checks suggested, RepositoryElement references curated, collaborative history stored.

---

## AJ3 Classify & Relate Asset

**Epic**: E4 Landscape & Reporting  
**User Stories**:
- US-AJ3.1 As an EA, I can assign classifications with scenario tags and effective windows.
- US-AJ3.2 As an EA, I receive placement rule validation results when saving assignments.
- US-AJ3.3 As a Solution Architect, I can create trace links to related assets and transformations.
- US-AJ3.4 As a Product Owner, I can view updated dependency overlays after classification.

**Personas**: EA, Solution Architect (SA)  
**Steps**
1. EA opens Classification panel, selects graph nodes, assigns `ClassificationAssignment` with scenario tags, windows, scope.
2. EA/SA select impacted `RepositoryElement` records (capabilities, value streams) and bind them via `ElementAttachment` so the classification references canonical ontology nodes.
3. SA creates `TraceLink` edges (satisfies, derives, depends_on) between this asset, related assets, transformations, and RepositoryElements.
4. Placement validation ensures asset state meets `PlacementRule` and ontology catalogue constraints (e.g., allowed element roles). Warnings require override justification.
5. Upon save, classification + trace edges stream via sync pipeline; ElementProjections and transformation boards update dependency overlays.

**Outcomes**: Asset contextually placed, scenario toggles ready, RepositoryElement graph enriched, dependency overlays refreshed.

---

## AJ4 Sync to Graph, Search, & Initiatives

**Epic**: E4 Landscape & Reporting  
**User Stories**:
- US-AJ4.1 As a Platform Analyst, I can monitor sync jobs for approved asset versions.
- US-AJ4.2 As a Product Owner, I see transformation dashboards update with release status.
- US-AJ4.3 As a Search Consumer, I can discover asset versions with updated facets.
- US-AJ4.4 As a Knowledge Graph Consumer, I can explore new nodes/edges produced by the asset sync.

**Personas**: Platform Analyst (PA), Product Owner  
**Steps**
1. Once asset version approved, `RepositorySyncJob` packages `AssetSyncPayload` including RepositoryElement deltas and dispatches to knowledge graph/search endpoints.
2. PA monitors sync dashboard; verifies knowledge graph ingestion (new element nodes/edges) and search index facet (language, subtype).
3. ElementProjections regenerate (capability maps, value streams) and transformations receive SSE update; board chips display release status and link to knowledge graph impact view.
4. PA exports `AssetSearchDocument` entry for QA; ensures boost score aligns with governance signals and RepositoryElement prominence.

**Outcomes**: Asset discoverable everywhere with consistent metadata; RepositoryElement projections refreshed; transformations aligned.

---

## AJ5 Manage Retention & Sunset

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-AJ5.1 As a Governance Lead, I can place retention holds with rationale and expiry.
- US-AJ5.2 As a Governance Lead, I can schedule purges and preview affected artifacts.
- US-AJ5.3 As a Compliance Officer, I can review and export retention event history.
- US-AJ5.4 As a Product Owner, I receive notifications when retained assets impact readiness.

**Personas**: Governance Lead, Compliance Officer, Product Owner  
**Steps**
1. Governance review identifies asset for retirement; GL applies retention hold or purge schedule via Asset settings and reviews affected `RepositoryElement` attachments.
2. System logs `AssetRetentionEvent`, notifies owner groups and POs, and updates graph dashboard including element impact summaries.
3. At retention expiry, system archives asset versions (`archived_at` set), deprecates release tags, updates search index/search documents, removes obsolete ElementAttachments/Projections, and enforces access changes.
4. Compliance officer exports audit trail (retention events, lifecycle transitions, element removals) for evidence; PO verifies readiness dashboards reflect archival status.

**Outcomes**: Asset sunset governed, RepositoryElement inventory updated, audit trail intact, downstream systems aware of removal.

**Telemetry & KPIs**: Count of assets under hold, time to purge completion, number of retention override requests, readiness risk notifications.

---

## AJ6 (Journey 237) Bulk Asset Operations & Validation

**Epic**: E1 Asset Lifecycle Management  
**User Stories**:
- US-AJ6.1 As a Repository Admin, I can upload bulk manifests (CSV/JSON) describing new or updated assets.
- US-AJ6.2 As a Repository Admin, I receive preflight validation highlighting conflicts, missing fields, and placement violations.
- US-AJ6.3 As an Asset Owner, I can approve or reject proposed changes before they commit.
- US-AJ6.4 As a Governance Lead, I can monitor bulk job progress, including partial failures and rollback actions.
- US-AJ6.5 As a Developer, I can view audit entries detailing which assets changed due to the bulk operation.

**Personas**: Repository Admin, Asset Owner, Governance Lead  
**Preconditions**: Bulk import feature flag enabled; asset schema + validation rules configured; staging environment available for dry runs.  
**Postconditions**: Assets created/updated according to manifest with validation recorded; failed rows quarantined; audit + telemetry captured.

**Steps**
1. Repository Admin downloads manifest template (schema includes asset metadata, classification hints, trace links, RepositoryElement references).
2. Admin populates manifest and uploads via Bulk Import UI/API; system performs schema + ontology validation and placement rule simulation (no writes yet).
3. Preflight report surfaces errors/warnings (including invalid element mappings); admin fixes manifest or accepts non-blocking warnings with justification.
4. Admin submits import; system processes in batches, emitting progress events. Successful records create/update assets, spawn new `AssetVersion`s, refresh `ElementAttachment`s, and queue classification/element syncs. Failures routed to quarantine table for manual resolution.
5. Asset Owners receive summary for their scope; can accept or roll back specific changes within retention window, including RepositoryElement updates.
6. Governance Lead reviews import telemetry (processed count, failures, SLA, element coverage) and triggers Journey 245 if failure rate exceeds threshold.

**Success Signals**
- ≥ 95% records succeed on first attempt; remaining routed with actionable errors.
- Audit log captures manifest ID, actor, timestamps, affected asset IDs, and RepositoryElement counts.
- Repository and transformation views reflect changes within expected sync SLA; element projections refreshed.

**Telemetry & KPIs**: Import success rate, validation error categories, time-to-resolve quarantined rows, sync latency post-import.

**Related Negative Path**: Journey 245 Bulk Import with Validation Failures elaborates remediation when imports exceed error thresholds.

---

## AJ5 Manage Retention & Sunset

**Personas**: Governance Lead, Compliance Officer  
**Steps**
1. Governance review identifies asset for retirement; GL applies retention hold or purge schedule via Asset settings and reviews impacted `RepositoryElement` attachments.
2. System logs `AssetRetentionEvent`, notifies owner groups, and updates graph dashboard, including RepositoryElement impact summaries.
3. Once retention window ends, asset version archived (`archived_at` set), release tags deprecated, search document removed (while preserving audit records), and element attachments retired.
4. Compliance officer exports audit trail (retention events, lifecycle transitions, element retirements) for evidence.

**Outcomes**: Asset sunset governed, RepositoryElement inventory updated, audit trail intact, downstream systems aware of removal.
