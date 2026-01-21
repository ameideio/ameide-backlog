---
title: "712 – Coder workspaces/tasks: Azure Workload Identity (no SP secret) + default tool auth"
status: draft
owners:
  - platform-devx
  - platform-sre
created: 2026-01-20
suite: "650-agentic-coding"
source_of_truth: false
---

# 712 – Coder workspaces/tasks: Azure Workload Identity (no SP secret) + default tool auth

## 0) Goal

Make **Coder workspaces and Coder tasks** authenticate to Azure via **Azure Workload Identity** (federated token, no client secret), while keeping the “default auth” story deterministic for:

- `gh` (GitHub CLI) via `Secret/gh-auth` (dev-only)
- `az` (Azure CLI) via Workload Identity (dev-only)
- `coder` (Coder CLI) via the AKV → Vault → ESO conveyor, with an in-cluster token rotator (dev-only)
- `codex` (Codex CLI) via slot secrets (dev-only)

## 1) Why

Shared Service Principal client secrets for workspaces/tasks:

- are hard to rotate safely (parallel workspaces)
- tend to drift across workspaces (ephemeral `$HOME` vs persistent `/workspaces`)
- increase blast radius (any workspace compromise leaks a reusable credential)

Workload Identity removes the long-lived secret distribution problem and aligns with existing platform patterns (cert-manager DNS, vault-bootstrap, CNPG backup).

## 2) Non-goals

- Do not change platform-wide DNS / ingress behavior.
- Do not grant broad Azure write permissions to workspaces/tasks by default; start with least-privilege read posture.
- Do not introduce a runtime “break-glass controller” that mutates Azure outside of CI/Terraform without an explicit contract.

## 3) Design (high-level)

### 3.1 Azure identity

- Terraform creates a **pool of user-assigned managed identities** for the “Coder execution plane” (workspaces + tasks).
- Role assignment (initial): `Reader` at the main resource group scope (tighten later as needed).

#### Constraints / limits

Microsoft Entra limits the number of **federated identity credentials** per **user-assigned managed identity**. Because workspace/task subjects are unique per namespace/serviceaccount (`system:serviceaccount:<ns>:workspace`), a single identity does not scale.

Decision for Ameide: **shard across a fixed pool of identities** (“Option B”).

### 3.2 K8s wiring

- GitOps materializes non-secret runtime facts for Coder templates:
  - `Secret/coder-workspaces-azure-wi` in the environment namespace contains:
    - `AZURE_TENANT_ID`
    - `AZURE_CLIENT_IDS_JSON` (JSON array)
  - Coder server exports these as Terraform variables for templates:
    - `TF_VAR_azure_workload_identity_tenant_id`
    - `TF_VAR_azure_workload_identity_client_ids_json`
- Coder templates use these values to configure:
  - `ServiceAccount` annotation `azure.workload.identity/client-id=<selected-client-id>`
  - Pod label `azure.workload.identity/use=true` (required for the webhook to inject token + env)

Shard selection is deterministic:

- `shardIndex = crc32(<workspace-namespace>) % len(clientIds)`
  - `clientId = clientIds[shardIndex]`

### 3.3 Azure federated credentials lifecycle

Workspaces/tasks use **dynamic namespaces** (e.g., `ameide-ws-<id>`), so federated credentials must be reconciled for:

- issuer: AKS OIDC issuer URL
- subject: `system:serviceaccount:<workspace-namespace>:workspace`

Mutation path (dev-only; GitOps-managed):

- A CronJob runs **in-cluster** (ArgoCD-managed) under Azure Workload Identity and reconciles federated identity credentials:
  - create missing
  - prune stale / relocated (prefix-scoped) to avoid hitting the Entra 20-FIC-per-identity limit

Implementation:

- Chart: `sources/charts/foundation/vault-bootstrap`
- CronJob: `vault-bootstrap-*-coder-ws-fic`

### 3.4 Seed Key Vault → cluster Key Vault sync (no GitHub workflows)

We keep a **seed/source Key Vault** as the root-of-truth for secret values (out-of-band seeded), but we no longer rely on GitHub Actions to copy secrets into the cluster Key Vault.

Implementation (dev-only; ArgoCD-managed):

- Chart: `sources/charts/foundation/vault-bootstrap`
- CronJob: `vault-bootstrap-*-keyvault-sync`
- Copies a **curated allowlist** of secret names from seed AKV → cluster AKV.

### 3.5 Coder CLI auth (workspace default; no manual login)

Goal: new workspaces can run `coder whoami` without running `coder login` interactively and without putting any token values into templates or Terraform state.

Implementation (dev-only; GitOps-managed):

1) Human seeds **one** long-lived `coder-bootstrap-token` into the seed Key Vault (once).

2) `foundation-vault-bootstrap` copies it into Vault KV (standard AKV → Vault flow):

- seed AKV → (CronJob sync) → cluster AKV → (vault-bootstrap) → Vault KV key `coder-bootstrap-token`

3) `platform-coder` maintains the derived workspace default token in Vault KV:

- CronJob: `coder-cli-session-token`
- Reads `coder-bootstrap-token` (via `Secret/coder-bootstrap-token`, sourced from Vault)
- Mints/validates `coder-cli-session-token` via the Coder API
- Writes `coder-cli-session-token` into Vault KV (via Vault Kubernetes auth role `coder-token-writer`)

4) ESO fans out into workspaces:

- Vault KV → `ExternalSecret`/`ClusterExternalSecret` → `Secret/coder-cli-auth` (per workspace namespace)

## 4) Evidence / “Done means”

From a freshly created workspace on the active template version:

- `gh auth status -h github.com` succeeds without manual login
- `az account show` succeeds without SP client secret injection (Workload Identity)
- `coder whoami` succeeds without interactive `coder login`
- `codex --version` works and uses the seeded slot auth

## 5) Cross references

- `backlog/713-seeding-contract.md` (seeding contract: failfast + self-heal; applies to workspace auth + Coder /setup)
- `backlog/626-ameide-devcontainer-service-coder.md` (workspace tool auth + persistence model)
- `backlog/654-agentic-coding-cli-surface-coder.md` (platform smoke + workspace/toolchain contracts)
- `backlog/651-agentic-coding-ameide-coding-agent.md` (Coder tasks execution plane)
- `backlog/652-agentic-coding-dev-workspace-coder.md` (workspace profile + storage contract)
- `backlog/438-cert-manager-dns01-azure-workload-identity.md` (Workload Identity patterns/rollout gotchas)
- `backlog/613-codex-auth-json-secret.md` + `backlog/675-codex-refresher.md` (Codex auth distribution model)
