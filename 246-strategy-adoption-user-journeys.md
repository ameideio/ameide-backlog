# Strategy, Value & Adoption Journeys

| Journey ID | Name | Epic | Primary Personas | Goal | Linked Specs |
| --- | --- | --- | --- | --- | --- |
| SJ1 | Align Digital Vision & Investment | E5 Strategy & Change Enablement | Sponsor, Enterprise Architect, Strategy Lead | Shape the digital vision, prioritise bets, and secure investment with governance alignment. | backlog/160-transformation-transformations-business-scenario.md; backlog/113-1-architecture.md |
| SJ2 | Drive Change Management & Enablement | E5 Strategy & Change Enablement | Change Manager, Product Owner, Communications Lead, Training Lead | Prepare stakeholders for change, deliver communications, and equip roles with guidance linked to graph assets. | backlog/113-8-appendices.md; backlog/160-transformation-transformations-implementation-plan.md |
| SJ3 | Realise & Measure Business Outcomes | E5 Strategy & Change Enablement | Sponsor, PMO Analyst, Product Owner, Finance Partner | Track planned benefits, capture adoption signals, and reconcile outcome KPIs with transformation/graph telemetry. | backlog/160-transformation-transformations-business-scenario.md; backlog/235-telemetry-user-journeys.md |

---

## SJ1 Align Digital Vision & Investment

**Epic**: E5 Strategy & Change Enablement  
**User Stories**:
- US-SJ1.1 As a Sponsor, I can articulate the digital vision, target operating model, and success metrics.
- US-SJ1.2 As a Strategy Lead, I can map proposed transformations to graph capabilities and governance requirements.
- US-SJ1.3 As an Enterprise Architect, I can validate that strategic bets align with current and target architecture states.
- US-SJ1.4 As a Sponsor, I can approve an integrated roadmap with funding stages and guardrails.

**Personas**: Sponsor (SP), Strategy Lead (SL), Enterprise Architect (EA)  
**Preconditions**: Enterprise strategy cycle underway; graph baseline and transformation templates available.  
**Postconditions**: Approved transformation transformation backlog with prioritised scope, funding envelopes, and governance guardrails.

1. Sponsor and Strategy Lead run a vision workshop referencing graph baselines, business objectives, market drivers, and `ElementProjection` views (capability/value-stream maps). Outcomes stored as `StrategicTheme` records referencing assets/transformations/elements.
2. Strategy Lead drafts transformation candidates in portfolio workspace, linking to graph capabilities, RepositoryElements, and governance expectations (required checks, policy coverage).
3. Enterprise Architect evaluates feasibility: compares current vs target architectures, highlights dependencies using RepositoryElement relationships, and records recommended sequencing.
4. Sponsorship board reviews business case, approves roadmap gated by governance metrics (coverage thresholds, SLA expectations), funding guardrails, and benefit hypotheses; telemetry join inputs defined for future Outcome tracking.

**Success Signals**
- Vision, target model, and benefit hypotheses captured in portfolio workspace with referenced RepositoryElements.
- Initiatives linked to governance guardrails before approval (no orphaned bets).
- Funding stage gates tied to measurable governance/telemetry checkpoints and configured OutcomeMetricSnapshots.

**Telemetry & KPIs**: Percent of transformations with traceable benefit hypotheses; alignment score between strategy themes and graph coverage; decision cycle time for roadmap approval.

---

## SJ2 Drive Change Management & Enablement

**Epic**: E5 Strategy & Change Enablement  
**User Stories**:
- US-SJ2.1 As a Change Manager, I can segment stakeholders and map them to affected assets/transformations.
- US-SJ2.2 As a Communications Lead, I can draft campaigns referencing source-of-truth assets and publish schedules.
- US-SJ2.3 As a Training Lead, I can assemble enablement packages linked to graph content and ADR decisions.
- US-SJ2.4 As a Product Owner, I can monitor adoption health signals and capture feedback for backlog intake.

**Personas**: Change Manager (CM), Communications Lead (CL), Training Lead (TL), Product Owner (PO)  
**Preconditions**: Strategic roadmap approved; impacted personas and transformations identified.  
**Postconditions**: Stakeholder change plans active, enablement materials delivered, adoption feedback loops running.

1. Change Manager segments stakeholders, maps them to affected assets/transformations and `RepositoryElement` clusters using classifications and ElementProjections; generates `ChangeImpactProfile`.
2. Communications Lead drafts campaign plan with cadence, channels, and key messages referencing canonical assets (e.g., standards, ADRs) and elements impacted. Plan stored as `CommunicationPlan` linked to transformations.
3. Training Lead assembles enablement playlist (guides, recordings, office hours) pulling from graph releases, knowledge articles, and element-specific guidance; publishes via change workspace.
4. Product Owner monitors adoption dashboards (feature usage, self-service readiness, sentiment) and RepositoryElement adoption signals, logs feedback as backlog items or governance follow-ups.
5. Change tasks automatically reference affected elements/themes so telemetry can attribute adoption shifts to architecture changes.

**Success Signals**
- Every stakeholder segment has an active communication and training plan tied to impacted elements.
- Enablement artifacts reference approved graph assets/ADRs and RepositoryElements (single source of truth).
- Adoption feedback loops yield actionable backlog items within SLA; adoption signals improving for targeted elements.

**Telemetry & KPIs**: Communication reach/open rates; training completion; adoption health index; number of feedback items triaged into backlog.

---

## SJ3 Realise & Measure Business Outcomes

**Epic**: E5 Strategy & Change Enablement  
**User Stories**:
- US-SJ3.1 As a Sponsor, I can define expected outcomes, KPIs, and benefit milestones per transformation.
- US-SJ3.2 As a PMO Analyst, I can reconcile financial and operational KPIs with graph and transformation telemetry.
- US-SJ3.3 As a Product Owner, I can trace outcome gaps back to blockers, governance signals, or adoption issues.
- US-SJ3.4 As a Finance Partner, I can validate benefit realisation and update forecasts based on live telemetry.

**Personas**: Sponsor, PMO Analyst (PA), Product Owner, Finance Partner (FP)  
**Preconditions**: Initiatives executing with telemetry instrumentation; benefits framework defined.  
**Postconditions**: Benefits tracked against plan, gaps flagged with remediation actions, portfolio decisions updated.

1. Sponsor defines benefit framework (financial, customer, operational) in portfolio workspace; links KPIs to transformation milestones, RepositoryElements, and governance guardrails.
2. PMO Analyst configures telemetry join pipeline combining `InitiativeMetricSnapshot`, `RepositoryStatisticSnapshot`, `GovernanceMetricSnapshot`, `OutcomeMetricSnapshot`, adoption feeds, and element coverage metrics; dashboards highlight deltas vs targets.
3. Product Owner reviews outcome dashboards, correlates gaps with blockers, governance anomalies, adoption feedback, or element readiness; raises mitigation stories or governance escalations.
4. Finance Partner updates benefit forecast and confirms realised value; decisions documented for steering reviews and quarterly architecture reviews (Journey 242), including RepositoryElement impact notes.

**Success Signals**
- Outcome dashboards refreshed with near-real-time operational + business metrics plus element/adoption views.
- Identified gaps have assigned mitigation owners and timelines tied to specific elements/themes.
- Steering reviews consume reconciled value reports before approving next investment stage; telemetry join outputs audited.

**Telemetry & KPIs**: Benefit realisation %, variance to plan, outcome gap resolution cycle time, correlation between adoption signals and KPI improvements.

---
