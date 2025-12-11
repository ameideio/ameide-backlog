# Transformation Initiative Business Scenarios (PO-first)

## Orientation
Transformation transformations coordinate people, plans, and scenarios across repositories. Governance (owners, approvals, required checks, rulesets, promotion queue) is owned by the graph; transformations consume governance outputs, surface blockers, and drive Go/No-Go decisions. The Product Owner (PO) is the primary consumer—they need clear views of blockers, readiness, and scenario impact.

---

## Scenario B1. Spin Up an Initiative with a PO
- **Trigger:** Executive kickoff for “Payments Modernization 2025.”
- **Owners:** Sponsor & Program Manager (create); PO assigned at creation.
- **Flow:** Create transformation from template (milestones) → link graph → assign roles (Sponsor, PM, **PO**, Lead Architect, Reviewers, Contributors, optional AI reviewers).
- **PO Value:** PO becomes default owner for board filters, readiness gates, and notifications.
- **KPIs:** Time to fully staff transformation; role coverage (% roles filled).
- **Spec references:** Initiative business persona, functional objectives (roles), IA tables (transformations, milestones, role assignments), Stage 1 lifecycle & roles.

## Scenario B2. Scope the Work from the Repository
- **Trigger:** Planning sprint (week 0–2).
- **Owners:** Lead Architect + PO.
- **Flow:** Link existing graph artifacts (current state) to milestones; create drafts for proposed work (graph assignments carry lifecycle windows/scenarios); transformation board surfaces “Existing vs New.”
- **PO Value:** Clear visibility into baseline vs new scope without shadow artefacts.
- **KPIs:** % scope linked to existing artifacts; # new drafts created; graph coverage ratio.
- **Spec references:** Initiative artifact links, graph FR‑6/18.

## Scenario B3. As-Is / To-Be / Scenario Toggle (PO Demo View)
- **Trigger:** Steering committee review.
- **Owner:** PO.
- **Flow:** PO toggles As-Is/To-Be/Scenario; workspace queries graph assignment windows & scenario tags via `assignment_id`; dependency overlays use `model_edges`; results cached for <300 ms perceived latency (lite) / <2 s P95 with full windows.
- **PO Value:** One click shows what changes, when, and who is affected.
- **KPIs:** Toggle latency (steady/P95), impacted system counts, unresolved dependency count.
- **Spec references:** FR‑18, graph Stage 5, transformation Stages 3 & 5.

## Scenario B4. Approval & Checks Triage (PO Blockers Funnel)
- **Trigger:** Weekly board review.
- **Owner:** PO.
- **Flow:** Filter “Needs approvals”, “Failing checks”, “In queue”; graph events refresh chip counts; PO assigns owners or reorders queue priority via governance lead.
- **PO Value:** Actionable list of who must act next; supports stand-ups and executive updates.
- **KPIs:** Time-to-unblock; approvals outstanding per week; % checks passing after rerun.
- **Spec references:** Repository FR‑12/13/14 outputs; transformation board UI module; Stage 3 (boards & roadmaps).

## Scenario B5. Release Readiness (Go/No-Go)
- **Trigger:** Milestone approaching Launch.
- **Owner:** PO (decision facilitator).
- **Flow:** Initiative workspace compiles release readiness report: approved & released versions, required check status, approvals list, queue history, release notes; export PDF/CSV.
- **PO Value:** Evidence-based Go/No-Go in two clicks.
- **KPIs:** % scope released; # critical checks remaining; documentation completeness.
- **Spec references:** Repository FR‑15; transformation KPI pane; Stage 4.

---

## Future & Portfolio Scenarios
### B6. Directory Change & Role Reassignment (Future)
- **Trigger:** Product Owner changes in corporate directory.
- **Owners:** Program Manager + incoming PO.
- **Flow:** Directory sync updates `transformation_role_assignments`; workspace flags gaps; notifications prompt acceptance; board filters switch to new PO.
- **PO Value:** No loss of accountability mid-stream.
- **Spec references:** Initiative directory integration (Stage 5), IA role assignments.

### B7. Portfolio Heatmap (Multi-Initiative View)
- **Trigger:** Quarterly planning.
- **Owners:** Sponsor/PMO; POs provide context.
- **Flow:** Portfolio dashboard aggregates milestone health, blocker counts, approval cycle time, readiness per transformation; drill-down links to blockers funnel.
- **PO Value:** Prioritization and capacity planning with governance-backed metrics.
- **Spec references:** Initiative Stage 5 portfolio views, governance event consumption.

---

## Acceptance Test (PO Blockers Funnel)
- **Scenario:** Given a board with mixed governance states, when the PO filters “Needs approvals,” then the cards show owner chips, deep links to artifact timelines, and counts update within 30 s of graph events. *(Repository FR‑12/13, Initiative Stage 3)*

---

## KPIs & Telemetry Checklist
- Board column load / scenario toggle <300 ms perceived; P95 <500 ms under 95th percentile tenant load.
- Approval cycle time P50/P95; queue wait P95; blockers resolved per week.
- Release readiness completeness (% artifacts released, checks cleared).
- Role coverage, directory sync resolution time.
- Impact delta (As-Is vs To-Be) per scenario.

---

## Why This Fits the Specs
- Initiatives consume graph truth (assignments, releases, governance outputs) without altering guardrails.
- POs are first-class: blockers, readiness, scenario toggles, and portfolio views are designed for decision support.
- Each scenario maps to functional requirements, IA structures, and staged delivery (Initiative Stages 1–5, Repository Stage 5), giving engineering clear definition-of-done.
