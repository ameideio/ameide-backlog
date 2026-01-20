# Observability Blueprint – Single-Tenant Stack (superseded)

This document is superseded by `backlog/300-400/334-logging-tracing-v3.md` (authoritative).

Status as of **Nov 1 2025**. The entire fleet now exports OpenTelemetry logs, traces, and metrics over OTLP gRPC to the shared collector (`otel-collector:4317`). Loki and Tempo remain multitenant (per-tenant limits in Loki, Tempo’s `multitenancyEnabled: true`), and Grafana exposes derived-field powered drill-downs across logs, traces, and metrics.

## Latest implementation highlights

- **Fleet instrumentation** – Platform, Repository, Chat, Workflow, Inference, Inference Gateway, Agents, Agents Runtime, Workflows Runtime, and `www-ameide-platform` call their telemetry bootstrap on startup, emit JSON logs with `trace_id`/`span_id`/request metadata, and propagate W3C Trace Context.
- **Collector pipeline** – The OpenTelemetry Collector deployment is the single ingress point for OTLP (logs, traces, metrics). Services export directly to it; the collector applies memory limiting + batching and fans out to Tempo, Loki, and Prometheus (`infra/kubernetes/values/infrastructure/otel-collector.yaml`).
- **Unified transport** – All services (Go, Node.js, Python) speak OTLP gRPC; the collector’s HTTP listener has been removed to simplify configuration.
- **Grafana experience** – Datasources now declare Loki derived fields + Tempo trace-to-logs bridges, enabling 1‑click pivots, and dashboards exist for Platform, Graph, Inference, and Workflows with exemplars turned on.
- **Runbooks** – `docs/runbooks/observability.md` documents helmfile sync, collector validation, and Grafana troubleshooting so on-call engineers can recover the stack quickly.

---

## 1. Reference architecture

- **Workloads** emit structured JSON logs to stdout and instrument OpenTelemetry with W3C Trace Context propagation. ([W3C][1])
- **Collection tier**: a single OpenTelemetry Collector deployment receives every workload’s OTLP gRPC traffic (logs/traces/metrics), adds resource attributes, enforces basic buffering/memory limits, and forwards to backends. ([OpenTelemetry][13])
- **Backends**:
  - **Loki** (multitenant) stores logs with per-tenant limits + S3-backed storage. ([Grafana Labs][3])
  - **Tempo** (multitenant) stores traces; the metrics generator (service-graphs + span-metrics) feeds Prometheus exemplars. ([Grafana Labs][4], [Grafana Labs][5])
  - **Prometheus/Mimir** (optional) collects metrics with exemplars for trace pivots.
- **Grafana** hosts dashboards and Explore. Derived fields and trace correlations provide 1-click navigation between logs, traces, and metrics. ([Grafana Labs][6])
- **Why no Alloy?** The fleet now exports OTLP logs directly from the services; removing Alloy simplified the pipeline and reduced operational burden while keeping the door open for future log-tailers if needed.

---

## 2. Logging (Loki) – best practices

### 2.1 Emit structured JSON with trace context

- Services use Pino/structlog/slog formatters that inject `service_name`, `trace_id`, `span_id`, log level, and request identifiers. OTel SDKs supply correlation helpers where available. ([OpenTelemetry][8])

### 2.2 Export over OTLP, keep labels low-cardinality

- Each service emits logs via its language SDK (Pino + hooks in Node, structlog/slog in Python/Go) and forwards them over OTLP gRPC to `otel-collector:4317`. The collector’s `logs` pipeline fan-outs to Loki using the `loki` exporter defined in `infra/kubernetes/values/infrastructure/otel-collector.yaml`.
- **Label hygiene**: keep labels stable (`service.name`, `deployment.environment`, `saas.tenant`). Never promote `trace_id`, `user_id`, or other high-cardinality fields to labels—leave them inside the JSON payload so Grafana derived fields can pick them up. ([Grafana Labs][10])
- **Resource attributes win**: because logs ride over OTLP, populate `service.namespace`, `service.instance.id`, and domain-specific attributes (tenant, agent, workflow identifiers). They automatically land in Loki labels without a side-car log pipeline.

---

## 3. Tracing (Tempo) – best practices

### 3.1 Instrument applications

- Adopt the official OpenTelemetry SDKs or auto-instrumentation and propagate context via W3C headers. ([W3C][1])
- Populate `service.name`, `service.namespace`, `service.version`, and `deployment.environment` so Grafana can group spans by component. ([OpenTelemetry][12])

### 3.2 Collector gateway

- The shared collector focuses on stability: `memory_limiter` protects the process, `resource` adds environment labelling, and `batch` smooths exporter pressure. Tail sampling is deferred until we have enough signal that per-service sampling is insufficient. ([OpenTelemetry][13], [OpenTelemetry][15])

```yaml
receivers:
  otlp:
    protocols:
      grpc: {}

processors:
  memory_limiter:
    check_interval: 5s
    limit_mib: 409
    spike_limit_mib: 128
  resource:
    attributes:
      - key: environment
        value: production|staging|local
        action: upsert
  batch: {}

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls: { insecure: true }
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  prometheus:
    endpoint: 0.0.0.0:9090

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [otlp/tempo]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [loki]
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, resource, batch]
      exporters: [prometheus]
```

---

## 4. Grafana correlations (Logs ↔ Traces ↔ Metrics)

1. **Logs → Traces**: Loki data source has a derived field that regex-extracts `trace_id` from JSON (`trace_id":"(?P<trace>[0-9a-f]{32})"`) and links to Tempo. ([Grafana Labs][6])
2. **Traces → Logs**: Tempo data source Trace-to-Logs correlations look up namespace + `trace_id` in Loki so span sidebars show matching log lines. ([Grafana Labs][18])
3. **Metrics → Traces**: Dashboards query Prometheus histograms/counters with exemplars; clicking an exemplar opens the originating trace in Tempo. ([Grafana Labs][5])

---

## 5. Grafana dashboards, alerts, and operational guard-rails

### Dashboards shipped

- **AMEIDE / Platform Overview** – error rates, percentile latency, dependency traffic for Platform, Repository, Chat, Workflow. Panels include exemplar links and “top failing RPCs”.
- **AMEIDE / RPC Observability** – gRPC client/server latency distributions, throughput per method, “top failing RPCs” table with quick Tempo links, and SLO capsule panels.
- **Inference Operations** – request rate, latency buckets, streaming completion ratio, and model/tool error codes. Alerts fire when p95 latency >1.5 s or error ratio >5 % for 5 min.
- **Workflows Runtime Health** – Temporal queue depth, activity runtimes, retries; log panels show the latest failure summaries aligned with the selected range.

### Grafana Explore & automation

- Derived fields and trace correlations enable 1-click navigation among logs, traces, and metrics.
- Alert rules (multi-window burn rate) cover gRPC availability, inference latency, workflows failure rate, and 500 ratio; Loki anomaly rules watch for WARN/ERROR spikes. Alertmanager routes by squad/severity.
- Synthetic checks hit onboarding and graph APIs; failures attach trace and log links directly in Grafana notifications.
- Runbooks reference the dashboards above, guiding on-call engineers from “Platform Overview” → “RPC Observability” → Tempo → Loki.

---

## Implementation status (2025‑10‑31)

- Platform gRPC: AsyncLocalStorage scope carries request metadata; spans record request/response/error events; Postgres instrumentation is enabled.
- Agents service: context-aware `slog` handler injects trace/span IDs; OTLP exporters retry with backoff; Connect interceptors manage propagation.
- Workflows runtime: `setup_telemetry` runs at startup; structlog JSON binds workflows/execution IDs; activity decorators measure duration and produce histogram metrics.
- Inference service: shared observability module handles FastAPI + gRPC instrumentation; metrics plus spans emit exemplars for Grafana.
- Inference gateway: OTLP exporters wired; `otelconnect` interceptor, structured `slog` with trace/span fields; `/health` reports exporter readiness.
- `docs/runbooks/observability.md` updated with helmfile sync steps, collector validation, and Grafana verification playbooks.

---

## Next steps

1. Validate refreshed dashboards/alerts in staging, then promote to production with the latest collector configuration.
2. Add CI smoke tests that fail when services skip telemetry bootstrap or Grafana derived fields stop resolving `trace_id`.
3. Extend runbooks with “first five minutes” checklists per dashboard and capture common Loki/Tempo query recipes for the wider team.

---

## References

[1]: https://www.w3.org/TR/trace-context/?utm_source=threadsgpt.com "Trace Context"
[3]: https://grafana.com/docs/loki/latest/fundamentals/?utm_source=threadsgpt.com "Loki fundamentals | Grafana documentation"
[4]: https://grafana.com/docs/tempo/latest/fundamentals/?utm_source=threadsgpt.com "Tempo fundamentals | Grafana documentation"
[5]: https://grafana.com/docs/tempo/latest/getting-started/metrics-from-traces/?utm_source=threadsgpt.com "Metrics from traces | Grafana Tempo documentation"
[6]: https://grafana.com/docs/grafana/latest/datasources/loki/configure-loki-data-source/?utm_source=threadsgpt.com "Configure the Loki data source | Grafana documentation"
[8]: https://opentelemetry.io/docs/languages/dotnet/logs/correlation/?utm_source=threadsgpt.com "Log correlation | OpenTelemetry"
[10]: https://grafana.com/docs/loki/latest/get-started/labels/bp-labels/?utm_source=threadsgpt.com "Label best practices | Grafana Loki documentation"
[11]: https://grafana.com/docs/loki/latest/send-data/otel/?utm_source=threadsgpt.com "Ingesting logs to Loki using OpenTelemetry Collector"
[12]: https://opentelemetry.io/docs/specs/semconv/resource/?utm_source=threadsgpt.com "Resource semantic conventions"
[13]: https://opentelemetry.io/docs/collector/deployment/gateway/?utm_source=threadsgpt.com "Gateway deployment | OpenTelemetry Collector"
[15]: https://opentelemetry.io/docs/platforms/kubernetes/collector/components/?utm_source=threadsgpt.com "Collector components for Kubernetes"
[18]: https://grafana.com/docs/grafana/latest/datasources/tempo/traces-in-grafana/trace-correlations/?utm_source=threadsgpt.com "Trace correlations | Grafana documentation"
