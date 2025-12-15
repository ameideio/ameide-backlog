# 526 SRE — Domain Primitive Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **sre-domain** primitive — the system-of-record for incidents, alerts, runbooks, SLOs, and health assessments.

---

## 1) Domain responsibilities

The `sre-domain` primitive is the **single-writer** for SRE aggregates:

- **Incident** — operational events requiring investigation/remediation
- **Alert** — monitoring signals that may trigger incidents
- **Runbook** — documented procedures for operational tasks
- **SLO** — Service Level Objective definitions (budget state, not raw measurements)
- **HealthCheck** — resource health assessments
- **FleetState** — aggregate GitOps/cluster state snapshots

It must:

- Accept change via **commands** (RPC) and/or **domain intents** (bus)
- Emit **domain facts** via transactional outbox (domain is the only fact emitter)
- Enforce business invariants (severity levels, state transitions, SLO constraints)
- Provide **minimal query APIs** (get-by-id, list-by-parent, current state)

**Domain does NOT:**
- Provide search/full-text APIs (projection responsibility)
- Store time-series data (projection responsibility)
- Store raw telemetry (use aggregated windows for SLI measurements)
- Become a read API for dashboards or agents (projection responsibility)

---

## 2) Aggregate definitions

### 2.1 Incident

**Identity:** `incident_id` (UUID)

**Attributes:**
- `title`, `description`
- `severity` — critical | high | medium | low
- `status` — open | acknowledged | investigating | mitigating | resolved | closed
- `source` — alert | manual | agent
- `affected_resources` — list of resource references
- `assigned_to` — responder identity
- `timeline` — ordered list of timeline entries

**State machine:**
```
open → acknowledged → investigating → mitigating → resolved → closed
```

### 2.2 Alert

**Identity:** `alert_id` (fingerprint-based)

**Attributes:**
- `fingerprint`, `name`, `severity`, `status`
- `labels`, `annotations`
- `source` — prometheus | custom
- `starts_at`, `ends_at`
- `incident_id` — correlated incident (optional)

### 2.3 Runbook

**Identity:** `runbook_id` (UUID or slug)

**Attributes:**
- `title`, `description`
- `trigger_conditions` — when this runbook applies
- `steps` — ordered list of procedure steps
- `automation_level` — manual | assisted | automated
- `approval_required` — boolean
- `owner`, `tags`

### 2.4 SLO

**Identity:** `slo_id` (UUID or slug)

**Attributes:**
- `name`, `description`
- `target` — target percentage (e.g., 99.9)
- `window` — measurement window (7d, 28d, 30d)
- `sli_spec` — what to measure (not raw data)
- `current_budget_remaining` — percentage
- `budget_consumed` — percentage
- `owner`, `scope`

**Note:** Domain owns SLO definitions and current budget state. Time-series of SLI measurements is owned by SLOBurndownProjection.

### 2.5 HealthCheck

**Identity:** `healthcheck_id` (UUID)

**Attributes:**
- `resource_ref` — cluster/namespace/app/pod
- `timestamp`
- `status` — healthy | degraded | unhealthy | unknown
- `conditions` — list of condition assessments
- `source` — argocd | probe | agent

### 2.6 FleetState

**Identity:** `fleetstate_id` (UUID per snapshot)

**Attributes:**
- `environment_id`, `cluster_id`, `timestamp`
- `total_applications`, `healthy_count`, `degraded_count`
- `progressing_count`, `unknown_count`, `out_of_sync_count`

---

## 3) Domain query APIs (minimal, strongly consistent)

Domain exposes **minimal** query APIs for get-by-id and bounded lists:

| Service | Operations | Notes |
|---------|------------|-------|
| `SreIncidentQueryService` | GetIncident, ListOpenIncidents | No search, no history |
| `SreAlertQueryService` | GetAlert, ListActiveAlerts | No search |
| `SreRunbookQueryService` | GetRunbook, ListRunbooks | No search |
| `SreSLOQueryService` | GetSLO, ListSLOs | Budget state only |
| `SreHealthQueryService` | GetCurrentFleetState | Current state only |

**What domain query APIs do NOT provide:**
- `SearchIncidents`, `SearchRunbooks` → projection responsibility
- `GetIncidentHistory`, `GetMTTRMetrics` → projection responsibility
- `GetSLITimeSeries`, `GetBurndownChart` → projection responsibility
- `SearchAlertsByLabel`, `GetAlertCorrelations` → projection responsibility

All search, dashboards, history, and time-series queries must go through **projection query services**.

---

## 4) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind domain --name sre --include-gitops
```

### Proto package

Target: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/`

(Following 496/509 canonical path structure)

### Outbox pattern

All domain facts emitted via transactional outbox for exactly-once delivery. Only the domain can emit facts; integrations emit intents.

### SLI measurement aggregation

Domain accepts `RecordSLIWindowRequested` with pre-aggregated measurements (per-minute or per-5-minute windows), not raw request-level metrics. Raw telemetry stays in Prometheus; domain gets windowed summaries.

---

## 5) Acceptance criteria

1. Domain primitive owns all SRE aggregates as single-writer
2. All state changes emit facts via transactional outbox
3. Query APIs are **minimal** (get-by-id, bounded lists, current state)
4. **No search or time-series APIs** in domain (projection responsibility)
5. Aggregate versions enable idempotent downstream consumption
6. Tenant isolation enforced on all queries and commands
7. SLI measurements are aggregated windows, not raw telemetry
