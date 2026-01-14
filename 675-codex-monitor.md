## Codex monitor (GitOps-owned; historical rate limits/credits)

### Goal

Persist and visualize Codex `/status`-equivalent fields (especially rate-limit windows, resets, credits/remaining %) **historically** per Codex account, so we can:

- see depletion trends over time (Grafana)
- alert before exhaustion (Prometheus alerting)

### Non-goals

- Storing history in Kubernetes objects (ConfigMap/Secret/Events/CRDs).
- Recreating the Codex TUI `/status` output exactly.
- Writing credentials (this component is read-only consumer of the rotating secret).

### Inputs (from 675-codex + 675-codex-refresher)

- Auth source: mount the **rotating** secret for each account, e.g. `Secret/codex-auth-rotating-default` (see `backlog/675-codex-refresher.md`).
- Retrieval: spawn `codex app-server` and call:
  - `initialize` → `initialized`
  - `account/rateLimits/read` (primary output)
  - optional: `config/read` + `thread/start` for additional metadata (no token-spending turn required)
  - reference protocol: `backlog/675-codex.md`

### K8s-native architecture (recommended)

Expose the data as Prometheus metrics; store history in Prometheus (and optionally remote-write to long-term storage like Thanos/Mimir).

**Preferred pattern (standard): Deployment exporter → Prometheus scrape**

- `Deployment/codex-monitor-<account>` runs an HTTP server:
  - `GET /metrics` (Prometheus exposition format)
  - optional `GET /status.json` (sanitized; no secrets/tokens)
- Internally polls Codex on a schedule (e.g. every 60s) and serves the **latest cached** metrics.
  - Avoid polling on every scrape to prevent thundering-herd from Prometheus.
- `Service` + `ServiceMonitor` so Prometheus Operator scrapes it.
- Optional `PrometheusRule` alerts (credits low, scrape failures, staleness).

**Alternative pattern (only if Pushgateway exists): CronJob → Pushgateway**

- `CronJob` runs every N minutes and pushes metrics to Pushgateway.
- Prometheus scrapes Pushgateway.

### Metric contract (example)

Expose metrics that are safe and stable:

- `codex_rate_limit_remaining{account,window}` gauge
- `codex_rate_limit_limit{account,window}` gauge
- `codex_rate_limit_reset_timestamp_seconds{account,window}` gauge
- `codex_credits_remaining{account}` gauge (if available in API)
- `codex_monitor_up{account}` gauge (1 on last successful poll, else 0)
- `codex_monitor_last_success_timestamp_seconds{account}` gauge
- `codex_monitor_poll_duration_seconds{account}` histogram (optional)

Avoid exporting any identifying tokens, refresh tokens, raw auth.json, or user/account IDs.

### GitOps ownership / placement

Create a new GitOps component (dev first):

- Domain: `observability` (recommended) or `foundation` if treated as a platform secret-adjacent concern.
- One Deployment/Service/ServiceMonitor per account, driven by `accounts[]` values.
- Use digest-pinned image refs.
- Use `imagePullSecrets: [ghcr-pull]` (private GHCR images).

### Multi-account plan

Model accounts explicitly (consistent with refresher):

- One rotating secret per account: `codex-auth-rotating-<account>`
- One exporter per account: `codex-monitor-<account>`
- Labels include `ameide.io/codex-account=<account>` for filtering dashboards/alerts.

### Alerts (starter set)

- `codex_monitor_up == 0` for > 10m (exporter cannot poll Codex)
- `time() - codex_monitor_last_success_timestamp_seconds > 10m` (stale metrics)
- `codex_rate_limit_remaining / codex_rate_limit_limit < 0.2` (window depletion)
- `codex_credits_remaining < <threshold>` (if credits are exposed)

### Acceptance criteria (dev)

- Prometheus scrapes the exporter(s) and stores time series.
- Grafana dashboard shows remaining/used over time per account/window.
- Alerts fire when thresholds are crossed (dry-run OK).
