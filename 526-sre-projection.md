# 526 SRE — Projection Primitive Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **sre-projection** primitive — read models for operations dashboards, incident history, and fleet health visualization.

---

## 1) Projection responsibilities

The `sre-projection` primitive builds **read-optimized views**:

- **Fleet health dashboards** — real-time application and cluster status
- **Incident timeline views** — historical incident data and patterns
- **SLO burn-down displays** — error budget tracking and trends
- **Alert correlation views** — grouped alerts and potential incidents
- **Runbook catalog** — searchable operational procedures

Projection primitives:

- Are **idempotent consumers** of domain and process facts
- Build **materialized views** optimized for read patterns
- Use **inbox or natural-key UPSERT** for exactly-once semantics
- Never perform writes to domain state

---

## 2) Projection catalog

### 2.1 FleetHealthProjection

**Purpose:** Real-time fleet status for operations dashboard.

**Consumed facts:** `FleetStateRecorded`, `HealthCheckRecorded`, `AlertIngested`

**View:** Aggregate fleet health with per-application breakdown, alert counts, open incidents.

### 2.2 ApplicationHealthProjection

**Purpose:** Per-application health status with drill-down.

**Consumed facts:** `HealthCheckRecorded`, `FleetStateRecorded`

**View:** Per-application sync status, health status, conditions.

### 2.3 IncidentTimelineProjection

**Purpose:** Historical incident data with MTTR metrics.

**Consumed facts:** `IncidentCreated`, `IncidentResolved`, `IncidentClosed`, `IncidentTriageCompleted`

**Views:**
- Incident summary with calculated metrics
- MTTR aggregations by environment, severity

### 2.4 SLOBurndownProjection

**Purpose:** SLO error budget tracking.

**Consumed facts:** `SLOCreated`, `SLIMeasurementRecorded`, `SLOBudgetExhausted`

**Views:**
- Current SLO status with budget remaining
- SLI time series for charts

### 2.5 AlertCorrelationProjection

**Purpose:** Grouped alerts for incident identification.

**Consumed facts:** `AlertIngested`, `AlertAcknowledged`, `AlertsCorrelatedToIncident`

**Views:**
- Alert groups by correlation key
- Active alerts with search

### 2.6 RunbookCatalogProjection

**Purpose:** Searchable runbook catalog.

**Consumed facts:** `RunbookCreated`, `RunbookUpdated`, `RunbookExecutionCompleted`

**View:** Runbook catalog with usage stats and full-text search.

---

## 3) Query APIs

- **FleetHealthQueryService** — GetFleetHealth, ListEnvironmentHealth
- **ApplicationHealthQueryService** — GetApplicationHealth, ListApplications
- **IncidentQueryService** — ListIncidents, SearchIncidents, GetIncidentMetrics
- **SLOQueryService** — ListSLOs, GetSLOStatus, GetSLOBurndown
- **AlertQueryService** — ListActiveAlerts, ListAlertGroups
- **RunbookQueryService** — ListRunbooks, SearchRunbooks

---

## 4) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind projection --name sre --include-gitops
```

### Topic subscriptions

- `sre.domain.facts.v1` → All projections
- `sre.process.facts.v1` → IncidentTimelineProjection

### Inbox pattern

All projections use inbox for exactly-once processing.

---

## 5) Acceptance criteria

1. All projections consume facts idempotently
2. Fleet health updates within seconds of fact publication
3. Full-text search available for incidents, alerts, runbooks
4. MTTR metrics aggregate correctly
5. Query APIs support pagination and filtering
