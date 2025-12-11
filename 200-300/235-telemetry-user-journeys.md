# Analytics & Telemetry Journeys

| Journey ID | Name | Epic | Primary Personas | Goal | Linked Specs |
| --- | --- | --- | --- | --- | --- |
| TJ1 | Monitor Repository Sync Health | E4 Landscape & Reporting | Platform Analyst, Reliability Engineer | Track `RepositorySyncJob` executions, ensure downstream systems stay up-to-date. | backlog/220-graph-entity-model.md (RepositorySyncJob, IntegrationEndpoint) |
| TJ2 | Investigate Governance Anomalies | E2 Governance Workflows | Platform Analyst, Governance Lead | Diagnose spikes in queue latency, check failures, or SLA breaches surfaced in snapshots. | RepositoryStatisticSnapshot, GovernanceMetricSnapshot |
| TJ3 | Reconcile Initiative & Repository KPIs | E4 Landscape & Reporting | Product Owner, PMO Analyst | Produce unified report of readiness, blockers, releases, and governance metrics. | InitiativeMetricSnapshot, Governance dashboards |
| TJ4 | Track Business Outcome & Adoption KPIs | E5 Strategy & Change Enablement | PMO Analyst, Sponsor, Finance Partner | Correlate business outcomes, financial benefits, and adoption health with transformation and graph telemetry. | backlog/246-strategy-adoption-user-journeys.md (SJ3); portfolio KPI feeds |

---

## TJ1 Monitor Repository Sync Health

**Epic**: E4 Landscape & Reporting  
**User Stories**:
- US-TJ1.1 As a Platform Analyst, I can view recent sync jobs with status, duration, and retry count.
- US-TJ1.2 As a Platform Analyst, I can filter sync jobs by integration type (graph, search, analytics).
- US-TJ1.3 As a Reliability Engineer, I can rerun failed sync jobs and capture remediation notes.
- US-TJ1.4 As a Platform Analyst, I can verify downstream acknowledgements post-sync.

1. Analyst opens Sync Operations dashboard listing recent `RepositorySyncJob`s (status, duration, retry count) with RepositoryElement delta counts.
2. Filters by integration type (knowledge graph, search, analytics, element_projections). Checks error logs for failed jobs.
3. For failures, reviews `error_state`, impacted elements/projections, triggers rerun, or opens support ticket (link to GovernanceSupportTicket).
4. Confirms downstream acknowledgement (knowledge graph ingestion timestamp, element projection refresh, search index version, analytics dataset update).
5. Spot-checks newly ingested `ProcessMiningDataset` records for critical applications to ensure telemetry aligns with RepositoryElement references and anomaly flags.

**Success Signals**: Retry success rate high; no jobs in failed > 1 hr; downstream timestamps, RepositoryElement coverage, and telemetry datasets (e.g., process mining) align.

---

## TJ2 Investigate Governance Anomalies

**Epic**: E2 Governance Workflows  
**User Stories**:
- US-TJ2.1 As a Platform Analyst, I can detect anomalies in graph or governance snapshots.
- US-TJ2.2 As a Platform Analyst, I can drill into metrics to identify impacted owner groups/checks.
- US-TJ2.3 As a Governance Lead, I can coordinate remediation and log mitigation steps.
- US-TJ2.4 As a Platform Analyst, I can confirm anomaly resolution in subsequent snapshots.

1. Dashboard flag (anomaly) indicates queue wait P95 spike or RepositoryElement quality regression. Analyst inspects `RepositoryStatisticSnapshot` details.
2. Cross-links to `GovernanceMetricSnapshot` and ElementProjections to determine affected owner groups, check categories, and capability clusters.
3. Drills into underlying `PromotionQueueEntry`, `CheckRun`, and impacted `RepositoryElement` attachments; identifies root cause (e.g., missing reviewer, outdated capability model).
4. Coordinates with Governance Lead/change manager to adjust routing, update change plans, or add resources; monitors subsequent snapshot and Outcome metrics for improvements.

**Success Signals**: Anomaly resolved within target window; follow-up notes logged; metrics and RepositoryElement quality scores return to baseline.

---

## TJ3 Reconcile Initiative & Repository KPIs

**Epic**: E4 Landscape & Reporting  
**User Stories**:
- US-TJ3.1 As a PMO Analyst, I can export combined graph/transformation snapshots.
- US-TJ3.2 As a Product Owner, I can review unified KPI reports and drill into blockers.
- US-TJ3.3 As a PMO Analyst, I can schedule automated KPI reconciliations.
- US-TJ3.4 As a Governance Lead, I can correlate KPI gaps with governance actions.

1. PMO analyst exports latest `InitiativeMetricSnapshot` joined with `RepositoryStatisticSnapshot`, `GovernanceMetricSnapshot`, `ArchitectureStateMaterialized`, and element coverage metrics via shared keys.
2. Builds report showing blockers, approvals, releases, data quality, RepositoryElement readiness/adoption, and SLA status per transformation.
3. Shares with Product Owners; highlights areas needing governance or change intervention, including specific capability/element hot spots.
4. Logs report metadata for future trend analysis and updates `OutcomeMetricSnapshot` references for audit.

**Success Signals**: Report generation automated; POs confirm visibility; RepositoryElement and ArchitectureState gaps captured; subsequent planning actions tracked.

---

## TJ4 Track Business Outcome & Adoption KPIs

**Epic**: E5 Strategy & Change Enablement  
**User Stories**:
- US-TJ4.1 As a PMO Analyst, I can ingest business outcome feeds (financial, customer, operational) and join them with transformation and graph telemetry.
- US-TJ4.2 As a Sponsor, I can view benefit realisation dashboards tied to SJ3 outcome milestones.
- US-TJ4.3 As a Finance Partner, I can reconcile forecasts against realised benefits and annotate variances.
- US-TJ4.4 As a Change Manager, I can correlate adoption health indicators with outcome trends and trigger remediation plays.

1. PMO analyst configures KPI ingestion jobs pulling from finance, customer experience, and operations systems; joins with `InitiativeMetricSnapshot`, `RepositoryStatisticSnapshot`, `OutcomeMetricSnapshot`, and relevant RepositoryElement projections (capability tags, value streams).
2. Dashboards visualise benefit forecast vs actuals, adoption scores, governance health, and RepositoryElement readiness; sponsor reviews outcome deltas during steering forums and quarterly architecture reviews (Journey 242).
3. Finance partner annotates variances, captures mitigation tasks or funding adjustments, and exports evidence for audit, including affected elements/themes.
4. Change manager cross-references adoption gaps with communication/training plans (SJ2) and impacted elements, creating follow-up backlog or governance actions.

**Success Signals**: Outcome dashboards refreshed within agreed cadence; variances assigned owners; benefit realisation decisions traceable to telemetry.

**Telemetry & KPIs**: Benefit realisation %, forecast variance, adoption health index, time-to-mitigation for negative outcome trends.

---

## Dashboards & Test Coverage (Platform Expectations)

| Area | Dashboard Expectations | Test/Alert Coverage |
| --- | --- | --- |
| TracePolicy Compliance | Visualise % of requirements with `satisfied_by` links, highlight violations by classification/transformation. | Automated RequiredCheck ensuring each TracePolicy is met (blocking), synthetic data test verifying violation surfaces on dashboard. |
| Control Coverage | Show controls mapped to RequiredChecks with pass/fail trend, recertification countdown. | Unit test verifying `ControlMapping` integrity; telemetry assertion that dashboards flag controls without active evidence. |
| ArchitectureStateMaterialized | Trend ArchitectureState snapshots (hash diff, refresh reason) and capability deltas. | Scheduled job test confirming materialized snapshot exists per scope; alert when refresh lag exceeds threshold. |
| Process Mining Telemetry | Surface ProcessMiningDataset freshness, anomalies, and linked RepositoryElements. | Ingestion test that seeds sample dataset and verifies KPI on dashboard; alert if dataset missing beyond SLA. |
| Outcome Join Health | Merge Repository, Governance, Initiative, Outcome snapshots to display variance + adoption signals. | Integration test validating TelemetryJoinView contains expected keys and dashboards render; alert when join lag or missing data detected. |

> Implementers should automate these tests within the telemetry pipeline (CI) and add synthetic monitors for production dashboards so regressions in data availability or policy compliance are caught early.
