# Observability Target State (Grafana + Loki + Tempo)

_Last updated: 2025-11-09_

## 1. Objectives

1. Give every service a consistent telemetry bootstrap (logs, traces, metrics) so engineers can pivot between signals in Grafana within seconds.
2. Keep observability infrastructure internal-only while still tagging telemetry with `tenant_id` / `saas.tenant` for analytics and cost attribution.
3. Operate a maintainable pipeline (Alloy + OpenTelemetry Collector + Loki + Tempo + Grafana) that can scale to production without bespoke per-team work.

## 2. Platform architecture (target)

| Layer | Target state |
| --- | --- |
| **Emitters** | All services emit JSON logs with `trace_id`, `span_id`, `tenant_id`, `request_id`, `service` metadata and bootstrap language-appropriate OTel SDKs (Go, Node, Python). |
| **Log pipeline** | `alloy-logs` DaemonSet tails pods via `loki.source.kubernetes`, parses JSON, sets labels, and forwards to Loki. `stage.tenant` continues to populate `X-Scope-OrgID` even though Loki currently runs single-tenant. |
| **Trace/metrics gateway** | A centralized OpenTelemetry Collector Deployment receives OTLP, enriches with `k8sattributes`, applies tail sampling (errors 100%, ≥1s latency 100%, baseline 5%), and exports to Tempo. |
| **Backends** | Loki (`infra/kubernetes/values/infrastructure/loki.yaml`) runs SingleBinary with `auth_enabled=false`; Tempo (`infra/kubernetes/values/infrastructure/tempo.yaml`) runs single-tenant with `multitenancyEnabled=false`; both retain tenant metadata for analytics and can re-enable auth via Helm flags. |
| **Grafana** | Datasources defined in `infra/kubernetes/values/infrastructure/grafana-datasources.yaml` (Prometheus, Loki, Tempo). No header injection required; derived fields + trace correlations remain enabled. |
| **Governance** | Telemetry guard-rails codified in backlog/334 plus service/infra READMEs; CI ensures services call the bootstrap and log schemas remain valid. |

## 3. Logging pipeline (Loki)

### 3.1 Service requirements
- Emit structured JSON with correlation IDs; do not log raw PII.
- Forward W3C trace context on every RPC/HTTP call so downstream spans/logs correlate automatically.
- Surface tenant metadata in logs (`tenant_id`, `saas.tenant`) even though Loki is single-tenant; this drives dashboards and future limits.

### 3.2 Alloy configuration
- `loki.source.kubernetes` tails pods (no hostPath required) and forwards to `loki.process`.
  - `stage.json` extracts fields, `stage.labels` keeps low-cardinality labels (`cluster`, `namespace`, `app`, `env`, `level`).
  - `stage.tenant { source = "tenant_id" }` is a no-op while Loki runs single-tenant but ensures we can flip multi-tenancy back on without changing log pipelines.
  - `loki.write.default` sends to `http://loki-gateway.ameide.svc.cluster.local/loki/api/v1/push`.

### 3.3 Loki configuration highlights
- SingleBinary mode, 7-day retention, MinIO-backed storage.
- `limits_config` enforces ingestion caps and max streams globally (shared tenant).
- `auth_enabled=false`; runtime tenant overrides removed.

## 4. Tracing pipeline (Tempo + OTel Collector)

### 4.1 Collector pipeline
```
receivers:
  otlp:
    protocols:
      http:
      grpc:

processors:
  k8sattributes: {}
  batch: {}
  tailsampling:
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: latency
        type: latency
        latency: { threshold_ms: 1000 }
      - name: baseline
        type: probabilistic
        probabilistic: { sampling_percentage: 5 }

exporters:
  otlphttp/tempo:
    endpoint: http://tempo.ameide:4318

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [k8sattributes, tailsampling, batch]
      exporters: [otlphttp/tempo]
```

### 4.2 Tempo configuration highlights
- Single-binary chart with `multitenancyEnabled=false` and inline `auth_enabled: false`.
- Local filesystem storage (PVC) with 7-day retention and metrics generator enabled (remote_write to Prometheus for span metrics + exemplars).
- Tail sampling decisions happen in the collector; Tempo stores whatever it receives.

## 5. Grafana & analytics
- Datasources:
  - Prometheus (default) → `http://prometheus-kube-prometheus-prometheus.ameide:9090`.
  - Loki → `http://loki.ameide:80`.
  - Tempo → `http://tempo.ameide:3200`.
- Derived fields remain: Loki regex extracts `trace_id` and links to Tempo; Tempo correlations map spans → logs using `service.name`, `trace_id`, ±5m window.
- Dashboards provisioned via `infra/kubernetes/values/infrastructure/grafana.yaml` (platform RPC, graph, threads, observability, etc.).
- Grafana admin credentials come from the `grafana-admin-credentials` Secret, which the Grafana chart now provisions itself via an `extraObjects`-backed ExternalSecret (see `platform-grafana.yaml`). No separate Layer 15 bundle is required; as soon as the foundation `ClusterSecretStore` and Vault bootstrap are healthy, Grafana can reconcile end-to-end. Credential rotation is handled entirely in Vault—update the `grafana-admin-user` / `grafana-admin-password` keys and annotate the ExternalSecret with `external-secrets.io/refresh=true` to fan the change out cluster-wide without manual Secrets.

### 5.1 Grafana admin credential lifecycle
1. Vault fixture seed (dev) or production ceremony writes `grafana-admin-user` / `grafana-admin-password`.
2. `foundation-vault-secret-store` exposes those keys via the `ameide-vault` ClusterSecretStore.
3. `platform-grafana` renders an ExternalSecret (`grafana-admin-credentials-sync`) via `extraObjects`, producing the runtime Secret.
4. Rotation: update the Vault keys, annotate the ExternalSecret, confirm Argo shows the Application `Healthy`, and sign in with the new values. No inline YAML or Helm overrides should reintroduce credentials into the repo.

## 6. Operations & governance

### 6.1 Deploy / verify
1. Wait for the foundation ApplicationSet (waves 15‑21) to finish so Vault + the `ClusterSecretStore` exist.
2. Sync `platform-grafana`, `platform-prometheus`, `platform-loki`, `platform-tempo`, and `platform-observability-smoke` via Argo (`kubectl patch application <name> ... refresh=hard` or `argocd app sync -l tier=platform,wave=wave41`).
3. Inspect Grafana/Prometheus/Tempo logs (`kubectl logs -n ameide <pod>`), verify the `grafana-admin-credentials` ExternalSecret is `Ready=True`, then run observability smoke tests (`argocd app sync platform-observability-smoke` or `helm test observability-smoke` via the chart).

### 6.2 CI / quality gates
- Telemetry lint checks ensure service entrypoints call the shared bootstrap (language-specific tests TBD).
- JSON schema tests validate log shape (tenant + trace IDs present).
- Infra tests confirm tail sampling config renders and that Loki/Tempo pods are single-tenant.

### 6.3 Future-ready toggles
- To re-enable multi-tenancy later:
  1. Set `loki.auth_enabled=true` and restore tenant limits/runtime config.
  2. Set `tempo.multitenancyEnabled=true` and `config.auth_enabled=true`.
  3. Ensure Alloy + OTel exporters add `X-Scope-OrgID` based on `tenant_id` or `saas.tenant`.
  4. Point Grafana datasources at a proxy that injects the header (or configure `secureJsonData`).

## 7. Single source of truth
- This document replaces the old runbook—no other observability doc is authoritative.
- Infra values: `infra/kubernetes/values/infrastructure/{loki,tempo,grafana,grafana-datasources}.yaml`.
- Pipelines: `infra/kubernetes/values/infrastructure/alloy-*.yaml`.

## 8. Reference docs
1. [Trace Context (W3C)](https://www.w3.org/TR/trace-context/)
2. [Alloy log source](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/)
3. [Loki multi-tenancy (for future toggle)](https://grafana.com/docs/loki/latest/operations/multi-tenancy/)
4. [Tempo multi-tenancy](https://grafana.com/docs/tempo/latest/operations/manage-advanced-systems/multitenancy/)
5. [Grafana derived fields](https://grafana.com/docs/grafana/latest/datasources/loki/configure-loki-data-source/)
6. [Tail sampling policies](https://grafana.com/docs/tempo/latest/operations/manage-advanced-systems/multitenancy/)
