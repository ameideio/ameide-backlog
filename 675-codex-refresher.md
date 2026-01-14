## Codex auth refresher (GitOps-owned; slot-based; daily)

### Problem

Codex CLI ChatGPT auth uses `$CODEX_HOME/auth.json`. Refreshing the token can rotate the refresh token; if we refresh repeatedly from a stale file, we can hit `refresh_token_reused`.

In this repo, `Secret/codex-auth-<slot>` and `Secret/codex-auth-rotating-<slot>` are **materialized by ExternalSecrets** from Vault/KeyVault. That means “patching the Secret” is the wrong long-term write target (it will be overwritten).

### Target state (big-bang; no `default`)

We do not use a “default account”. We use **account slots** only:

- `0`, `1`, `2` (extendable later)

Each slot is a full stack of:

- seed auth (manual source-of-truth)
- rotating auth (automation-owned refreshed state)
- refresher CronJob

**Naming (slot = `N`)**

- Seed Vault/AKV key: `codex-auth-json-b64-N`
- Seed K8s Secret: `codex-auth-N` (key: `auth.json`)
- Rotating Vault key: `codex-auth-json-b64-rotating-N`
- Rotating K8s Secret: `codex-auth-rotating-N` (key: `auth.json`)
- Refresher CronJob: `codex-auth-refresher-N`

Big-bang implication: before GitOps applies this, **AKV must already contain all seed keys** (`codex-auth-json-b64-0`, `-1`, `-2`) or bootstrap/external-secrets will fail.

### Design (two secrets per slot)

We keep the existing pipeline **as-is** for initial/manual auth seeding, and introduce a second secret for refreshed credentials:

1) **Seed secret (manual auth source-of-truth)**

- Vault/AKV key: `codex-auth-json-b64-<slot>` (base64 of `auth.json`)
- `ExternalSecret` → `Secret/codex-auth-<slot>` (key `auth.json`)
- Humans seed/rotate this manually via device auth + KeyVault, per `backlog/613-codex-auth-json-secret.md`.

2) **Rotating secret (automation-owned refreshed state)**

- Vault key: `codex-auth-json-b64-rotating-<slot>` (base64 of refreshed `auth.json`)
- `ExternalSecret` → `Secret/codex-auth-rotating-<slot>` (key `auth.json`)
- All automated consumers mount/use the rotating Secret (not the seed).

### Automation

`CronJob/codex-auth-refresher-<slot>` runs **daily**, `concurrencyPolicy: Forbid`, and:

1. Copies `/seed/auth.json` into a writable `$CODEX_HOME/auth.json` (`emptyDir`).
   - Prefer `/seed-rotating/auth.json` if present; fall back to `/seed/auth.json`.
2. Spawns `codex app-server` and performs:
   - `initialize` → `initialized`
   - `account/read` with `{"refreshToken": true}` (forces refresh)
   - optional verify: `account/rateLimits/read`
3. Reads the updated `$CODEX_HOME/auth.json` from disk (never prints it).
4. Writes base64(auth.json) into Vault KV at `secret/<rotatingVaultKey>` under field `value`.
5. Exits **non-zero** if refresh or write fails (and must not overwrite the rotating key on failure).

### Slot model (multi-account)

The slot model is the multi-account model:

- One seed key per slot: `codex-auth-json-b64-<slot>`
- One rotating key per slot: `codex-auth-json-b64-rotating-<slot>`
- One rotating Secret per slot: `codex-auth-rotating-<slot>`
- One CronJob per slot: `codex-auth-refresher-<slot>`

This isolates failure, credentials, and blast radius per slot.

### Image approach (tiny dedicated image)

Use a small dedicated image that contains:

- pinned `codex` CLI (match the platform pin; see `backlog/433-codex-cli-057.md`)
- a small refresher script (Python stdlib only)
- CA certs

### Vault RBAC

The refresher does **not** need Kubernetes RBAC for secrets if secrets are mounted as volumes.
It does need a Vault Kubernetes auth role + policy that grants write access only to:

- `secret/data/codex-auth-json-b64-rotating-*` (create/update/read)

### GitOps ownership

All resources are declared in Git and reconciled by Argo CD:

- `ExternalSecret` for `codex-auth-*` (seed) and `codex-auth-rotating-*` (rotating)
- `CronJob/codex-auth-refresher-*` and its `ServiceAccount`
- Vault role/policy created by `foundation-vault-bootstrap`

### Implementation plan (PR breakdown)

1) Add rotating secret materialization:
- Extend `foundation-codex-auth` to render all slot secrets and rotating secrets.

2) Add Vault role/policy for refresher + optional one-time mirror:
- `foundation-vault-bootstrap` creates `codex-auth-refresher` Vault role/policy.
- Optional: mirror each slot seed key to its rotating key when rotating is unset, so consumers can switch immediately.

3) Add `foundation-codex-auth-refresher` CronJob component:
- One CronJob per slot, schedule daily, fail-fast.

4) Switch automated consumers to use rotating secret:
- all automated consumers mount `codex-auth-rotating-0` (or a slot-selection mechanism when multi-slot is enabled).

### Definition of done (big-bang; slots-only)

**Seeding**

- Azure Key Vault contains:
  - `codex-auth-json-b64-0`, `codex-auth-json-b64-1`, `codex-auth-json-b64-2`
- Vault contains:
  - `secret/codex-auth-json-b64-0..2` (copied from AKV)
  - `secret/codex-auth-json-b64-rotating-0..2` (created by mirror or refresher)

**GitOps / ESO**

- ArgoCD app `foundation-codex-auth` is `Synced/Healthy`.
- In `ameide-dev`, ExternalSecrets are `Ready=True` for:
  - `codex-auth-sync-0..2` → `Secret/codex-auth-0..2`
  - `codex-auth-rotating-sync-0..2` → `Secret/codex-auth-rotating-0..2`
- Workspace fan-out (dev) creates the same `codex-auth-rotating-0..2` secrets in `ameide-ws-*` namespaces.

**Refresher**

- ArgoCD app `foundation-codex-auth-refresher` is `Synced/Healthy`.
- `CronJob/codex-auth-refresher-0..2` exist and have a successful Job run after the last rotation window.
- Refresher logs show refresh + Vault write succeeded without printing secret content.

**Consumer contract**

- All Codex-consuming workloads mount only `codex-auth-rotating-N` (no references to `default`).
