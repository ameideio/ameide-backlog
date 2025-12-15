# 526 SRE — Integration Primitive Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **sre-integration** primitive — adapters for external systems including ArgoCD, Prometheus/AlertManager, ticketing, and paging systems.

---

## 1) Integration responsibilities

The `sre-integration` primitive implements **external seams**:

- **ArgoCD adapter** — sync ArgoCD application state to SRE domain
- **Prometheus/AlertManager adapter** — receive alerts
- **Ticketing adapter** — create/update tickets (Jira, GitHub Issues)
- **Paging adapter** — escalate incidents (PagerDuty, OpsGenie)
- **Backlog adapter** — query Transformation domain for known patterns

Integration primitives:

- Translate external contracts to SRE domain intents/facts
- Handle **inbound webhooks** with idempotency
- Perform **outbound calls** with retry and circuit-breaking

---

## 2) ArgoCD Integration

### Purpose

Sync ArgoCD Application state to SRE domain for fleet health tracking.

### Mode

**Polling (default):** Poll ArgoCD API at 30s interval, emit `RecordFleetState` and `RecordHealthCheck` intents for changes.

### Configuration

```yaml
argocd_integration:
  enabled: true
  mode: polling
  poll_interval: 30s
  argocd_server: argocd-server.argocd.svc.cluster.local:443
```

---

## 3) Prometheus/AlertManager Integration

### Purpose

Receive alerts from AlertManager and create SRE domain alerts.

### Webhook receiver

- Endpoint: `/webhooks/alertmanager`
- Deduplicate via fingerprint-based message ID
- Translate to `IngestAlertRequested` intent

### Configuration

```yaml
alertmanager_integration:
  enabled: true
  webhook_path: /webhooks/alertmanager
  auto_create_incident:
    enabled: true
    min_severity: high
```

---

## 4) Ticketing Integration

### Purpose

Create and update tickets in external systems.

### Supported systems

- **Jira** — Create issues, update status, add comments
- **GitHub Issues** — Create issues, add labels, close

### Configuration

```yaml
ticketing_integration:
  enabled: true
  default_provider: jira
  jira:
    base_url: https://company.atlassian.net
    project_key: SRE
```

---

## 5) Paging Integration

### Purpose

Escalate critical incidents to on-call systems.

### Supported systems

- **PagerDuty** — Create/acknowledge/resolve incidents
- **OpsGenie** — Create/acknowledge/close alerts

### Escalation logic

- Page for critical/high severity
- Configurable business hours for non-critical
- Deduplication via incident ID

---

## 6) Backlog Integration

### Purpose

Query Transformation domain for known incident patterns.

### Query interface

Search backlog items by symptoms, components, tags. Return scored matches.

---

## 7) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind integration --name sre --include-gitops
```

### Health checks

All integrations expose health checks for monitoring.

---

## 8) Acceptance criteria

1. ArgoCD poller syncs state every 30 seconds
2. AlertManager webhook receives and deduplicates alerts
3. Ticketing creates incidents in Jira
4. Paging escalates critical incidents to PagerDuty
5. Backlog searches Transformation domain for patterns
6. All integrations have retry logic and circuit breakers
