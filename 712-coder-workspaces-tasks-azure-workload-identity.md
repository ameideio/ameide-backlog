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
- `coder` (Coder CLI) via a scoped token injected from secrets (dev-only)
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

- GitOps materializes `Secret/coder-workspaces-azure-wi` in the environment namespace (e.g., `ameide-dev`) containing:
  - `AZURE_CLIENT_IDS_JSON` (JSON array of managed identity client ids; sharded pool)
  - `AZURE_TENANT_ID`
- Coder templates read this secret and configure:
  - `ServiceAccount` annotation `azure.workload.identity/client-id=<selected-client-id>`
  - Pod label `azure.workload.identity/use=true` (required for the webhook to inject token + env)

Shard selection is deterministic:

- `shardIndex = crc32(<workspace-namespace>) % len(clientIds)`
- `clientId = clientIds[shardIndex]`

### 3.3 Azure federated credentials lifecycle

Workspaces/tasks use **dynamic namespaces** (e.g., `ameide-ws-<id>`), so federated credentials must be reconciled for:

- issuer: AKS OIDC issuer URL
- subject: `system:serviceaccount:<workspace-namespace>:workspace`

Preferred mutation path:

- a dedicated CI workflow reconciles federated credentials for active workspace/task namespaces (create missing; optionally prune stale).

## 4) Evidence / “Done means”

From a freshly created workspace on the active template version:

- `gh auth status -h github.com` succeeds without manual login
- `az account show` succeeds without SP client secret injection (Workload Identity)
- `coder whoami` succeeds without interactive `coder login`
- `codex --version` works and uses the seeded slot auth

## 5) Cross references

- `backlog/626-ameide-devcontainer-service-coder.md` (workspace tool auth + persistence model)
- `backlog/654-agentic-coding-cli-surface-coder.md` (platform smoke + workspace/toolchain contracts)
- `backlog/651-agentic-coding-ameide-coding-agent.md` (Coder tasks execution plane)
- `backlog/652-agentic-coding-dev-workspace-coder.md` (workspace profile + storage contract)
- `backlog/438-cert-manager-dns01-azure-workload-identity.md` (Workload Identity patterns/rollout gotchas)
- `backlog/613-codex-auth-json-secret.md` + `backlog/675-codex-refresher.md` (Codex auth distribution model)
