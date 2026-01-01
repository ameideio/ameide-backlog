---
title: 613 – Codex `auth.json` as a Managed Secret (Local + Azure)
status: draft
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
  - Local enabled via `sources/values/env/local/foundation/foundation-codex-auth.yaml`
  - Other envs can enable by adding `codexAuth.enabled: true` in their env overlay.

It creates:
- `ExternalSecret/codex-auth-sync` (Vault → K8s)
- `Secret/codex-auth` with a single key `auth.json` (decoded from base64)

## Vault bootstrap sourcing (implemented)

`foundation-vault-bootstrap` now supports **optional secrets** that are synced into Vault when present in either local or Azure sources:

- Configuration: `sources/values/_shared/foundation/foundation-vault-bootstrap.yaml`
  - `secrets.optionalSecrets: ["codex-auth-json-b64"]`

Behavior:
- Local: if `vault-bootstrap-local-secrets` contains `codex-auth-json-b64`, bootstrap writes it to Vault.
- Azure: if Azure Key Vault contains `env-codex-auth-json-b64` (or the configured prefix), bootstrap writes it to Vault.
- If missing in both sources, it is skipped (not treated as required).

## Local workflow (implemented)

### Files

- `infra/scripts/export-codex-auth-env.sh`: writes `CODEX_AUTH_JSON_B64` into `.env` or `.env.local`
- `infra/scripts/seed-codex-auth-local.sh`: local helper that:
  1) exports `auth.json` into the env file
  2) seeds `Secret/vault-bootstrap-local-secrets` from `.env/.env.local`
  3) waits for GitOps-owned `ExternalSecret/codex-auth-sync` to become Ready and `Secret/codex-auth` to appear

### Commands

1) Login once:
   - `codex login --device-auth`

2) Export to `.env`:
   - `infra/scripts/export-codex-auth-env.sh --file .env`

3) Seed K8s + Vault + ExternalSecrets:
   - `infra/scripts/seed-codex-auth-local.sh --namespace ameide-local --env-file .env`

### Verification

Safe checks (do not print the secret content):

- `kubectl -n ameide-local get externalsecret codex-auth-sync`
- `kubectl -n ameide-local get secret codex-auth`

## Azure/AKS workflow (implemented path; enable per env)

Target state: treat `auth.json` like other external/third-party secrets.

1) Store `env-codex-auth-json-b64` in **Azure Key Vault** (value = base64(auth.json)).
2) `foundation-vault-bootstrap` copies it from AKV into Vault KV `secret/codex-auth-json-b64` (optional secret).
3) Enable the GitOps component (`codexAuth.enabled: true`) in the target environment to materialize:
   - `ExternalSecret/codex-auth-sync` → `Secret/codex-auth` (with `auth.json`)

## Security considerations

- `auth.json` is effectively a portable credential; treat it like a high-privilege session token.
- Prefer a dedicated “automation” ChatGPT/Codex identity and restrict access to only the workloads/namespaces that need it.
- Do not print secrets in logs (`kubectl get secret -o yaml` leaks values).
- Rotation plan: re-run device auth → re-export → update source-of-truth → allow ExternalSecrets refresh.

## Smoke test (implemented)

`platform-secrets-smoke` includes an optional Codex auth CLI test:

- Values: `sources/values/_shared/platform/platform-secrets-smoke.yaml`
  - `codexAuthSmoke.enabled` (default false; enabled in local via `sources/values/env/local/platform/platform-secrets-smoke.yaml`)
- Test name: `codex-auth-cli-smoke`

It:
- waits for `ExternalSecret/codex-auth-sync` Ready
- downloads pinned `codex-cli` (`0.57.0` by default)
- runs `codex login status` using the materialized `auth.json` (does not print credentials)

Manual runner (useful for local clusters):
- `infra/scripts/codex-auth-smoke.sh --namespace ameide-local`

## Open questions / follow-ups

- Which workloads actually require Codex CLI ChatGPT login (vs API key)?
- Do we want a `PushSecret` mirror (K8s → Vault) for this secret, or keep Vault as the only distribution layer?
- If we standardize the Azure path, should we add `codex-auth-json-b64` to the Azure `env_secrets` documentation/runbooks so it is reconciled like other secrets?

## Acceptance criteria

- Local: `infra/scripts/seed-codex-auth-local.sh` results in `ExternalSecret/codex-auth-sync` Ready and `Secret/codex-auth` present in `ameide-local`.
- Azure: providing `env-codex-auth-json-b64` in AKV results in the same `Secret/codex-auth` in the target namespace once the component is enabled, without manual kubectl steps.
- No secret content is committed to git, and operational docs warn against printing secrets.
