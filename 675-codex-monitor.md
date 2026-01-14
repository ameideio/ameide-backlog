## Codex monitor (GitOps-owned; historical rate limits/credits)

### Goal

Persist and visualize Codex `/status`-equivalent fields (especially rate-limit windows, resets, credits/remaining %) **historically** per Codex slot, so we can:

- see depletion trends over time (Grafana)
- alert before exhaustion (Prometheus alerting)
- expose a simple “is this slot safe to use?” signal in Kubernetes for consumers (Coder tasks)

### Non-goals

- Storing history in Kubernetes objects (ConfigMap/Secret/Events/CRDs).
- Recreating the Codex TUI `/status` output exactly.
- Writing credentials (this component is read-only consumer of the rotating auth secret).

### Inputs (from 675-codex + 675-codex-refresher)

- Slot model: only numeric slots `0`, `1`, `2` (no `default`) (see `backlog/675-codex-refresher.md`).
- Auth source: mount the **rotating** secret for each slot, e.g. `Secret/codex-auth-rotating-0`.
- Retrieval: spawn `codex app-server` and call:
  - `initialize` → `initialized`
  - `account/rateLimits/read` (primary output)
  - optional: `config/read` + `thread/start` for additional metadata (no token-spending turn required)
  - reference protocol: `backlog/675-codex.md`

### Consumer state decision (K8s-visible; per-slot; Secret)

Consumers (especially Coder tasks) need a fast, in-cluster preflight signal to avoid using an exhausted slot.

Decision:

- Publish **one object per slot**: `Secret/codex-account-status-<slot>` with `status.json`
- Treat missing/empty rotating `auth.json` as **depleted/unusable** (reason: `missing_auth`)
- Treat inability to authenticate to Codex (e.g. `account/rateLimits/read` auth error) as **depleted/unusable** (reason: `auth_error`)

Even though this payload is non-sensitive, using a Secret keeps the distribution pipeline consistent (Vault → ESO → K8s) and makes workspace fan-out straightforward.

### K8s-native architecture (recommended)

Expose the data as Prometheus metrics; store history in Prometheus (and optionally remote-write to long-term storage like Thanos/Mimir).

**Preferred pattern (standard): Deployment exporter → Prometheus scrape**

- `Deployment/codex-monitor-<slot>` runs an HTTP server:
  - `GET /metrics` (Prometheus exposition format)
  - optional `GET /status.json` (sanitized; no secrets/tokens; safe only inside cluster)
- Internally polls Codex on a schedule (e.g. every 60s) and serves the **latest cached** metrics.
  - Avoid polling on every scrape to prevent thundering-herd from Prometheus.
- `Service` + `ServiceMonitor` so Prometheus Operator scrapes it.
- Optional `PrometheusRule` alerts (credits low, scrape failures, staleness).

**Additionally (for consumers): publish current slot status into K8s**

Prometheus is great for history/alerts, but not a universal dependency for arbitrary consumers (e.g., Coder tasks). The monitor should also publish a **non-sensitive** “current status” snapshot into Kubernetes so consumers can fail fast before using an exhausted slot.

Recommended distribution (GitOps-owned):

- Monitor writes to Vault KV:
  - key: `codex-account-status-<slot>`
  - field: `value`
- ExternalSecrets materializes:
  - `ExternalSecret/codex-account-status-sync-<slot>` → `Secret/codex-account-status-<slot>` (key: `status.json`)

The payload must contain no tokens/IDs; only depletion-relevant fields (remaining/limit/reset, plus timestamps).

**Alternative pattern (only if Pushgateway exists): CronJob → Pushgateway**

- `CronJob` runs every N minutes and pushes metrics to Pushgateway.
- Prometheus scrapes Pushgateway.

### Metric contract (example)

Expose metrics that are safe and stable:

- `codex_rate_limit_remaining{slot,window}` gauge
- `codex_rate_limit_limit{slot,window}` gauge
- `codex_rate_limit_reset_timestamp_seconds{slot,window}` gauge
- `codex_credits_remaining{slot}` gauge (if available in API)
- `codex_monitor_up{slot}` gauge (1 on last successful poll, else 0)
- `codex_monitor_last_success_timestamp_seconds{slot}` gauge
- `codex_monitor_poll_duration_seconds{slot}` histogram (optional)

Avoid exporting any identifying tokens, refresh tokens, raw auth.json, or user/account IDs.

### GitOps ownership / placement

Create a new GitOps component (dev first):

- Domain: `observability` (recommended) or `foundation` if treated as a platform secret-adjacent concern.
- One Deployment/Service/ServiceMonitor per slot, driven by `accounts[]`/`slots[]` values.
- Use digest-pinned image refs.
- Use `imagePullSecrets: [ghcr-pull]` (private GHCR images).

### Multi-account plan

Model slots explicitly (consistent with refresher):

- One rotating auth secret per slot: `codex-auth-rotating-<slot>`
- One exporter per slot: `codex-monitor-<slot>`
- One status secret per slot: `codex-account-status-<slot>`
- Labels include `ameide.io/codex-account-slot=<slot>` for filtering dashboards/alerts.

### Alerts (starter set)

- `codex_monitor_up == 0` for > 10m (exporter cannot poll Codex)
- `time() - codex_monitor_last_success_timestamp_seconds > 10m` (stale metrics)
- `codex_rate_limit_remaining / codex_rate_limit_limit < 0.2` (window depletion)
- `codex_credits_remaining < <threshold>` (if credits are exposed)

### Acceptance criteria (dev)

- Prometheus scrapes the exporter(s) and stores time series.
- Grafana dashboard shows remaining/used over time per slot/window.
- Alerts fire when thresholds are crossed (dry-run OK).
- `Secret/codex-account-status-0..2` exist (in env + workspace namespaces) so consumers can inspect them before using a slot.

### Definition of done (slots-only)

- Auth inputs exist for all slots (seed + rotating), per `backlog/675-codex-refresher.md`.
- ArgoCD app for the monitor is `Synced/Healthy` and creates `Deployment/Service/ServiceMonitor` per slot.
- `codex_monitor_up{slot="0..2"} == 1` and updates at least once per polling interval.
- `Secret/codex-account-status-0..2` update periodically and include `depleted` + reset/remaining fields (no secret material).
- Coder tasks (and any other Codex consumers) implement a preflight:
  - read `codex-account-status-<slot>`
  - fail fast (or pick another slot) when `depleted=true` or remaining is below policy threshold.
