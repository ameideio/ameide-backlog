# 526 SRE — Integration Primitive Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **sre-integration** primitive — adapters for external systems including Kubernetes, ArgoCD, Prometheus/AlertManager/Grafana/Loki/Tempo, ticketing, and paging systems.

---

## 1) Integration responsibilities

The `sre-integration` primitive implements **external seams**:

- **KubernetesIntegration** — direct K8s API access for resource state, events, logs
- **ArgoCDIntegration** — sync ArgoCD application state to SRE domain
- **ObservabilityIntegration** — OTel/Prometheus/AlertManager/Grafana/Loki/Tempo
- **TicketingIntegration** — create/update tickets (Jira, GitHub Issues)
- **PagingIntegration** — escalate incidents (PagerDuty, OpsGenie)

Integration primitives:

- **Translate external signals into SRE domain intents/commands** (never facts)
- Handle **inbound webhooks** with idempotency and deduplication
- Perform **outbound calls** with retry and circuit-breaking

**Critical rule (per 496/520/528):**
> Integrations **emit intents/commands only**. Only the SRE Domain can emit facts via transactional outbox.

Integrations do NOT:
- Emit domain facts directly (violates single-writer rule)
- Call domain write APIs that bypass command validation
- Query Transformation domain directly (use projection-backed knowledge index)

---

## 2) Kubernetes Integration

### Purpose

Direct access to Kubernetes API for resource state, events, and pod logs. This is the primary source of truth for cluster state beyond what ArgoCD tracks.

### Mode

**Watch + Poll hybrid:**
- **Watch:** Stream events and resource changes in real-time
- **Poll:** Periodic full-state reconciliation (every 60s)

### Capabilities

| Capability | Description |
|------------|-------------|
| **Resource state** | Get/list pods, deployments, services, configmaps, secrets (metadata only), CRDs |
| **Events** | Watch K8s events (warnings, errors, lifecycle events) |
| **Logs** | Fetch pod logs (with tail/since filters) |
| **Exec** | Execute commands in pods (SRECoder only, approval required) |
| **Apply/Delete** | Modify resources (SRECoder only, approval required) |

### Emitted intents

| Intent | When |
|--------|------|
| `RecordHealthCheckRequested` | Pod/deployment health state changes |
| `IngestAlertRequested` | K8s event with Warning type detected |

### Data flow

```
K8s API (watch/poll) → KubernetesIntegration
    ↓
RecordHealthCheckRequested intent → sre.domain.intents.v1 topic
    ↓
SRE Domain (processes intent, persists, emits fact)
    ↓
HealthCheckRecorded fact → sre.domain.facts.v1 topic
```

### Configuration

```yaml
kubernetes_integration:
  enabled: true
  clusters:
    - name: local
      kubeconfig_secret_ref: platform/local-kubeconfig
    - name: dev
      kubeconfig_secret_ref: platform/dev-kubeconfig
    - name: staging
      kubeconfig_secret_ref: platform/staging-kubeconfig
    - name: production
      kubeconfig_secret_ref: platform/production-kubeconfig
  watch:
    namespaces: ["ameide-*", "argocd", "monitoring"]
    resource_types: [pods, deployments, services, events]
  poll_interval: 60s
  log_tail_lines: 500
```

### SRECoder tool exposure

The Kubernetes integration exposes tools to SRECoder (A2A server) for execution:

| Tool | Risk | Approval |
|------|------|----------|
| `k8s.get_resource` | read | none |
| `k8s.list_resources` | read | none |
| `k8s.get_logs` | read | none |
| `k8s.describe_resource` | read | none |
| `k8s.get_events` | read | none |
| `k8s.exec_command` | write | required |
| `k8s.apply_manifest` | write | required |
| `k8s.delete_resource` | write | required |
| `k8s.rollout_restart` | write | required |

---

## 3) ArgoCD Integration

### Purpose

Sync ArgoCD Application state to SRE domain for fleet health tracking.

### Mode

**Polling (default):** Poll ArgoCD API at 30s interval, emit domain intents for changes.

### Emitted intents

| Intent | When |
|--------|------|
| `RecordFleetStateRequested` | Every poll cycle with aggregate fleet state |
| `RecordHealthCheckRequested` | Per-application health changes detected |

### Data flow

```
ArgoCD API → ArgoCDIntegration (poll)
    ↓
RecordFleetStateRequested intent → sre.domain.intents.v1 topic
    ↓
SRE Domain (processes intent, persists, emits fact)
    ↓
FleetStateRecorded fact → sre.domain.facts.v1 topic
```

### Configuration

```yaml
argocd_integration:
  enabled: true
  mode: polling
  poll_interval: 30s
  argocd_server: argocd-server.argocd.svc.cluster.local:443
  environments:
    - local
    - dev
    - staging
    - production
```

---

## 4) Observability Integration (OTel/Grafana/Loki/Tempo)

### Purpose

Unified observability stack integration for metrics, logs, and traces. Provides alert triggers and diagnostic context for incident investigation.

### Components

| Component | Purpose | Integration Mode |
|-----------|---------|------------------|
| **Prometheus** | Metrics storage and alerting rules | Query API |
| **AlertManager** | Alert routing and grouping | Webhook receiver |
| **Grafana** | Dashboards and visualization | API for annotations, dashboard links |
| **Loki** | Log aggregation | Query API for log search |
| **Tempo** | Distributed tracing | Query API for trace lookup |

### Alert sources (inbound)

AlertManager remains the primary alert router, but alerts can originate from:

1. **Prometheus alerting rules** — metric-based alerts
2. **Loki alerting rules** — log-based alerts (error rate, patterns)
3. **Custom OTel collectors** — application-level alerts via OTel SDK

All alerts flow through AlertManager for deduplication and routing.

### Query capabilities (diagnostic context)

| Query | Source | Purpose |
|-------|--------|---------|
| `GetMetricRange(query, start, end)` | Prometheus | Time-series for dashboards, SLI calculation |
| `SearchLogs(query, labels, start, end)` | Loki | Log search during incident investigation |
| `GetTrace(trace_id)` | Tempo | Distributed trace for request debugging |
| `GetSpansByService(service, start, end)` | Tempo | Service-level trace search |

### Emitted intents

| Intent | When |
|--------|------|
| `IngestAlertRequested` | Alert received from AlertManager (any source) |
| `RecordSLIWindowRequested` | Scheduled SLI calculation from Prometheus metrics |
| `CreateIncidentRequested` | Auto-create enabled and severity threshold met |

### Data flow (AlertManager)

```
Prometheus/Loki alerting rules → AlertManager
    ↓
POST /webhooks/alertmanager
    ↓
ObservabilityIntegration (validate, dedupe, translate)
    ↓
IngestAlertRequested intent → sre.domain.intents.v1 topic
    ↓
SRE Domain (processes intent, persists, emits fact)
    ↓
AlertIngested fact → sre.domain.facts.v1 topic
```

### Data flow (SLI calculation)

```
Scheduled job (cron)
    ↓
ObservabilityIntegration queries Prometheus
    ↓
RecordSLIWindowRequested intent → sre.domain.intents.v1 topic
    ↓
SRE Domain (processes intent, updates SLO budget, emits fact)
    ↓
SLIWindowRecorded fact → sre.domain.facts.v1 topic
```

### Configuration

```yaml
observability_integration:
  enabled: true
  prometheus:
    url: http://prometheus.monitoring.svc.cluster.local:9090
    timeout: 30s
  alertmanager:
    webhook_path: /webhooks/alertmanager
    idempotency:
      key_source: fingerprint
      dedup_window: 5m
    auto_create_incident:
      enabled: true
      min_severity: high
  loki:
    url: http://loki.monitoring.svc.cluster.local:3100
    timeout: 30s
  tempo:
    url: http://tempo.monitoring.svc.cluster.local:3200
    timeout: 30s
  grafana:
    url: http://grafana.monitoring.svc.cluster.local:3000
    secret_ref: platform/grafana-api-key
  sli_calculation:
    schedule: "*/5 * * * *"  # Every 5 minutes
```

### SRECoder tool exposure

The Observability integration exposes diagnostic tools to SRECoder:

| Tool | Source | Purpose |
|------|--------|---------|
| `obs.query_metrics` | Prometheus | Execute PromQL query |
| `obs.search_logs` | Loki | Search logs by labels and pattern |
| `obs.get_trace` | Tempo | Fetch trace by ID |
| `obs.search_traces` | Tempo | Search traces by service/operation |
| `obs.get_dashboard_url` | Grafana | Generate dashboard link with time range |

---

## 5) Prometheus/AlertManager Integration (Legacy)

> **Note:** This section is superseded by §4 Observability Integration. Retained for backwards compatibility during migration.

### Purpose

Receive alerts from AlertManager and create SRE domain alerts.

### Webhook receiver

- Endpoint: `/webhooks/alertmanager`
- Deduplicate via fingerprint-based idempotency key
- Translate AlertManager payload to `IngestAlertRequested` intent

### Emitted intents

| Intent | When |
|--------|------|
| `IngestAlertRequested` | Alert received from AlertManager |
| `CreateIncidentRequested` | Auto-create enabled and severity threshold met |

### Data flow

```
AlertManager → POST /webhooks/alertmanager
    ↓
AlertManagerIntegration (validate, dedupe, translate)
    ↓
IngestAlertRequested intent → sre.domain.intents.v1 topic
    ↓
SRE Domain (processes intent, persists, emits fact)
    ↓
AlertIngested fact → sre.domain.facts.v1 topic
```

### Configuration

```yaml
alertmanager_integration:
  enabled: true
  webhook_path: /webhooks/alertmanager
  idempotency:
    key_source: fingerprint
    dedup_window: 5m
  auto_create_incident:
    enabled: true
    min_severity: high
```

---

## 6) Ticketing Integration

### Purpose

Create and update tickets in external systems when incidents are created/updated.

### Supported systems

- **Jira** — Create issues, update status, add comments
- **GitHub Issues** — Create issues, add labels, close

### Consumed facts

| Fact | Action |
|------|--------|
| `IncidentCreated` | Create ticket in external system |
| `IncidentResolved` | Update ticket status to resolved |
| `IncidentClosed` | Close ticket |

### Configuration

```yaml
ticketing_integration:
  enabled: true
  default_provider: jira
  jira:
    base_url: https://company.atlassian.net
    project_key: SRE
    secret_ref: platform/jira-credentials
  github:
    repo: org/sre-incidents
    secret_ref: platform/github-token
```

---

## 7) Paging Integration

### Purpose

Escalate critical incidents to on-call systems.

### Supported systems

- **PagerDuty** — Create/acknowledge/resolve incidents
- **OpsGenie** — Create/acknowledge/close alerts

### Consumed facts

| Fact | Action |
|------|--------|
| `IncidentCreated` (severity=critical/high) | Page on-call |
| `IncidentAcknowledged` | Acknowledge page |
| `IncidentResolved` | Resolve page |

### Escalation logic

- Page immediately for critical severity
- Page for high severity during business hours (configurable)
- Deduplication via incident ID

### Configuration

```yaml
paging_integration:
  enabled: true
  default_provider: pagerduty
  pagerduty:
    routing_key_ref: platform/pagerduty-routing-key
  escalation:
    critical: immediate
    high: business_hours_only
    business_hours:
      start: "09:00"
      end: "18:00"
      timezone: UTC
```

---

## 8) Knowledge Index (NOT Transformation Domain)

### Purpose

Provide pattern lookup for the 525 backlog-first triage workflow.

### Critical architectural decision

> **SRE runtime processes/agents do NOT query Transformation domain directly.**

Instead, pattern lookup uses the **KnowledgeIndexQueryService**, a projection-backed query service that:

- Consumes Transformation domain facts (backlog item changes)
- Builds a searchable index (full-text + vector embeddings)
- Exposes stable query APIs for pattern matching

### Query interface

SRE agents and processes call:
- `SearchPatterns(symptoms, tags)` → scored matches with backlog item IDs
- `GetSimilarIncidents(incident_description)` → semantically similar past incidents
- `SearchRunbooksBySemantic(query)` → vector search over runbook content

This service is part of **sre-projection** (or a shared platform projection), NOT an integration adapter.

See: [526-sre-projection.md](526-sre-projection.md) §KnowledgeIndexProjection

---

## 9) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind integration --name sre --include-gitops
```

### Health checks

All integrations expose health checks for monitoring upstream/downstream connectivity.

### Idempotency

All inbound webhooks must:
- Extract idempotency key from payload (fingerprint, event ID, etc.)
- Deduplicate within configured window
- Return 200 OK for duplicates (silent success)

### Retry and circuit breaking

All outbound calls must:
- Implement exponential backoff retry
- Use circuit breaker for failing upstreams
- Log failures with correlation IDs

---

## 10) Acceptance criteria

1. **Kubernetes integration** provides resource state, events, logs across all clusters
2. ArgoCD poller emits `RecordFleetStateRequested` and `RecordHealthCheckRequested` **intents** (not facts)
3. **Observability integration** unifies Prometheus/AlertManager/Loki/Tempo access
4. AlertManager webhook emits `IngestAlertRequested` **intent** (not fact)
5. **Domain is the only fact emitter** — integrations never emit facts directly
6. Ticketing creates issues by consuming `IncidentCreated` facts
7. Paging escalates by consuming `IncidentCreated` facts (severity gated)
8. **No direct Transformation domain coupling** — pattern lookup uses projection-backed knowledge index
9. All integrations have retry logic, circuit breakers, and idempotency
10. **SRECoder has tool access** to K8s and observability APIs (read without approval, write with approval)
