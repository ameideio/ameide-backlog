## Codex monitor (GitOps-owned; historical rate limits/credits)

### Status

Legacy (slot-based). New work should target the generalized broker model in `backlog/675-codex-broker.md` (n accounts, n sessions, lease-based allocation). This monitor should only remain while any consumers still depend on per-slot `codex-account-status-<slot>` signals.

### Implementation status (GitOps)

- The broker GitOps scaffolding exists and is deployed in dev with a placeholder image to validate the platform wiring; see `backlog/675-codex-broker.md`.
- No new investment is planned in the slot-based monitor once the broker publishes per-account depletion metrics directly (Prometheus/Grafana).

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

In the broker model (`backlog/675-codex-broker.md`), consumers should not read per-slot Secrets directly; they should ask the broker for a leased session and optionally consume broker-provided account depletion snapshots. The monitor remains useful as a transitional interface for `codex-account-status-<slot>` while Coder/Che templates are migrated.

Decision:

- Publish **one object per slot**: `Secret/codex-account-status-<slot>` with `status.json`
- Treat missing/empty `auth.json` as **unusable** (reason: `empty_auth_json`)
- Treat invalid `auth.json` JSON as **unusable** (reason: `invalid_auth_json`)
- Treat inability to authenticate/poll Codex as **unusable** (reason: `codex_error`)
- Compute a simple selection score so consumers can pick the least depleted slot:
  - `score` = min remaining percent across windows (`0..100`, higher is better)

Even though this payload is non-sensitive, using a Secret keeps the distribution pipeline consistent (Vault → ESO → K8s) and makes workspace fan-out straightforward.

### Status schema (consumer contract)

`Secret/codex-account-status-<slot>` key `status.json`:

- `status`: `ok|error|bootstrap` (human-friendly)
- `ok`: boolean (poll succeeded)
- `usable`: boolean (consumer is allowed to use this slot)
- `depleted`: boolean (hard stop signal)
- `score`: int `0..100` (higher = less depleted; used for `auto` selection)
- `reason`: string (e.g. `empty_auth_json`, `invalid_auth_json`, `depleted`, `codex_error`)
- `ts_epoch`: unix seconds (freshness check)
- `windows`: sanitized rate-limit window snapshots (no tokens)

Backwards compatibility: during rollout, consumers also accept the legacy schema `{status:"ok", depleted:false}` when `usable` is missing.

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

- Auth inputs exist for the configured slots (seed + rotating), per `backlog/675-codex-refresher.md`.
- ArgoCD app for the monitor is `Synced/Healthy` and creates `Deployment/Service/ServiceMonitor` per slot.
- `codex_monitor_up{slot="0..2"} == 1` and updates at least once per polling interval.
- `Secret/codex-account-status-<slot>` update periodically and include `usable` + `score` (no secret material).
- Coder tasks (and any other Codex consumers) implement a preflight:
  - read `codex-account-status-<slot>`
  - fail fast (or pick another slot) when `usable=false` or `depleted=true`
  - when multiple slots are usable, pick the highest `score`.

### E2E deployment + verification (GitOps)

1) Seed auth into the “source” Key Vault (manual):
   - `codex-auth-json-b64-0`, `codex-auth-json-b64-1` (and `-2` when ready)
2) Run the Azure apply workflow to sync secrets into the cluster Key Vault.
3) Publish the monitor image (manual workflow):
   - GitHub Actions: `Publish Codex Monitor Image`
   - Take the output digest and pin it in `sources/values/env/dev/foundation/foundation-codex-monitor.yaml`.
4) Let ArgoCD reconcile; then verify (read-only):
   - `kubectl -n ameide-dev get deploy | rg 'codex-monitor-(0|1)'`
   - `kubectl -n ameide-dev get secret | rg 'codex-account-status-(0|1)'`
   - `kubectl -n ameide-dev get secret codex-account-status-0 -o jsonpath='{.data.status\\.json}' | base64 -d`
5) Consumer verification:
   - Create a Coder workspace with `codex_account_slot=auto`
   - Verify `cat $HOME/.codex/account_slot` picks the slot with the higher `score`.
