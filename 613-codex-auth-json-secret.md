---
title: 613 – Codex `auth.json` as a Managed Secret (Local + Azure)
status: implemented
owners:
  - platform
created: 2026-01-01
---

## Summary

Codex CLI login (including headless device auth) produces a portable `auth.json` credential file (typically `~/.codex/auth.json`). We want a reproducible way to inject that file into Kubernetes workloads without copying host files into pods by hand.

This backlog documents a **local-first** implementation and the intended **Azure/AKS** extension, aligned with the existing secrets pipeline:

`source-of-truth` → Vault bootstrap → Vault KV → ExternalSecrets → Kubernetes Secret → mount into workload

This is now implemented with a single GitOps-owned `ExternalSecret` component and a single vault-bootstrap path that supports both local and Azure sources.

## Context

- Codex CLI auth uses a browser approval flow; on headless machines use device auth:
  - `codex login --device-auth`
  - approve the code in a browser
  - Codex writes `auth.json` under `~/.codex/auth.json` (or `$CODEX_HOME/auth.json`).
- In this repo, external secrets converge via:
  - **Local**: `.env/.env.local` → `Secret/vault-bootstrap-local-secrets` (key `env-secrets.json`) → `foundation-vault-bootstrap` (local mode)
  - **Azure**: Azure Key Vault → `foundation-vault-bootstrap` (azure mode via Workload Identity)
  - Then: Vault KVv2 `secret/<key>` with a `value` field → ExternalSecrets read `property: value`.

Related:
- `backlog/433-codex-cli-057.md` (Codex CLI pin + auth.json sharing policy)
- `backlog/451-secrets-management.md` (secrets architecture)
- `backlog/444-terraform.md` (local terraform + local secret seeding)

## Decision: represent the file as base64

`.env` is line-oriented; raw JSON is error-prone to quote/escape. We store the file as a single-line base64 blob:

- Env var: `CODEX_AUTH_JSON_B64=<base64(auth.json)>`
- Canonical secret key name after sanitization: `codex-auth-json-b64`
  - This becomes a Vault KV key: `secret/codex-auth-json-b64` with field `value=<base64>`

ExternalSecrets should use `decodingStrategy: Base64` to turn that blob back into the literal `auth.json` bytes.

## GitOps ownership (implemented)

The Codex auth materialization is **GitOps-owned** as a dedicated component:

- Component: `environments/_shared/components/foundation/secrets/codex-auth/component.yaml`
- Values: `sources/values/_shared/foundation/foundation-codex-auth.yaml`
  - Dev enabled via `sources/values/env/dev/foundation/foundation-codex-auth.yaml`
  - Other envs can enable by adding `codexAuth.enabled: true` in their env overlay.

It creates:
- `ExternalSecret/codex-auth-sync` (Vault → K8s)
- `Secret/codex-auth` with a single key `auth.json` (decoded from base64)

### Coder workspaces (implemented, dev-only)

Human Coder workspaces run in dynamically created namespaces (`ameide-ws-*`), so they cannot rely on a namespace-scoped `SecretStore` pre-existing.

We support this by:

- `ClusterSecretStore` created by `foundation-vault-secret-store` when `vault.clusterSecretStore.enabled: true` (dev only).
- `ClusterExternalSecret/coder-workspaces-codex-auth` (in `foundation-codex-auth`) that:
  - selects Coder workspace namespaces by labels
  - creates `ExternalSecret/codex-auth-sync` in each workspace namespace
  - materializes `Secret/codex-auth` (key `auth.json`)

## Vault bootstrap sourcing (implemented)

`foundation-vault-bootstrap` copies selected secrets from the upstream secret source into Vault KVv2:

- **Azure**: Azure Key Vault → Vault (via Workload Identity)
- **Local**: `Secret/vault-bootstrap-local-secrets` → Vault (local clusters)

For Azure, the dev cluster treats Codex auth as **required** (to avoid “it works on my machine” drift):

- Config: `sources/values/env/dev/foundation/foundation-vault-bootstrap.yaml`
  - `azure.keyVault.requiredSecrets` includes `codex-auth-json-b64`

This causes vault-bootstrap to:
1) read Key Vault secret `codex-auth-json-b64` (prefix is empty in managed clusters), and
2) write Vault KV key `secret/codex-auth-json-b64` with field `value=<base64>`.

## Local workflow (supported)

Local clusters use the existing “local secrets” path:

1) Login once (writes `~/.codex/auth.json`):
   - `codex login --device-auth`

2) Export the base64 blob into `.env.local` (do not commit):
   - `CODEX_AUTH_JSON_B64=$(base64 -w0 ~/.codex/auth.json)`

3) Seed local bootstrap secrets from `.env/.env.local`:
   - `infra/scripts/seed-local-secrets.sh --namespace ameide-local`

### Verification

Safe checks (do not print the secret content):

- `kubectl -n ameide-local get externalsecret codex-auth-sync`
- `kubectl -n ameide-local get secret codex-auth`

## Azure/AKS workflow (implemented)

Target state: treat `auth.json` like other external/third-party secrets.

1) Store `codex-auth-json-b64` in **Azure Key Vault** (value = base64(auth.json)).
2) `foundation-vault-bootstrap` copies it from AKV into Vault KV `secret/codex-auth-json-b64` (dev requires it).
3) Enable the GitOps component (`codexAuth.enabled: true`) in the target environment to materialize:
   - `ExternalSecret/codex-auth-sync` → `Secret/codex-auth` (with `auth.json`)

Recommended dev flow (KV-only, reproducible):
- Put `CODER_GITHUB_OAUTH_CLIENT_ID`, `CODER_GITHUB_OAUTH_CLIENT_SECRET` in `.env`
- Put `CODEX_AUTH_JSON_B64` in `.env.local`
- Run `infra/scripts/deploy.sh azure-secrets`

## Security considerations

- `auth.json` is effectively a portable credential; treat it like a high-privilege session token.
- Prefer a dedicated “automation” ChatGPT/Codex identity and restrict access to only the workloads/namespaces that need it.
- Do not print secrets in logs (`kubectl get secret -o yaml` leaks values).
- Rotation plan: re-run device auth → re-export → update source-of-truth → allow ExternalSecrets refresh.

## Open questions / follow-ups

- Which workloads actually require Codex CLI ChatGPT login (vs API key)?
- Do we want a `PushSecret` mirror (K8s → Vault) for this secret, or keep Vault as the only distribution layer?
- If we standardize beyond dev, decide whether `codex-auth-json-b64` should be required (fail fast) or optional (best-effort).

## Acceptance criteria

- Local: `infra/scripts/seed-codex-auth-local.sh` results in `ExternalSecret/codex-auth-sync` Ready and `Secret/codex-auth` present in `ameide-local`.
- Azure: providing `env-codex-auth-json-b64` in AKV results in the same `Secret/codex-auth` in the target namespace once the component is enabled, without manual kubectl steps.
- No secret content is committed to git, and operational docs warn against printing secrets.
