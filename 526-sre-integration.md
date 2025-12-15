# 526 SRE — Integration Primitive Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **sre-integration** primitive — adapters for external systems including ArgoCD, Prometheus/AlertManager, ticketing, and paging systems.

---

## 1) Integration responsibilities

The `sre-integration` primitive implements **external seams**:

- **ArgoCDIntegration** — sync ArgoCD application state to SRE domain
- **AlertManagerIntegration** — receive alerts from Prometheus
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

## 2) ArgoCD Integration

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

## 3) Prometheus/AlertManager Integration

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

## 4) Ticketing Integration

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

## 5) Paging Integration

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

## 6) Knowledge Index (NOT Transformation Domain)

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

## 7) Implementation notes

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

## 8) Acceptance criteria

1. ArgoCD poller emits `RecordFleetStateRequested` and `RecordHealthCheckRequested` **intents** (not facts)
2. AlertManager webhook emits `IngestAlertRequested` **intent** (not fact)
3. **Domain is the only fact emitter** — integrations never emit facts directly
4. Ticketing creates issues by consuming `IncidentCreated` facts
5. Paging escalates by consuming `IncidentCreated` facts (severity gated)
6. **No direct Transformation domain coupling** — pattern lookup uses projection-backed knowledge index
7. All integrations have retry logic, circuit breakers, and idempotency
