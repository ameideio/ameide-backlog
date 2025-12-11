# Governance & Compliance Journeys

| Journey ID | Name | Epic | Primary Personas | Goal | Linked Specs |
| --- | --- | --- | --- | --- | --- |
| GJ1 | Configure Governance Stack | E2 Governance Workflows | Governance Lead, Repository Admin | Establish policies, owner groups, checks, routing, and SLA monitors. | FR‑12..16, FR‑20; backlog/220-governance-entity-model.md |
| GJ2 | Orchestrate Review Queue & SLAs | E2 Governance Workflows | Governance Lead, Reviewer | Maintain healthy approval pipeline, manage queue priority, resolve breaches. | FR‑13, FR‑14; backlog/150 A2 |
| GJ3 | Diagnose Required Check Failures | E2 Governance Workflows | Reviewer, Automation Bot, Compliance Officer | Investigate failing checks, rerun with context, document outcomes. | FR‑13; backlog/220-governance-entity-model.md |
| GJ4 | Run Surveys & Recertification | E2 Governance Workflows | Governance Lead, Compliance Officer | Drive data-quality surveys, process responses, and recertify assets. | FR‑20; backlog/150 A5 |
| GJ5 | Handle Notifications & Escalations | E2 Governance Workflows | Governance Lead, Support Analyst | Route alerts, escalate blockers, and coordinate remediation playbooks. | GovernanceRoutingRule, SlaBreachEvent |
| GJ6 (Journey 236) | SLA Escalation & Resilience | E2 Governance Workflows | Governance Lead, Repository Admin, Product Owner | Detect SLA breaches, orchestrate escalations, and restore healthy throughput. | FR‑14, SlaBreachEvent, Journey 244 |
| GJ7 (Journey 238) | Data Quality Survey & Follow-up | E2 Governance Workflows | Governance Lead, Asset Owner | Issue targeted surveys, collect responses, trigger recertification if needed. | FR‑20; DataQualitySurvey, SurveyResponse |
| GJ8 (Journey 239) | Retention Hold & Purge Workflow | E2 Governance Workflows | Compliance Officer, Asset Owner | Apply legal holds, schedule purges, and audit retention events. | AssetRetentionEvent; FR‑20 |
| GJ9 (Journey 240) | Governance Waiver Lifecycle | E2 Governance Workflows | Governance Lead, Reviewer | Approve time-bound waivers, monitor expiry, and close out remediation. | GovernanceRecord (waiver); FR‑13/14 |

---

## GJ1 Configure Governance Stack

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-GJ1.1 As a Governance Lead, I can activate a governance profile with version history.
- US-GJ1.2 As a Governance Lead, I can assign owner groups and map them to graph roles.
- US-GJ1.3 As a Governance Lead, I can configure required checks and binding scopes from a catalog.
- US-GJ1.4 As a Governance Lead, I can author routing rules that direct incidents to the correct escalation channel.

**Steps**
1. Governance Lead selects graph; reviews existing `GovernancePolicy` profile. Chooses version or clones new profile with evidence templates and playbook.
2. Validates `OntologyCatalogue` activation so approved `RepositoryElement` types/relationships are governed and versioned.
3. Defines/updates `GovernanceOwnerGroup` coverage, links to `RepositoryAccessRole`, sets default review SLA overrides.
4. Configures `RequiredCheck` catalog (automated scans, manual sign-offs), maps each check to the relevant `Control` records, and associates via `CheckBinding` to classifications, placement rules, transformations, or RepositoryElements.
5. Sets up `GovernanceRoutingRule` for triggers (classification assignments, element updates, check failures, survey flags) specifying owner groups and fallback chain.
6. Activates governance profile; system records policy version, triggers baseline `RepositoryStatisticSnapshot`, `OutcomeMetricSnapshot` alignment, and publishes governance banner.

**Telemetry & KPIs**: Policy activation events, ontology catalogue status, classification hierarchy health, control coverage %, check catalog completeness, routing rule test results.

---

## GJ2 Orchestrate Review Queue & SLAs

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-GJ2.1 As a Governance Lead, I can monitor queue depth and SLA timers in real time.
- US-GJ2.2 As a Governance Lead, I can reprioritize queue entries and reassign reviewers.
- US-GJ2.3 As a Governance Lead, I receive alerts when SLA thresholds are breached.
- US-GJ2.4 As a Product Owner, I can track queue changes affecting my assets.

**Steps**
1. Governance dashboard shows queue depth, SLA status, priority, and impacted `RepositoryElement` clusters. Lead filters queue by escalation state or element impact.
2. For items nearing breach, lead reorders queue or reassigns approvers via `PromotionQueueEntry` controls; system logs change and updates impacted element overlays.
3. SLA breach occurs → `SlaBreachEvent` created, notifications sent (including adoption/change stakeholders when element risk is high). Lead acknowledges, adds mitigation plan, and tracks resolution.
4. Once approvals/checks complete, queue item processed, lifecycle event recorded, breach resolved (if triggered). Dashboard reflects updated metrics and clears element risk indicators.

**Telemetry & KPIs**: Queue wait P95, number of breaches, time to resolve escalations, reassignment audit count.

---

## GJ3 Diagnose Required Check Failures

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-GJ3.1 As a Reviewer, I can inspect failed check details with evidence and logs.
- US-GJ3.2 As an Automation Bot, I can suggest remediation steps based on failure signatures.
- US-GJ3.3 As a Reviewer, I can rerun checks and capture rationale for each attempt.
- US-GJ3.4 As a Compliance Officer, I can link persistent failures to support tickets.

**Steps**
1. Reviewer receives alert for failed check (`CheckRun.status=failed`). Opens check detail slide-over with evidence, failure details, impacted `RepositoryElement` projections, and sync payload ID.
2. Reviews logs/artifacts, consults automation bot suggestions, evaluates element-level impact, and links corrective tasks or initiates rerun.
3. Adds rationale comment; if external fix required, creates support ticket referencing check run and affected elements.
4. Rerun executed; status updates. On success, timeline notes resolution and clears element risk indicators; on repeated failure, escalates via governance routing.

**Telemetry & KPIs**: Rerun success rate, mean time to resolution, recurring failure patterns, support ticket linkage count.

---

## GJ4 Run Surveys & Recertification

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-GJ4.1 As a Governance Lead, I can launch targeted surveys and track completion.
- US-GJ4.2 As an Asset Owner, I can respond to surveys and flag follow-up actions.
- US-GJ4.3 As a Governance Lead, I can trigger recertification jobs from survey insights.
- US-GJ4.4 As a Product Owner, I can view assets in recertification and understand readiness impact.

**Steps**
1. Governance Lead identifies low data-quality area; launches `DataQualitySurvey` with targeted owner group/classification and associated `RepositoryElement` clusters.
2. Respondents submit `SurveyResponse`; system updates governance metrics, flags assets and elements needing recertification, and records adoption feedback for change teams.
3. Trigger `RecertificationJob` for impacted assets/elements; assets revert to `in_review`, notifications sent, transformations and change managers flagged.
4. Track job progress; upon completion, metrics updated, audit trail stored, `OutcomeMetricSnapshot` recalculated to show benefit/adoption impact.

**Telemetry & KPIs**: Survey completion %, recertification cycle time, residual gaps count, element quality score delta, number of assets returned to green status.

---

## GJ5 Handle Notifications & Escalations

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-GJ5.1 As a Governance Lead, I can triage governance notifications by severity and status.
- US-GJ5.2 As a Governance Lead, I can raise escalations with mitigation owners and due dates.
- US-GJ5.3 As a Support Analyst, I can open support tickets linked to escalations and track resolution.
- US-GJ5.4 As a Governance Lead, I can close notifications automatically once mitigated.

**Steps**
1. `GovernanceNotification` queue lists alerts (review requests, SLA breaches, check failures, adoption drops). Lead filters by status, severity, impacted `RepositoryElement`, or stakeholder segment.
2. For critical issues, escalates via `GovernanceEscalation`, assigning mitigation owner and plan. Change-impact notifications automatically invite change managers when adoption signals fall.
3. Support analyst may open `GovernanceSupportTicket` when integrations fail (e.g., sync job errors) or when RepositoryElement inconsistencies require tooling support, linking to escalation.
4. Once mitigated, notifications acknowledged, escalation closed, support ticket resolved with knowledge entry, OutcomeMetricSnapshot updated if value impact occurred.

**Telemetry & KPIs**: Notification response time, escalation resolution time, support ticket recurrence, acknowledgement rate, adoption-signal response time.

---

## GJ6 (Journey 236) SLA Escalation & Resilience

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-GJ6.1 As a Monitoring Bot, I can emit SLA breach alerts with context (queue item, owner group, elapsed time).
- US-GJ6.2 As a Governance Lead, I can initiate recovery playbooks for stuck approvals/checks.
- US-GJ6.3 As a Repository Admin, I can enable fallback approvers or temporary access roles to clear bottlenecks.
- US-GJ6.4 As a Product Owner, I receive status updates on escalated items affecting my scope.
- US-GJ6.5 As a Compliance Officer, I can review post-mortems for SLA breaches.

**Steps**
1. Monitoring service detects SLA breach and creates `SlaBreachEvent` (Journey 244 details failure path). Notification includes context (review case, owner group, impacted `RepositoryElement`, elapsed time).
2. Governance Lead assesses breach, triggers escalation playbook (e.g., reassign approver, add temporary reviewer via `RepositoryAccessRole` grant) and coordinates with change managers if adoption signals are declining for the same elements.
3. Repository Admin validates new assignments, ensures segregation-of-duties still satisfied, and logs manual overrides, including temporary RepositoryElement access if needed.
4. Approvers act, queue item completes; breach resolved. Governance Lead records mitigation summary, updates OutcomeMetricSnapshot if benefits impacted, and attaches to escalation record.
5. Compliance Officer reviews incident, ensures lessons captured and optionally updates routing rules or SLA targets, plus ontology governance rules when element patterns contributed to the breach.

**Success Signals**
- Breach mean time to recovery under defined threshold.
- Escalation records fully documented (owner, steps, resolution).
- POs confirm visibility; readiness dashboards updated accordingly.

**Telemetry & KPIs**: Breach frequency, mean time to recovery, number of manual overrides, recurrence of same root cause.

---

## GJ7 (Journey 238) Data Quality Survey & Follow-up

**Overview**: Governance lead issues targeted `DataQualitySurvey` to close metadata gaps, captures `SurveyResponse`, and drives remediation/recertification if owners fail to respond.

**Key Steps**
1. Launch survey against a classification segment with due dates and escalation plan.
2. Owners submit responses; automation reruns metadata checks and refreshes dashboards.
3. Reminders/escalations run for non-responders; PromotionQueueEntry + SLA states track overdue items.
4. Survey closes; telemetry updates data quality KPIs and triggers `RecertificationJob` for non-compliant assets.

**Telemetry & KPIs**: Completion rate, response SLA, data quality uplift, recertification count.

---

## GJ8 (Journey 239) Retention Hold & Purge Workflow

**Overview**: Compliance officer applies legal holds, prevents purge while active, releases hold when case resolved, and confirms purge execution with audit evidence.

**Key Steps**
1. Apply `AssetRetentionEvent` (`hold_applied`) with legal reference; notify stakeholders.
2. Scheduled purge job respects hold state and logs skipped purge.
3. Release hold, schedule purge, and communicate timeline.
4. Purge executes, creating `purge_executed` event and updating telemetry.

**Telemetry & KPIs**: Hold counts vs purges, hold duration, purge success, skipped purge metrics.

---

## GJ9 (Journey 240) Governance Waiver Lifecycle

**Overview**: Governance lead and compliance officer approve time-bound waivers, monitor expiry, and retire the waiver once mitigation completes.

**Key Steps**
1. Asset owner submits waiver request; draft `GovernanceRecord` created.
2. Approvers review conditions; waiver approved, checks marked `waived` with expiration.
3. Reminder job escalates as expiry nears; queue SLA states track risk.
4. Mitigation completes prior to expiry; waiver retired, checks rerun.

**Telemetry & KPIs**: Active vs expired waivers, waiver MTTR, waived check coverage, escalation frequency.
