# Enterprise Repository Business Scenarios (PO-first)

## Orientation
The Enterprise Architecture graph is the tenant-wide system of record for approved architecture assets, structured according to TOGAF 9.2 (Architecture Capability, Metamodel, Landscape Baseline/Target/Transition, Standards Information Base, Reference Library, Requirements Repository, and Governance Log). Repository configuration now includes:
- **Governance Bundles**: `GovernancePolicy`, `PlacementRule`, and `RequiredCheck` cataloged once per graph.
- **Access Fabric**: `RepositoryAccessRole` + `RepositoryMembership` supply least-privilege access and auto-link stakeholders to the right `GovernanceOwnerGroup`.
- **Operational Telemetry**: `RepositorySyncJob` publishes changes to knowledge graph, search, and analytics endpoints; `RepositoryStatisticSnapshot` captures queue, approval, and SLA health.

The graph still operates in a three-context model:
1. **Enterprise Repository** – Organization-level TOGAF structure containing approved, governed assets, including releases and sync telemetry.
2. **Initiative Workspace** – ADM phase structure (Preliminary, A–H, Requirements Management) for work-in-progress deliverables.
3. **Initiative Repository View** – Filtered, read-only references showing which approved assets an transformation consumes, including release versions and retention flags.

Governance—`GovernanceOwnerGroup`, approval rules, required checks, routing rules, promotion queue—lives at graph/tenant scope and applies to every artifact review. Transformation transformations consume graph truth (assignments, effective windows, releases, governance outputs) without redefining policy. The scenarios below describe the PO-ready playbook for operating the graph with the updated governance and telemetry fabric.

---

## Scenario A1. Govern Once, Enforce Everywhere
- **Trigger:** New graph or governance refresh.
- **Owner:** Governance Lead (configures); Product Owner (consumes outcomes).
- **Flow:** Provision `RepositoryAccessRole` templates → assign memberships → configure `GovernancePolicy` (rules, playbook, retention, SLA) → link `GovernanceOwnerGroup` and `PlacementRule` → publish governance profile banner. From this point, any artifact requesting review automatically inherits guardrails, routing rules, and SLA expectations.
- **PO Value:** Initiative boards surface the same required checks/approvals with zero per-transformation setup; graph dashboard exposes governance readiness via `RepositoryStatisticSnapshot`.
- **KPIs:** % artifacts blocked by missing owner group; approval cycle time P50/P95; required check pass rate; SLA breach count.
- **Spec references:** FR‑12..16, FR‑20; Repository Stage 3 (Governance Deep Dive).

## Scenario A2. Draft → In Review → Approved (Artifact-centric)
- **Trigger:** Architect requests review on an artifact tied to an transformation.
- **Owner:** Artifact Owner (executes); PO monitors status.
- **Flow:** `Request review` → reviewers auto-assigned from `GovernanceOwnerGroup` membership (fed by `RepositoryAccessRole`) → required checks execute with `CheckRun` logging → approvals collected via `ApprovalDecision` → promotion queue validates `PlacementRule` and SLA → `LifecycleEvent` set to `approved`; `RepositorySyncJob` publishes AssetVersion updates.
- **PO Value:** Board cards display a single chip summarizing checks, approvals, queue position, and sync status (search/graph).
- **KPIs:** Approval cycle time P50/P95; queue wait P95; % artifacts with `release_package_id`; sync success rate.
- **Spec references:** FR‑11, FR‑13, FR‑14, FR‑15; Repository Stages 2–4.

## Scenario A3. Time Windows & Scenarios at the Source
- **Trigger:** EA prepares As-Is/To-Be or scenario planning data.
- **Owner:** Enterprise Architect (records data); PO consumes toggles in transformations.
- **Flow:** EA sets `effective_from`, `effective_to`, `scenario_tag`, and `visibility` on `ClassificationAssignment`; smart folders expose the metadata; transformations consume via `InitiativeReference`. `RepositorySyncJob` emits `AssetSyncPayload` so transformation dashboards update automatically.
- **PO Value:** Initiative scenario toggles are trustworthy because the graph is the canonical store.
- **KPIs:** % transformation-linked assignments with windows; variance between As-Is/To-Be counts; scenario toggle latency (graph → transformation sync); % assignments auto-validated by `PlacementRule`.
- **Spec references:** FR‑18; graph DDL (`graph_assignments`); Stage 5 Time-Aware Modeling.

## Scenario A4. Release the Decision
- **Trigger:** Artifact reaches `approved` state.
- **Owner:** Architect (publishes release); PO shares readiness.
- **Flow:** Publish Release (notes + assets) → protected tags generated → `AssetVersion` flagged with release metadata → `RepositorySyncJob` dispatches release payload to search/analytics → transformations pull release data into readiness reports with synced status badges.
- **PO Value:** Release readiness reports draw directly from graph releases, required checks, and sync telemetry to confirm dissemination.
- **KPIs:** Releases per milestone; % cards released; download counts per release asset; release sync success latency.
- **Spec references:** FR‑15, governance dashboards, Stage 4 collaboration.

## Scenario A5. Data Quality & Recertification
- **Trigger:** Missing metadata or policy change.
- **Owner:** Governance Lead; PO tracks risk.
- **Flow:** Data-quality surveys enter queue with `RepositoryStatisticSnapshot` context → responses feed `GovernanceMetricSnapshot` → recertification jobs push approved items back to `in_review`, logging `AssetRetentionEvent` and `SlaBreachEvent` where needed → downstream systems updated via `RepositorySyncJob`.
- **PO Value:** Repository flags “Needs data” or “Recertified” so transformations can adjust plans.
- **KPIs:** Data-quality score trend; overdue surveys; recertification cycle time; # recertified artifacts synced within SLA.
- **Spec references:** FR‑20; Stage 4.

---

## Edge Cases & Playbooks
- **Scenario Collision:** Setting overlapping windows for the same artifact/scenario triggers validation (optional `btree_gist` exclusion). Queue refuses promotion until one window is adjusted. *Owners:* PO + Governance Lead.
- **Queue Contention:** Multiple artifacts with identical priority remain queued; graph queue shows FIFO position and responsible owners. PO coordinates priority adjustments with Governance Lead.

---

## Acceptance Tests (Repository-backed, PO-centric)
1. **Blockers Funnel (PO view):** Given mixed governance states, when a PO filters “Needs approvals” via transformation board, then graph events update card chips and funnel counts within 30 s, including deep links to artifact timelines and latest `RepositorySyncJob` status. *(FR‑12/13; Repository Stage 3)*
2. **Recertification Trigger:** Given a policy change, when recertification pushes an approved artifact back to `in_review`, then graph marks the artifact, logs the reason, emits `AssetRetentionEvent`, and transformations display the risk flag and sync timestamp in readiness reports. *(FR‑20; Stage 4)*

---

## KPIs & Telemetry Checklist
- Search P95 < 2.5 s; tree fetch P95 < 300 ms.
- Approval cycle time P50/P95; queue wait P95; SLA breach count (per `SlaBreachEvent`).
- % artifacts with pinned releases; release sync latency P95; data-quality score ≥ threshold.
- Collision count/time-to-resolution; survey completion rate; RepositorySyncJob success rate.

---

## Why This Fits the Specs
- Governance configured once at graph scope (FR‑12..16) applies everywhere.
- Lifecycle remains artifact-level; discussions and review requests stay artifact-specific.
- Initiatives consume graph truth (assignments, windows, releases, governance outputs) without duplicating policies.
- Metrics and acceptance tests tie directly to implementation stages (2–5), providing ready-made definition of done for POs and engineering.
