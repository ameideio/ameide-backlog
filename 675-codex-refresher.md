## Codex auth refresher (GitOps-owned; multi-account; daily)

### Problem

Codex CLI ChatGPT auth uses `$CODEX_HOME/auth.json`. Refreshing the token can rotate the refresh token; if we refresh repeatedly from a stale file, we can hit `refresh_token_reused`.

In this repo, `Secret/codex-auth` is **materialized by ExternalSecrets** from Vault/KeyVault. That means “patching the Secret” is the wrong long-term write target (it will be overwritten).

### Design (two secrets)

We keep the existing pipeline **as-is** for initial/manual auth seeding, and introduce a second secret for refreshed credentials:

1) **Seed secret (manual auth source-of-truth)**

- Vault/AKV key: `codex-auth-json-b64` (base64 of `auth.json`)
- `ExternalSecret` → `Secret/codex-auth` (key `auth.json`)
- Humans seed/rotate this manually via device auth + KeyVault, per `backlog/613-codex-auth-json-secret.md`.

2) **Rotating secret (automation-owned refreshed state)**

- Vault key: `codex-auth-json-b64-rotating-<account>` (base64 of refreshed `auth.json`)
- `ExternalSecret` → `Secret/codex-auth-rotating-<account>` (key `auth.json`)
- All automated consumers mount/use the rotating Secret (not the seed).

### Automation

`CronJob/codex-auth-refresher-<account>` runs **daily**, `concurrencyPolicy: Forbid`, and:

1. Copies `/seed/auth.json` into a writable `$CODEX_HOME/auth.json` (`emptyDir`).
   - Prefer `/seed-rotating/auth.json` if present; fall back to `/seed/auth.json`.
2. Spawns `codex app-server` and performs:
   - `initialize` → `initialized`
   - `account/read` with `{"refreshToken": true}` (forces refresh)
   - optional verify: `account/rateLimits/read`
3. Reads the updated `$CODEX_HOME/auth.json` from disk (never prints it).
4. Writes base64(auth.json) into Vault KV at `secret/<rotatingVaultKey>` under field `value`.
5. Exits **non-zero** if refresh or write fails (and must not overwrite the rotating key on failure).

### Multi-account plan (more than one initial auth)

Model Codex accounts explicitly:

- One seed key per account: `codex-auth-json-b64-<account>` (or keep `codex-auth-json-b64` as the default account).
- One rotating key per account: `codex-auth-json-b64-rotating-<account>`.
- One rotating Secret per account: `codex-auth-rotating-<account>`.
- One CronJob per account: `codex-auth-refresher-<account>`.

This isolates failure, credentials, and blast radius per account.

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

- `ExternalSecret` for `codex-auth` (seed) and `codex-auth-rotating-*` (rotating)
- `CronJob/codex-auth-refresher-*` and its `ServiceAccount`
- Vault role/policy created by `foundation-vault-bootstrap`

### Implementation plan (PR breakdown)

1) Add rotating secret materialization:
- Extend `foundation-codex-auth` to optionally create `Secret/codex-auth-rotating-<account>` sourced from Vault key(s).

2) Add Vault role/policy for refresher + optional one-time mirror:
- `foundation-vault-bootstrap` creates `codex-auth-refresher` Vault role/policy.
- Optional: mirror seed key to rotating key when rotating is unset, so consumers can switch immediately.

3) Add `foundation-codex-auth-refresher` CronJob component:
- One CronJob per account, schedule daily, fail-fast.

4) Switch automated consumers to use rotating secret:
- e.g., `workrequests-runner` mounts `codex-auth-rotating-default` in dev.

### Verification (E2E)

- GitOps: ArgoCD sync shows `ExternalSecret` Ready for both seed and rotating secrets.
- CronJob: latest run succeeded; Pod logs show refresh + Vault write succeeded (no secret content printed).
- Consumer: a Codex-consuming job can successfully call `account/rateLimits/read` using the rotating secret.
