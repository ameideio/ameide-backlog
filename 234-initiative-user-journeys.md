# Transformation Initiative Journeys

| Journey ID | Name | Epic | Primary Personas | Goal | Linked Specs |
| --- | --- | --- | --- | --- | --- |
| IJ1 | Launch Initiative with Governance Alignment | E3 Initiative Management | Program Manager, Product Owner, Lead Architect | Create transformation, staff roles, link graph governance, and initialize milestones. | backlog/160 B1; FR‑19 |
| IJ2 | Scope Work from Repository | E3 Initiative Management | Product Owner, Lead Architect | Map existing assets to milestones, capture new drafts, and expose scope delta. | backlog/160 B2; FR‑6, FR‑18 |
| IJ3 | Toggle As-Is / To-Be Scenarios | E4 Landscape & Reporting | Product Owner, Sponsor | Visualize scenario impact and timelines during steering reviews. | backlog/160 B3; FR‑18 |
| IJ4 | Operate Blockers Funnel | E2 Governance Workflows | Product Owner, Governance Lead | Triage approvals, checks, queue items, and move work forward. | backlog/160 B4; FR‑13, FR‑14 |
| IJ5 | Deliver Release Readiness / Go-No-Go | E3 Initiative Management | Product Owner, Sponsor | Compile evidence-based readiness, share release exports, support decision. | backlog/160 B5; FR‑15 |
| IJ6 | Manage Portfolio Heatmap (Future) | E3 Initiative Management | PMO, Sponsor | Aggregate transformation health, blockers, and readiness across portfolio. | backlog/160 B7 |

---

## IJ1 Launch Initiative with Governance Alignment

**Epic**: E3 Initiative Management  
**User Stories**:
- US-IJ1.1 As a Program Manager, I can create an transformation from a template with governance defaults.
- US-IJ1.2 As a Program Manager, I can assign transformation roles and track acceptance.
- US-IJ1.3 As a Product Owner, I can view governance banner and readiness metrics upon launch.
- US-IJ1.4 As a Lead Architect, I can confirm graph linkages and milestone scaffolding.

1. Program Manager selects transformation template (ADM-aligned). Enters name, description, target dates.
2. Links enterprise graph; system imports governance defaults (roles, SLA, required checks) and active `ChangeManagementProfile` / `ChangePlaybook` guidance.
3. Aligns transformation with `StrategicTheme`/`BenefitHypothesis`, selects impacted `RepositoryElement` projections (capabilities, value streams) for scope context.
4. Assigns roles (Sponsor, PO, Lead Architect, Reviewers, Contributors). Directory sync invites members; acceptance updates `InitiativeRoleAssignment`.
5. Initiative milestones seeded; PO reviews default boards, readiness metrics, and initial adoption telemetry placeholders.

**Success Signals**: Role coverage ≥ 80%; graph governance banner visible in workspace; strategic theme + change profile set; telemetry logs provisioning event.

---

## IJ2 Scope Work from Repository

**Epic**: E3 Initiative Management  
**User Stories**:
- US-IJ2.1 As a Lead Architect, I can import graph assets into transformation milestones.
- US-IJ2.2 As a Product Owner, I can distinguish baseline assets from new drafts.
- US-IJ2.3 As a Product Owner, I can capture new work items directly in transformation workspace.
- US-IJ2.4 As a Product Owner, I can monitor scope coverage and risk metrics.

1. Lead Architect navigates graph view inside transformation; links baseline assets and associated `RepositoryElement` projections to milestones (drag from graph tree or capability map).
2. Creates drafts for new work items; lifecycle state remains `draft`; new ElementAttachments recorded for planned capability/process changes.
3. Initiative board highlights existing vs new scope; PO reviews coverage, element impact, and risk.
4. Metrics update (linked scope %, element coverage %, new drafts count).

**Success Signals**: Scope coverage ratio and RepositoryElement coverage meet target; graph assignments established; transformation board reflects classification tags.

---

## IJ3 Toggle As-Is / To-Be Scenarios

**Epic**: E4 Landscape & Reporting  
**User Stories**:
- US-IJ3.1 As a Product Owner, I can toggle between As-Is, To-Be, and scenario views.
- US-IJ3.2 As a Sponsor, I can see visual overlays summarizing impact per scenario.
- US-IJ3.3 As a Product Owner, I can export scenario delta reports for stakeholders.
- US-IJ3.4 As a Platform Analyst, I can ensure scenario results cache appropriately.

1. During steering review, PO opens scenario toggle (As-Is, To-Be, Scenario N).
2. Workspace queries graph assignments and impacted `RepositoryElement` projections for effective windows, scenario tags; caches results.
3. Visual overlays (systems map, roadmap, capability map) update; difference report highlights impacted capabilities and adoption status.
4. Sponsor reviews change delta, checks OutcomeMetricSnapshot variance, decides adjustments.

**Success Signals**: Toggle latency <300 ms cached / <2 s P95 live; impact counts accurate; OutcomeMetricSnapshot updated; decision recorded.

---

## IJ4 Operate Blockers Funnel

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-IJ4.1 As a Product Owner, I can filter blockers by approvals, checks, and queue state.
- US-IJ4.2 As a Governance Lead, I can reprioritize queue items directly from transformation view.
- US-IJ4.3 As a Reviewer, I receive prompts to act on blockers assigned to me.
- US-IJ4.4 As a Product Owner, I can log mitigation actions on each blocker.

1. PO opens blockers funnel view with filters “Needs approvals”, “Failing checks”, “In queue”, and RepositoryElement impact severity.
2. Repository events update chip counts and statuses via SSE (<30 s); element overlays highlight affected capabilities. PO assigns owners or coordinates with Governance Lead/change manager to reorder queue.
3. Reruns checks or nudges approvers/change tasks; statuses transition to resolved and adoption signals monitored.

**Success Signals**: Outstanding blockers reduced; queue wait P95 improving; SLA breaches resolved promptly; RepositoryElement risk indicators cleared.

---

## IJ5 Deliver Release Readiness / Go-No-Go

**Epic**: E3 Initiative Management  
**User Stories**:
- US-IJ5.1 As a Product Owner, I can generate readiness reports summarizing scope, checks, approvals, and releases.
- US-IJ5.2 As a Sponsor, I can access Go/No-Go decision summaries with evidence.
- US-IJ5.3 As a Product Owner, I can log decision outcomes and create mitigation tasks.
- US-IJ5.4 As a Governance Lead, I can verify readiness artifacts meet compliance requirements.

1. Ahead of milestone, PO runs readiness report: pulls approved & released versions, check status, queue history, release notes, outstanding risks, and impacted `RepositoryElement` coverage/adoption signals.
2. Shares export (PDF/CSV) with sponsor; runs Go/No-Go meeting.
3. Decision logged; if Go, release execution tracked; if No-Go, mitigation tasks created.
4. Sponsor and Finance partner reconcile outcome expectations with SJ3/TJ4 dashboards and confirm RepositoryElement readiness before final approval.

**Success Signals**: Report generated in <10 s; completeness score high; decision recorded with evidence; RepositoryElement readiness/adoption deltas understood and assigned.

---

## IJ6 Manage Portfolio Heatmap (Future)

**Epic**: E3 Initiative Management  
**User Stories**:
- US-IJ6.1 As a PMO Analyst, I can view transformation KPIs aggregated across portfolio.
- US-IJ6.2 As a Sponsor, I can filter portfolio by risk tier or segment.
- US-IJ6.3 As a PMO Analyst, I can drill into specific transformations to inspect blockers.
- US-IJ6.4 As a Product Owner, I can respond to portfolio-driven requests for updates.

1. PMO accesses portfolio dashboard aggregating transformation KPIs (health, blockers, readiness) plus Strategy & Change metrics (benefit variance, adoption score) and RepositoryElement coverage.
2. Filters by graph, segment, risk tier, strategic theme, or element cluster.
3. Drill-down to specific transformation blockers, scenario deltas, or element/adoption hot spots; launches change or governance playbooks as needed.

**Success Signals**: Portfolio view used for quarterly planning; heatmap aligns with governance, RepositoryElement coverage, and outcome telemetry metrics.
