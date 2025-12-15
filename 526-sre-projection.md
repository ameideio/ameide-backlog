# 526 SRE — Projection Primitive Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **sre-projection** primitive — read models for operations dashboards, incident history, fleet health visualization, and semantic search.

---

## 1) Projection responsibilities

The `sre-projection` primitive builds **read-optimized views**:

- **Fleet health dashboards** — real-time application and cluster status
- **Incident timeline views** — historical incident data, patterns, and MTTR metrics
- **SLO burn-down displays** — error budget tracking and SLI time-series
- **Alert correlation views** — grouped alerts and potential incidents
- **Runbook catalog** — searchable operational procedures
- **Knowledge index** — vector-backed semantic search for pattern matching (backlog-first triage)

Projection primitives:

- Are **idempotent consumers** of domain and process facts
- Build **materialized views** optimized for read patterns
- Use **inbox or natural-key UPSERT** for exactly-once semantics
- **Never perform writes to domain state**
- Provide **all search and time-series APIs** (domain only provides minimal get-by-id)

**Key architectural role:**
> UISurfaces and Agents query projections, NOT domain internals. All search, dashboards, history, and time-series go through projection query services.

---

## 2) Projection catalog

### 2.1 FleetHealthProjection

**Purpose:** Real-time fleet status for operations dashboard.

**Consumed facts:** `FleetStateRecorded`, `HealthCheckRecorded`, `AlertIngested`

**View:** Aggregate fleet health with per-application breakdown, alert counts, open incidents.

**Query operations:**
- `GetFleetHealth(environment)` — aggregate health summary
- `ListEnvironmentHealth()` — health across all environments
- `GetApplicationHealth(app_id)` — per-application detail

### 2.2 IncidentTimelineProjection

**Purpose:** Historical incident data with search and MTTR metrics.

**Consumed facts:** `IncidentCreated`, `IncidentSeverityChanged`, `IncidentResolved`, `IncidentClosed`, `IncidentTriageCompleted`

**Views:**
- Incident summary with calculated metrics (time-to-detect, time-to-resolve)
- MTTR aggregations by environment, severity, time window
- Full incident timeline with all events

**Query operations:**
- `SearchIncidents(query, filters)` — full-text search
- `GetIncidentTimeline(incident_id)` — detailed event history
- `GetMTTRMetrics(environment, window)` — aggregated reliability metrics

### 2.3 SLOBurndownProjection

**Purpose:** SLO error budget tracking and SLI time-series.

**Consumed facts:** `SLOCreated`, `SLIWindowRecorded`, `SLOBudgetExhausted`

**Views:**
- Current SLO status with budget remaining
- SLI time series for charts (aggregated by minute/hour/day)
- Burn rate trends

**Query operations:**
- `GetSLOBurndown(slo_id)` — current status + budget
- `GetSLITimeSeries(slo_id, start, end, granularity)` — time-series data

**Note:** Projection stores SLI time-series data; domain only stores current budget state.

### 2.4 AlertCorrelationProjection

**Purpose:** Grouped alerts for incident identification.

**Consumed facts:** `AlertIngested`, `AlertAcknowledged`, `AlertsCorrelatedToIncident`

**Views:**
- Alert groups by correlation key (fingerprint, labels)
- Active alerts with search
- Alert-to-incident correlation

**Query operations:**
- `ListActiveAlerts(filters)` — currently firing alerts
- `ListAlertGroups()` — alerts grouped by correlation key
- `SearchAlerts(query)` — full-text search

### 2.5 RunbookCatalogProjection

**Purpose:** Searchable runbook catalog with usage statistics.

**Consumed facts:** `RunbookCreated`, `RunbookUpdated`, `RunbookExecutionCompleted`

**View:** Runbook catalog with tags, usage stats, and full-text search.

**Query operations:**
- `SearchRunbooks(query)` — full-text search
- `ListRunbooksByTag(tags)` — filtered by tags
- `GetRunbookUsageStats(runbook_id)` — execution history

### 2.6 KnowledgeIndexProjection (Vector-backed semantic search)

**Purpose:** Semantic search for the 525 backlog-first triage workflow. Provides pattern matching without runtime Transformation domain coupling.

**Consumed facts:**
- SRE domain: `IncidentCreated`, `IncidentResolved`, `RunbookCreated`, `RunbookUpdated`
- Transformation domain (via CDC/facts): Backlog item changes (title, description, symptoms, tags)

**Technology:** PostgreSQL + pgvector (CNPG)

**Views:**
- Incident pattern embeddings (title + description + symptoms + resolution)
- Runbook content embeddings (title + steps + trigger conditions)
- Backlog pattern embeddings (consumed from Transformation facts)

**Query operations:**
- `SearchPatterns(symptoms, tags)` — scored matches with backlog item IDs
- `GetSimilarIncidents(incident_description)` — vector similarity search
- `SearchRunbooksBySemantic(query)` — semantic search over runbooks

**Critical architectural role:**
> This projection replaces direct Transformation domain queries for "backlog lookup" in the 525 workflow. SRE agents/processes call KnowledgeIndexQueryService, never Transformation domain directly.

**Embedding generation:**
- Projections compute embeddings (not domain)
- Embedding model configurable (OpenAI, local model)
- Embeddings recomputed on content change

---

## 3) Query APIs summary

| Service | Projection | Operations |
|---------|------------|------------|
| `FleetHealthQueryService` | FleetHealthProjection | GetFleetHealth, ListEnvironmentHealth, GetApplicationHealth |
| `IncidentQueryService` | IncidentTimelineProjection | SearchIncidents, GetIncidentTimeline, GetMTTRMetrics |
| `SLOQueryService` | SLOBurndownProjection | GetSLOBurndown, GetSLITimeSeries |
| `AlertQueryService` | AlertCorrelationProjection | ListActiveAlerts, ListAlertGroups, SearchAlerts |
| `RunbookQueryService` | RunbookCatalogProjection | SearchRunbooks, ListRunbooksByTag, GetRunbookUsageStats |
| `KnowledgeIndexQueryService` | KnowledgeIndexProjection | SearchPatterns, GetSimilarIncidents, SearchRunbooksBySemantic |

---

## 4) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind projection --name sre --include-gitops
```

### Topic subscriptions

- `sre.domain.facts.v1` → All projections
- `sre.process.facts.v1` → IncidentTimelineProjection
- `transformation.domain.facts.v1` → KnowledgeIndexProjection (for backlog patterns)

### Inbox pattern

All projections use inbox for exactly-once processing.

### Database technology

- **Standard projections:** PostgreSQL (CNPG)
- **KnowledgeIndexProjection:** PostgreSQL + pgvector extension
- **SLI time-series:** PostgreSQL with TimescaleDB extension (optional) or partitioned tables

### Embedding configuration

```yaml
knowledge_index:
  embedding_model:
    provider: openai  # or "local"
    model: text-embedding-3-small
    dimensions: 1536
  reindex_on_startup: false
  batch_size: 100
```

---

## 5) Acceptance criteria

1. All projections consume facts idempotently
2. Fleet health updates within seconds of fact publication
3. **Full-text search available** for incidents, alerts, runbooks
4. **Semantic search available** via KnowledgeIndexProjection
5. MTTR metrics aggregate correctly
6. SLI time-series supports multiple granularities
7. Query APIs support pagination and filtering
8. **SRE agents query KnowledgeIndexQueryService** (not Transformation domain)
9. Vector embeddings computed by projection (not domain)
