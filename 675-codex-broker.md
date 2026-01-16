## Codex broker (GitOps-owned; n accounts; n sessions; lease-based)

### Problem

Codex ChatGPT auth uses rotating refresh tokens persisted to `$CODEX_HOME/auth.json`.
If two independent runtimes share the same `auth.json` (same refresh-token chain) they will eventually race a refresh: one rotates the token and the other hits `refresh_token_reused` and becomes unusable.

Typical “shared auth file” approaches address **account selection** and **depletion awareness**, but they do not provide a general mechanism for:

- allocating **exclusive** auth “sessions” to concurrent consumers (devcontainers, tasks, jobs)
- scaling to **n accounts** with **n sessions** per account (many concurrent consumers) without `refresh_token_reused`
- a human-friendly workflow to create and rotate sessions (device flow) without copying host files into pods

### Goal

Provide a GitOps-owned service that:

- manages a pool of **accounts** and **sessions** (auth.json instances) for Codex/ChatGPT auth
- allocates sessions to consumers via a **lease** so each `auth.json` has a **single writer** at a time
- publishes **account depletion/rate-limit status** for selection and observability
- supports human interactive device authorization to mint new sessions

### Non-goals

- Increasing capacity beyond account plan limits (sessions do not increase rate limits).
- Patching Kubernetes `Secret/*` as the authoritative session store (ExternalSecrets will overwrite).
- Letting two consumers concurrently write the same session’s `auth.json`.
- Providing “default” accounts; selection is explicit or `auto`.

### Implementation status (GitOps)

Current state (implemented in `ameideio/ameide-gitops`):

- Broker application exists and is deployed in dev:
  - App source/image: `images/codex-broker/` (Go)
  - Dev image is digest-pinned (see `ameideio/ameide-gitops/sources/values/env/dev/apps/codex-broker.yaml`)
    - Example (2026-01-16): `ghcr.io/ameideio/codex-broker@sha256:040284aa5ef13b4970ef1305ee0f258f002c443b980c5503522b4e9d8b9da79c`
- Storage model matches the intended architecture:
  - Vault KVv2 holds session material (`auth.json`)
  - Postgres holds non-secret metadata + atomic leases/indexing
- UI exists and is protected by Keycloak OIDC (realm role gate):
  - UI entry: `https://codex-broker.dev.ameide.io/ui/`
  - OIDC callback: `/auth/callback` (PKCE enabled)
  - UI styling: GitHub Primer CSS, with light/dark/auto theme toggle (client-side; stored in localStorage)
  - Rollout verification: `GET https://codex-broker.dev.ameide.io/version` returns JSON build info (version/tag, commit, build time)
- Device-auth session minting is implemented:
  - UI flow “Create session (device login)” starts Codex device auth server-side and shows `verificationUri` + `userCode`
  - On successful device auth completion, the broker stores a new session (Vault + Postgres) automatically
  - API: `POST /v1/admin/sessions/device-auth/start`, `GET /v1/admin/sessions/device-auth/{id}`, `POST /v1/admin/sessions/device-auth/{id}/cancel`
- Lease flow is implemented for in-cluster consumers:
  - Consumers authenticate via Kubernetes `TokenReview` (`CODEX_BROKER_AUTH_MODE=kubernetes`)
  - Lease + materialization API: `POST /v1/leases`, `GET/PUT /v1/leases/{leaseId}/auth.json` (ETag/If-Match CAS), `POST /heartbeat`, `POST /release`
- E2E verification (dev) completed for the core API flow:
  - Create account + session, acquire lease, fetch `auth.json` with ETag, upload updated `auth.json` with `If-Match`, heartbeat, release.

Notes:

- The public Gateway route is currently UI-first (paths like `/ui`, `/auth`, `/healthz`, `/readyz`). `/v1/*` is intended for in-cluster consumers (use service DNS or port-forward for manual API testing).
- Account depletion monitoring/generalized scoring is not implemented yet (this doc’s “Depletion monitoring” section remains the target design).

### Vocabulary (generalized)

- **Account**: who you are logged in as (a Codex/ChatGPT identity). Accounts have shared rate limits/credits.
- **Session**: one OAuth credential set (`auth.json`) with its own refresh-token chain. Many sessions can exist per account.
- **Consumer**: a workspace/pod/job that uses Codex. A consumer must hold exactly one session lease while it is active.
- **Lease**: an exclusive claim on a session (with TTL + heartbeat). One session can have at most one live lease.

### Core invariants

1) **Single writer per session**: only the lease holder may refresh/update that session’s `auth.json`.
2) **Separation of concerns**:
   - depletion/rate limits are **account-level**
   - collision avoidance is **session-level**
3) **No shared auth.json** across concurrent consumers.

### Token lifecycle notes (Codex CLI behavior)

- Codex stores credentials in `$CODEX_HOME/auth.json`; the refresh token is rotating, so “same file shared by multiple runtimes” eventually leads to `refresh_token_reused`.
- Codex refresh is primarily reactive (on auth failures) and also has a proactive refresh heuristic in upstream code (`TOKEN_REFRESH_INTERVAL = 8 days` as of recent `codex` sources).
- Refresh calls `https://auth.openai.com/oauth/token` (upstream supports override via `CODEX_REFRESH_TOKEN_URL_OVERRIDE`).
- The access-token lifetime is not a stable “code constant”; it is determined by the auth server response and/or observed failures, so consumers should treat refresh as “whenever required”.
- Long-running pods are fine as long as they are the **only writer** of their session `auth.json` and can reach the refresh endpoint; the failure mode is multi-writer concurrency, not “pod age”.

### Validation checklist (one account, many sessions)

- Create 2+ device-authorized sessions for the same account and run parallel `codex app-server` for long enough to force refresh; confirm no cross-session invalidation.
- Establish a practical per-account session limit (if any) by minting sessions until failure, then encode that as an operator guideline.

### Architecture (high level)

**Components**

1) **`codex-broker` service**
   - API: session lease allocation, status, and session materialization.
   - UI: human device authorization to create new sessions + admin tables.
   - Storage: Vault KVv2 is authoritative for session material (`auth.json`).
     - Optional: SQL (Postgres) for non-secret indexing/audit/UI pagination (see “Admin UI + indexing”).
   - Observability: Prometheus metrics + optional “status snapshot” publishing (see below).

2) **Consumers (Coder workspaces, task pods, jobs)**
   - On startup, request a session lease from broker (`auto` or explicit account/session).
   - Fetch the leased `auth.json`, copy into a writable `$CODEX_HOME/auth.json`, run Codex normally.
   - Periodically heartbeat the lease.
   - On shutdown, release the lease and (optionally) push back the updated `auth.json`.

**Why Vault, not Kubernetes Secrets**

In this repo, Kubernetes secrets are materialized by ExternalSecrets (Vault/KeyVault → ESO → K8s). A broker that “patches Secrets” would create a second mutation path and will be overwritten.
Vault is the correct authoritative store for session material, and consumers can obtain material either:

- directly from broker over HTTPS (recommended for dynamic session allocation), or
- via a controlled materialization layer that is still Vault-authored (only if we need file mounts).

### Data model (Vault KV)

Paths are illustrative; exact mount/key naming can be chosen to match existing conventions.

**Accounts**

- `secret/codex/accounts/<account_id>/config`
  - metadata (human label, ownership)
  - selection weights/priority (optional)
  - status settings (staleness thresholds)

**Sessions**

- `secret/codex/sessions/<session_id>`
  - `account_id`
  - `auth_json` (string, stored as JSON; never logged)
  - `created_ts`, `last_used_ts`
  - `last_refresh_ts` (optional; derived from auth.json)
  - `auth_sha256` (optional; lets broker/consumers avoid redundant sync)
  - `vault_version` (optional; for API ETags / CAS)
  - `last_auth_upload_ts` (optional; detects “sync lag”)
  - `refresh_fail_count`, `last_fail_reason` (optional; supports auto-quarantine)
  - `last_successful_use_ts` (optional; supports LRU selection and debugging)
  - `state`: `ready|revoked|expired|quarantined`
  - `purpose`: `leaseable|probe` (recommended: reserve 1 probe session per account for depletion polling)

**Leases**

- `secret/codex/leases/<lease_id>`
  - `session_id`
  - `consumer_id` (workspace/job identity)
  - `issued_ts`, `expires_ts`
  - `last_heartbeat_ts`
  - `release_reason` (optional; `normal|timeout|revoke|error`)

### Admin UI + indexing (tables)

We want an operator-friendly UI (tables) for:

- accounts (enable/disable, labels/tags, selection policy)
- sessions (state, last used, health/quarantine)
- leases (who holds what, TTL/heartbeat, revoke)
- account depletion (current snapshot + hourly graphs)

**V1 (Vault-only)**

The broker can serve tables by listing Vault KV prefixes and materializing rows on demand:

- list accounts: `secret/metadata/codex/accounts/`
- list sessions: `secret/metadata/codex/sessions/`
- list leases: `secret/metadata/codex/leases/`

This is acceptable for small-to-medium pools (tens to low hundreds), but it is not a database:

- pagination/sorting is broker-implemented (not pushed down)
- “search by consumer_id” requires scanning or additional indexing docs

**V2 (recommended for a “decent UI”)**

Add a small SQL index for non-secret metadata and audit while keeping secrets in Vault:

- Vault remains authoritative for `auth.json`.
- SQL stores:
  - accounts table (metadata, policy knobs)
  - sessions table (account_id, session_id, state, timestamps; no tokens)
  - leases table (session_id, consumer_id, issued/expires/heartbeat)
  - events/audit table (lease acquired/released, session created/quarantined, refresh failure reasons)

This makes tables fast, supports filtering/sorting, and keeps secrets out of SQL.

**Storage recommendation (ideal)**

- Vault KVv2 for `auth.json` (secrets) + Postgres for leases/indexing/audit (non-secrets) is the most operationally robust combo for “n accounts / n sessions” with a real UI.
- Redis is fine as an optimization (rate-limit cache, request coalescing), but avoid making Redis the only source of truth for leases unless you’re comfortable with its persistence/HA semantics.

**Atomic lease allocation (required for HA)**

Leasing must be atomic; “list then pick then write” is unsafe with multiple broker replicas.

- Recommended (V2): Postgres is authoritative for lease allocation and uses a single transaction, e.g. `SELECT ... FOR UPDATE SKIP LOCKED` on `sessions where state='ready' and purpose='leaseable' and lease_expires_ts < now()` followed by `UPDATE ... set lease_id/consumer_id/expires_ts`.
- Vault-only (V1): store lease state inside the **session record** and acquire with Vault KV CAS on `secret/codex/sessions/<session_id>` (the session doc is the lock). Separate `leases/<lease_id>` records can exist for UI/audit, but correctness must come from CAS on the session doc.

### Broker API (conceptual)

**Auth**

Use Kubernetes service account identity for in-cluster consumers (TokenReview or projected service account JWT with configured audience).
Human UI uses standard SSO for the platform (out of scope for this backlog’s details).

**Endpoints**

- `POST /v1/leases`
  - input: `{ "accountSelector": "auto|<account_id>", "sessionSelector": "auto|<session_id>", "purpose": "workspace|task|job", "ttlSeconds": 3600 }`
  - output: `{ "leaseId": "...", "sessionId": "...", "accountId": "...", "expiresTs": "...", "status": { ...optional account snapshot... } }`

- `POST /v1/leases/{leaseId}/heartbeat`
  - extends TTL; returns current lease state
  - recommended: allow optional `authJsonSha256` + “upload if changed” payload so auth sync is tied to liveness

- `POST /v1/leases/{leaseId}/release`
  - releases lease; optional `finalAuthJsonSha256` for audit

- `GET /v1/leases/{leaseId}/auth.json`
  - returns auth.json for the leased session (only to lease holder)
  - should include an `etag` (or `vaultVersion`) so consumers can do CAS-protected uploads

- `PUT /v1/leases/{leaseId}/auth.json`
  - allows the lease holder to push back an updated auth.json (required for pooled sessions so session reuse stays fresh)
  - requires a precondition (`If-Match` / `baseVaultVersion`) so stale uploads cannot overwrite newer rotated tokens
  - broker validates basic JSON shape and stores to Vault; never logs content

- `GET /v1/accounts/status`
  - returns depletion snapshots per account (see “Depletion monitoring”)

**Admin endpoints (UI)**

The UI should not talk to Vault directly. It uses broker-admin endpoints that expose only non-sensitive data:

- Implemented:
  - `POST /v1/admin/accounts` (create/update account metadata)
  - `POST /v1/admin/sessions` (store a session given `authJson`)
  - `POST /v1/admin/sessions/device-auth/start` (returns `verificationUri` + `userCode`)
  - `GET /v1/admin/sessions/device-auth/{id}` (poll status; includes `sessionId` on completion)
  - `POST /v1/admin/sessions/device-auth/{id}/cancel`
- Still desired (not implemented yet): list/filter admin endpoints, session state transitions/quarantine, lease revoke, and metrics/dashboard helpers.

### Session allocation algorithm

Inputs:

- `accountSelector`: `auto` or explicit `account_id`
- `purpose`: `workspace|task|job` (can influence TTL and selection)
- `status`: account depletion (rate limits), broker health, and session state

Algorithm (recommended):

1) Filter accounts to candidates:
   - enabled
   - `account_status.usable == true` (or `ok == true` if legacy)
2) For each candidate account, filter sessions:
   - `state == ready`
   - not currently leased
3) Choose:
   - prefer account with highest `score` (least depleted)
   - within account, prefer least-recently-used session (spread refresh churn)
4) Create lease with TTL and return.

### Depletion monitoring (generalized)

Reuse the `codex-monitor` concept, but generalize from fixed slots to dynamic accounts:

- Broker periodically polls `codex app-server` `account/rateLimits/read` for each **account** using a designated **probe session** (recommended: 1 per account, excluded from leasing), so monitoring continues even when all leaseable sessions are in use.
- Broker computes `score` consistently with `backlog/675-codex-monitor.md` (`min remaining percent` across windows).
- Broker exposes:
  - Prometheus metrics (recommended; enables hourly graphs across accounts)
  - an optional sanitized status JSON per account for consumers that need a fast preflight

Important: depletion is account-level; do not attempt to treat sessions as independent capacity.

#### Hourly depletion graphs (across accounts)

Do not store time series in Vault. Use Prometheus as the historical store (same pattern as `codex-monitor`).

Broker should export per-account gauges with stable labels:

- `codex_account_usable{account_id}`
- `codex_account_depleted{account_id}`
- `codex_account_score{account_id}` (0–100, higher is better)
- `codex_used_percent{account_id,window}`
- `codex_remaining_percent{account_id,window}`
- `codex_reset_ts_seconds{account_id,window}`
- `codex_monitor_ok{account_id}`
- `codex_monitor_last_ok_ts_seconds{account_id}`

Grafana can graph these directly with `min interval = 1h`, or we can add recording rules to downsample for cheaper queries, e.g.:

- `codex_account_score_hourly = min_over_time(codex_account_score[1h])`
- `codex_remaining_percent_hourly = min_over_time(codex_remaining_percent[1h])`
- `codex_used_percent_hourly = max_over_time(codex_used_percent[1h])`

### Consumer integration (Coder workspaces)

Current Coder templates choose among `codex-account-status-{0..2}` and copy `/var/run/ameide/codex-auth-rotating-<slot>/auth.json` to `$HOME/.codex/auth.json`.

With broker:

- `CODEX_ACCOUNT_SLOT` becomes either:
  - `codex_account=auto|<account_id>` (preferred naming), or
  - keep existing `CODEX_ACCOUNT_SLOT` for backwards compatibility during migration.
- Startup script flow:
  1) obtain lease from broker
  2) fetch `auth.json` for lease
  3) write to `$HOME/.codex/auth.json` (writable) and continue
  4) run heartbeat loop while workspace is alive

#### Coder consumption details

**Identity**

- Workspace pods authenticate to broker using their Kubernetes service account (projected JWT) and a broker-configured audience.
- Broker authorizes by namespace/labels (e.g. “coder workspace namespaces”), and ties `consumer_id` to immutable workspace identity (namespace + pod name + Coder agent id).

**Parameters / env**

- `CODEX_BROKER_URL` (in-cluster service DNS)
- `CODEX_ACCOUNT` = `auto|<account_id>` (preferred) or continue supporting `CODEX_ACCOUNT_SLOT` during migration
- `CODEX_LEASE_TTL_SECONDS` (e.g. 3600) and `CODEX_LEASE_HEARTBEAT_SECONDS` (e.g. 30–60)

**Materialization**

- Fetch `auth.json` over HTTPS from broker and write it to a writable `$CODEX_HOME/auth.json` (typically `$HOME/.codex/auth.json`).
- Codex refresh updates the file in place (normal Codex behavior).

**Session write-back (recommended)**

To make session reuse safe, pooled sessions should be synced back **periodically**, not only on shutdown:

- upload on every heartbeat or every N minutes
- upload only when `sha256(auth.json)` changes (avoid churn)
- use CAS (`etag`/`vaultVersion`) so uploads cannot regress the session to an older refresh-token chain

#### Che consumption details

Che workspaces and tasks are Kubernetes pods, so the integration is the same lease/materialize/heartbeat pattern, but the hook points differ:

- **Che workspace**: acquire lease and materialize `auth.json` in the workspace container entrypoint (before starting IDE processes).
- **Che task/job**: acquire lease and materialize `auth.json` in an init step (or pre-run), optionally omit heartbeat for short tasks if TTL safely covers execution, then release on completion.

Che tasks should use shorter TTL defaults and must not share a session concurrently; session pools exist to enable parallelism without `refresh_token_reused`.

### Consumer fail-closed requirement (lease safety)

Lease TTL + heartbeat only protects exclusivity if consumers stop using the session when they cannot renew the lease.

- Heartbeat every 30–60s; TTL 3–5 minutes (tunable per purpose).
- If the consumer cannot renew after a small grace window (e.g. 2–3 missed heartbeats), it must fail closed: stop/kill Codex usage for that session (otherwise the broker may reassign it and create a double-writer).

### Security / guardrails

- Never log `auth.json` (or any token fields).
- Broker must enforce:
  - lease-holder-only access to session material
  - lease TTL + heartbeat expiry
  - optional quarantine if repeated refresh failures occur (revoked/expired/reused)
- Vault policies:
  - broker has read/write for `secret/codex/**`
  - consumers have no direct Vault access (recommended); they call broker.

### Failure modes and expected behavior

- **Stale/invalid session**: lease allocation skips it; UI prompts re-auth to mint a new session.
- **Consumer crashes**: lease expires by TTL; session becomes available again.
- **Refresh-token collisions**: only possible if the same session is double-assigned; prevented by leases.
- **Account depleted**: broker returns “no usable account” (or selects a different account) based on status.
- **Broker unavailable / network partition**: consumers fail closed when they cannot renew; broker should avoid allocating new leases while unhealthy.
- **All sessions leased**: broker returns a clear “no available sessions” response (HTTP 429 + `Retry-After` is reasonable).
- **Auth sync CAS conflict**: consumer re-fetches latest auth and retries upload only if its local auth differs (or skips if unchanged).

### Definition of done

- Broker deployed GitOps-first (ArgoCD) and is reachable from workspace namespaces.
- Human UI can mint sessions for an account via device auth without leaking tokens.
- Consumers can obtain a lease, fetch auth.json, run Codex, heartbeat, and release.
- Broker publishes account status (`ok/usable/depleted/score/windows`) generalized to `account_id` and not limited to 0/1/2.
- Parallel consumers do not hit `refresh_token_reused` provided they are on distinct sessions.
- Validate “one account, many sessions” in practice: 2+ sessions under the same account run concurrently long enough to require refresh, without invalidating each other.
- Admin UI supports:
  - accounts table + create/enable/disable/delete
  - sessions table + device-auth creation + void/delete
  - leases table + revoke/delete
  - account depletion dashboard with hourly graphs via Prometheus/Grafana

DoD status (dev):

- Done: broker deployed; lease API (including auth.json ETag/If-Match CAS) verified end-to-end; UI + Keycloak OIDC + device-auth session minting implemented; basic admin destructive actions (delete/void/revoke).
- Pending: generalized depletion monitoring/score/windows; practical “one account, many sessions” long-run validation against real token refresh; quarantine flows; dashboards.
