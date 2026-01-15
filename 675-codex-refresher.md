## Codex auth refresher (GitOps-owned; slot-based; daily)

### Status

Legacy (slot-based). New work should target the generalized broker model in `backlog/675-codex-broker.md` (n accounts, n sessions, lease-based allocation). This refresher should only remain while any consumers still depend on per-slot rotating Secrets (`codex-auth-rotating-N`).

### Implementation status (GitOps)

- The broker GitOps scaffolding exists and is deployed in dev with a placeholder image to validate the platform wiring; see `backlog/675-codex-broker.md`.
- The broker model eliminates shared-refresh-token collisions by leasing distinct sessions per consumer; that makes “refreshing a shared rotating slot secret” an anti-pattern in the clean target state.

### Problem

Codex CLI ChatGPT auth uses `$CODEX_HOME/auth.json`. Refreshing the token can rotate the refresh token; if we refresh repeatedly from a stale file, we can hit `refresh_token_reused`.

In this repo, `Secret/codex-auth-<slot>` and `Secret/codex-auth-rotating-<slot>` are **materialized by ExternalSecrets** from Vault/KeyVault. That means “patching the Secret” is the wrong long-term write target (it will be overwritten).

### Target state (no `default`)

We do not use a “default account”. We use **account slots** only:

- `0`, `1`, `2` (extendable later; environments may provision a subset)

This document describes the **slot-based refresher**. In the broker model (`backlog/675-codex-broker.md`), token rotation becomes a **session lifecycle** concern with exclusive leases; the refresher remains useful as a transitional mechanism for the existing slot secrets (`codex-auth-rotating-N`).

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

Before GitOps applies this, **AKV must contain all seed keys for the configured slots**.

- In this repo, slots are configured per environment in `codexAuth.accounts` (GitOps values).
- CI copies the configured seed keys from the “source Key Vault” into the cluster Key Vault.
- Slot `2` is treated as optional in CI to support “2 out of 3” provisioning while we ramp up.

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
   - reference protocol: `backlog/675-codex.md`
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

Important: do not seed the same account into multiple slots. Multiple refreshers for the same underlying account can trigger `refresh_token_reused`.

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

### Definition of done (slots-only)

**Seeding**

- Azure Key Vault contains seed keys for the configured slots (e.g. dev: `codex-auth-json-b64-0`, `codex-auth-json-b64-1`).
- Vault contains:
  - `secret/codex-auth-json-b64-<slot>` (copied from AKV)
  - `secret/codex-auth-json-b64-rotating-<slot>` (created by mirror or refresher)

**GitOps / ESO**

- ArgoCD app `foundation-codex-auth` is `Synced/Healthy`.
- In `ameide-dev`, ExternalSecrets are `Ready=True` for:
  - `codex-auth-sync-<slot>` → `Secret/codex-auth-<slot>`
  - `codex-auth-rotating-sync-<slot>` → `Secret/codex-auth-rotating-<slot>`
- Workspace fan-out (dev) creates the same `codex-auth-rotating-0..2` secrets in `ameide-ws-*` namespaces.

**Refresher**

- ArgoCD app `foundation-codex-auth-refresher` is `Synced/Healthy`.
- `CronJob/codex-auth-refresher-<slot>` exist and have a successful Job run after the last rotation window.
- Refresher logs show refresh + Vault write succeeded without printing secret content.

**Consumer contract**

- All Codex-consuming workloads mount only `codex-auth-rotating-N` (no references to `default`).
