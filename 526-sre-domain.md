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
- **SLO** — Service Level Objective definitions
- **SLI** — Service Level Indicator measurements
- **HealthCheck** — resource health assessments
- **FleetState** — aggregate GitOps/cluster state snapshots

It must:

- Accept change via **commands** (RPC) and/or **domain intents** (bus)
- Emit **domain facts** via transactional outbox
- Enforce business invariants (severity levels, state transitions, SLO constraints)
- Provide **query services** for read-only access

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
- `backlog_refs` — links to Transformation backlog items
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
- `incident_id` — correlated incident

### 2.3 Runbook

**Identity:** `runbook_id` (UUID or slug)

**Attributes:**
- `title`, `description`
- `trigger_conditions` — when this runbook applies
- `steps` — ordered list of procedure steps
- `automation_level` — manual | assisted | automated
- `approval_required` — boolean
- `owner`

### 2.4 SLO

**Identity:** `slo_id` (UUID or slug)

**Attributes:**
- `name`, `description`
- `target` — target percentage (e.g., 99.9)
- `window` — measurement window (7d, 28d, 30d)
- `sli_query` — how to measure the SLI
- `error_budget_policy`
- `owner`, `scope`

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

## 3) Query services

- **SreIncidentQueryService** — list/get incidents, search, timeline
- **SreAlertQueryService** — list/get alerts, active alerts
- **SreRunbookQueryService** — list/get runbooks, search by symptoms
- **SreSLOQueryService** — list/get SLOs, budget status
- **SreHealthQueryService** — get fleet state, health history

---

## 4) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind domain --name sre --include-gitops
```

### Proto package

Target: `ameide_core_proto.sre.core.v1`

### Outbox pattern

All domain facts emitted via transactional outbox for exactly-once delivery.

---

## 5) Acceptance criteria

1. Domain primitive owns all SRE aggregates as single-writer
2. All state changes emit facts via transactional outbox
3. Query services provide read-only access
4. Aggregate versions enable idempotent downstream consumption
5. Tenant isolation enforced on all queries and commands
