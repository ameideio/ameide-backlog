# Repository User Journeys

| Journey ID | Name | Epic | Primary Personas | Goal | Linked Specs |
| --- | --- | --- | --- | --- | --- |
| JR1 | Provision Governance-Ready Repository | E2 Governance Workflows | Governance Lead, Product Owner, Repository Admin | Stand up a graph with governance bundles, access roles, and telemetry sinks configured. | FR‑1, FR‑2, FR‑12..16; backlog/150 A1; backlog/220-graph-entity-model.md |
| JR2 | Execute Artifact Review Lifecycle | E1 Asset Lifecycle Management, E2 Governance | Artifact Owner, Reviewer, Governance Lead | Move an asset version from draft to approved while honoring checks, queueing, and sync propagation. | FR‑6, FR‑11..15; backlog/150 A2; backlog/220-asset-entity-model.md |
| JR3 | Maintain Time-Aware Classification | E4 Landscape & Reporting | Enterprise Architect, Product Owner | Capture scenario tags, effective windows, and placement validation for assignments. | FR‑18; backlog/150 A3; backlog/220-asset-entity-model.md |
| JR4 | Publish Release & Broadcast | E2 Governance Workflows | Architect, PO, Dev Consumer | Package approved work into releases and ensure downstream systems ingest the update. | FR‑15; backlog/150 A4; backlog/220-graph-entity-model.md |
| JR5 | Drive Data Quality & Recertification | E2 Governance Workflows | Governance Lead, Compliance Officer | Detect metadata gaps, trigger surveys, recertify impacted assets, and surface telemetry. | FR‑20; backlog/150 A5; backlog/220-governance-entity-model.md |

---

## JR1 Provision Governance-Ready Repository

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-JR1.1 As a Repository Admin, I can create a graph from a template with default metadata.
- US-JR1.2 As a Governance Lead, I can attach a governance policy and placement rules during setup.
- US-JR1.3 As a Governance Lead, I can assign graph access roles and memberships.
- US-JR1.4 As a Repository Admin, I can validate integration endpoints with test sync jobs.
- US-JR1.5 As a Product Owner, I can review governance readiness metrics before enabling the graph.

**Personas**: Governance Lead (GL), Product Owner (PO), Repository Admin (RA)  
**Preconditions**: Tenant has available graph slot; governance profile templates exist.  
**Postconditions**: Repository is active with policies, access roles, sync endpoints, and statistic snapshots scheduled.

1. RA selects template (Standards Catalog / Reference Library) and provides metadata (name, slug, visibility, locale, default time zone, default review SLA). System provisions graph (`RepositoryAdminService.Create`) and seeds default tree.
2. GL activates the ontology catalogue and classification hierarchy (parent/child, synonyms) so policies can target entire capability trees; baseline ElementProjections synchronise for capability/value maps.
3. GL configures governance bundle: selects `GovernancePolicy` version, attaches `PlacementRule` library (enabling subtree enforcement), assigns required checks, and links default `GovernanceOwnerGroup`.
4. GL maps personas to `RepositoryAccessRole` (admin, curator, reviewer) and invites members. Acceptances create `RepositoryMembership` rows and, when flagged, auto-enroll members in owner groups.
5. RA validates integration endpoints (search, knowledge graph, analytics). Each endpoint triggers a test `RepositorySyncJob`; status displays in setup wizard.
6. PO reviews governance banner (rules summary, SLA meters, TracePolicy coverage) and confirms graph readiness. System generates initial `RepositoryStatisticSnapshot` and publishes RepositoryElement coverage metrics for change/strategy dashboards.

**Success Signals**
- Governance coverage meter ≥ 90%, ontology catalogue approved, classification hierarchy activated without conflicts.
- Initial sync job succeeded; knowledge graph + search status = “Active”; RepositoryElement projections available.
- TracePolicy coverage initialised; audit log entries created for policy activation, role assignments, and ontology/classification setup.

**Telemetry & KPIs**: Provision time < 10 min; governance coverage; SLA defaults stored.

---

## JR2 Execute Artifact Review Lifecycle

**Epic**: E1 Asset Lifecycle Management / E2 Governance Workflows  
**User Stories**:
- US-JR2.1 As an Artifact Owner, I can request governance review for a draft version.
- US-JR2.2 As a Reviewer, I receive notifications and can approve or request changes in queue order.
- US-JR2.3 As a Governance Lead, I see check status and SLA timers for review cases.
- US-JR2.4 As an Artifact Owner, I can resolve required check failures and rerun checks.
- US-JR2.5 As a Product Owner, I see board chips update within SLA after approvals.

**Personas**: Artifact Owner (AO), Reviewer (RV), Governance Lead  
**Preconditions**: Asset exists with draft `AssetVersion`; owner group + required checks are configured.  
**Postconditions**: AssetVersion reaches `approved`, release candidate available, sync jobs dispatched.

1. AO enriches metadata, attaches evidence, and requests review (`ArtifactCommandBus.RequestReview`). System creates `ReviewCase` with SLA and queue item.
2. AO maps impacted `RepositoryElement` records through `ElementAttachment`; preview graph shows capability/application touchpoints that reviewers must assess.
3. Owner assignments fetch from linked `GovernanceOwnerGroup`; reviewers receive `GovernanceNotification` and appear in queue sidebar.
4. Required checks run automatically or manually; `CheckRun` status cards update in governance panel, including TracePolicy compliance and control coverage. AO resolves failures; reruns recorded.
5. Reviewers submit `ApprovalDecision`; segregation-of-duties enforced via role hints. Queue state monitors SLA; if breach imminent, `SlaBreachEvent` triggers.
6. Once checks + approvals clear, queue finalizes transition, emits `LifecycleEvent` (`approved`), updates linked RepositoryElements, and triggers `RepositorySyncJob` to broadcast updated asset version + element projections.
7. PO sees board chip updating (approvals, checks, sync status) within 30 s. AO optionally pins version or creates release (hand-off to JR4).

**Success Signals**
- Queue item processed without overrides.
- Check runs status = passed/waived with evidence; TracePolicy coverage satisfied.
- RepositoryElement impact graph refreshed; sync job status = succeeded; asset appears in search/graph.

**Telemetry & KPIs**: Approval cycle P50/P95; queue wait; sync latency; % decisions on-time.

---

## JR3 Maintain Time-Aware Classification

**Epic**: E4 Landscape & Reporting  
**User Stories**:
- US-JR3.1 As an Enterprise Architect, I can assign scenario tags and windows to an asset version.
- US-JR3.2 As an Enterprise Architect, I receive placement rule validation feedback inline.
- US-JR3.3 As a Product Owner, I can toggle transformation scenarios and see the updated assignments.
- US-JR3.4 As a Platform Analyst, I can verify sync payloads for classification changes.

**Personas**: Enterprise Architect (EA), Product Owner  
**Preconditions**: Approved asset versions exist; assignment templates configured.  
**Postconditions**: Assignments include accurate scenario tags, windows, placement validation, and sync metadata.

1. EA opens classification drawer, selects assignments (baseline/target/transition). Sets `effective_from`, `effective_to`, `scenario_tag`, `visibility`, and scope.
2. System validates against `PlacementRule` (lifecycle state, scope compatibility). Violations show remediation guidance; EA adjusts asset metadata if required.
3. EA links classification updates to affected `RepositoryElement` projections so capability/value-stream maps stay aligned.
4. EA confirms classification; `ClassificationAssignment` recorded with `placement_source=manual` and `last_validated_at` timestamp.
5. `RepositorySyncJob` picks up assignment change, generates `AssetSyncPayload`, updates search facets + knowledge graph edges (TraceLink updates) and rehydrates ElementProjections.
6. Initiatives referencing assignment receive SSE notifying scenario toggle updates; PO toggles As-Is/To-Be to confirm new window.

**Success Signals**
- Assignment passes validation with no warning or purposeful override.
- Sync job marks target systems as updated; ElementProjections regenerate; scenario toggles reflect change in <300 ms cached / <2 s P95 live.
- Repository analytics reflect updated baseline/target counts.

**Telemetry & KPIs**: % assignments with windows; placement validation success rate; scenario toggle latency.

---

## JR4 Publish Release & Broadcast

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-JR4.1 As an Architect, I can create a release package with selected approved versions.
- US-JR4.2 As a Governance Lead, I ensure releases respect retention and tagging policies.
- US-JR4.3 As a Platform Analyst, I confirm release sync jobs reach all endpoints.
- US-JR4.4 As a Developer, I discover releases through search/smart folders with metadata.

**Personas**: Architect, Product Owner, Developer Consumer  
**Preconditions**: AssetVersion in `approved`; required checks passed; release package template available.  
**Postconditions**: Release package published, protected tag created, downstream consumers notified.

1. Architect opens Releases tab, selects approved versions, adds release notes, readiness checklist results, and artifacts bundle (URL or attachments).
2. System marks AssetVersion with `release_package_id`, generates protected semantic tag, and enforces retention policies while snapshotting referenced `RepositoryElement` states.
3. `RepositorySyncJob` emits release payload to search (update release facet), knowledge graph (release node), ElementProjections (release overlays), and analytics (release metrics).
4. Initiatives ingest release via readiness dashboard; board cards show “Released” with timestamp, checks, and download links.
5. Developer consumer accesses release via search/smart folder; confirms signature/protection; optionally subscribes to release channel.

**Success Signals**
- Release status = `published`; protected tag verified.
- Sync job succeeded across endpoints; release metrics added to statistic snapshot.
- Readiness dashboard shows 100% aligned scope with release versions.

**Telemetry & KPIs**: Releases per milestone; release adoption (downloads/views); sync success latency.

---

## JR5 Drive Data Quality & Recertification

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-JR5.1 As a Governance Lead, I can launch targeted data-quality surveys.
- US-JR5.2 As an Asset Owner, I respond to survey prompts with metadata updates.
- US-JR5.3 As a Governance Lead, I trigger recertification jobs when policy changes occur.
- US-JR5.4 As a Product Owner, I see readiness cards reflect recertification progress.
- US-JR5.5 As a Compliance Officer, I export audit trails for recertification outcomes.

**Personas**: Governance Lead, Compliance Officer, Product Owner  
**Preconditions**: Repository governance profile active; surveys + recertification job types configured.  
**Postconditions**: Metadata gaps resolved, recertifications completed, telemetry updated.

1. Governance dashboard flags low data-quality score; GL launches survey targeting affected classifications (`DataQualitySurvey` with snapshot context) and highlights impacted `RepositoryElement` clusters and associated controls.
2. Owners receive notifications; submit `SurveyResponse` including flagged follow-ups. Responses roll up into `GovernanceMetricSnapshot`, update element-level quality indicators, and identify Control gaps.
3. Policy change or control failure triggers `RecertificationJob`; impacted AssetVersions revert to `in_review`, generating `AssetRetentionEvent`, updating linked RepositoryElements, and notifying transformations.
4. AO/RV follow JR2 lifecycle path to reconfirm approvals/checks under new policy.
5. Completion updates graph + governance metric snapshots; SlaBreach events resolved; Outcome/Telemetry joins show improved element quality; PO sees readiness cards switch from “Recertifying” to “Green”.

**Success Signals**
- Survey completion ≥ threshold; data-quality, RepositoryElement quality, and control coverage scores improve.
- Recertified assets back to `approved` within SLA.
- Telemetry dashboards and OutcomeMetricSnapshots display resolved flags; transformations show sync timestamp.

**Telemetry & KPIs**: Data-quality score trend; recertification cycle time; % recertified items synced; survey completion rate.
