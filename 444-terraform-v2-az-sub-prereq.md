# 444 – Terraform Infrastructure (v2) – Azure Subscription Prerequisites (Runbook)

**Created**: 2026-01-10  
**Status**: Draft / runbook

This document lists the **manual, one-time prerequisites** for deploying the Ameide AKS environment via **GitHub CI** into a **new Azure subscription**, while keeping the **parent DNS zone `ameide.io`** in the existing subscription:

- **Parent DNS subscription (kept)**: `92f1f4d0-00d9-4ffb-bce3-f2c3be49d748`
- **Target AKS subscription (new)**: `AZURE_SUBSCRIPTION_ID` (GitHub Actions variable; change as needed)

The intent is that **Terraform owns infra** (cluster, identities, Key Vault, storage, etc.) and **CI owns bootstrap + verification**, while we keep certain “account bootstrap” items manual because they are cross-cutting or sensitive.

---

## Scope and Assumptions

- The AKS deployment is driven by GitHub Actions workflows in:
  - `.github/workflows/terraform-azure-apply.yaml`
  - `.github/workflows/terraform-azure-plan.yaml`
  - `.github/workflows/terraform-azure-destroy.yaml`
- Terraform workspace: `infra/terraform/azure`
- Cluster model (today): **one AKS cluster** hosting namespaces `ameide-dev`, `ameide-staging`, `ameide-prod`.
  - If you intend to keep `prod` in the old subscription, the code needs to be split (this runbook assumes the cluster is in the new subscription).

---

## Inputs (GitHub Variables)

These must be set in GitHub repo variables (environment `azure` if you use environments):

- `AZURE_CLIENT_ID` (Entra app / service principal client ID used by GitHub OIDC)
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID` (the **new** subscription where AKS will be deployed)
- `AKS_LINUX_ADMIN_PUBLIC_KEY`
- `DNS_PARENT_ZONE_NAME` = `ameide.io`
- `DNS_PARENT_ZONE_RESOURCE_GROUP` = the RG that contains `ameide.io` **in the parent DNS subscription** (currently `Ameide`)
- `TF_ENV_SECRETS_KEYVAULT_NAME` = name of the **CI secrets source** Key Vault (see below)

---

## 1) Entra / GitHub OIDC Bootstrap (Manual)

### 1.1 Ensure the GitHub OIDC app exists

- Ensure the Entra application/service principal referenced by `AZURE_CLIENT_ID` exists in `AZURE_TENANT_ID`.
- Ensure the app has a **federated credential** for GitHub Actions OIDC for this repo (subject typically ties to repo + environment/branch).

### 1.2 Grant the CI identity sufficient Azure RBAC in the new subscription

Terraform needs to create:
- AKS, networking, storage, identities, Key Vault, role assignments.

**Recommended** (simplest):
- Assign **Owner** on the new subscription scope to the CI principal.

Alternative (more complex):
- `Contributor` + `User Access Administrator` at required scopes (to create role assignments).

---

## 2) Azure Resource Provider Registration (Manual)

Because Terraform is configured not to auto-register providers (`resource_provider_registrations = "none"`), the new subscription must have required providers registered.

Minimum set to register:
- `Microsoft.ContainerService`
- `Microsoft.Compute`
- `Microsoft.Network`
- `Microsoft.Storage`
- `Microsoft.KeyVault`
- `Microsoft.ManagedIdentity`
- `Microsoft.Authorization`
- `Microsoft.OperationalInsights`
- `Microsoft.Insights`

Notes:
- Registration can take minutes. Ensure `registrationState == Registered` before running CI apply.

---

## 3) Resource Groups (Manual)

Terraform expects an existing RG (it references it as data), and CI workflows also create/read backend resources.

Create/ensure:
- `Ameide` (location `westeurope`) – main infra RG for AKS resources
- Optional/if used: `ameide-tfstate` (location `westeurope`) – dedicated TF state RG

---

## 4) Terraform State Backend (Manual vs CI)

Goal: remote state + locking in Azure Storage.

### Recommended target state
- A **dedicated** RG like `ameide-tfstate` in the target subscription.
- A dedicated Storage Account + container `tfstate`.

### Current workflow behavior (implementation detail)
- Workflows attempt to ensure the backend storage account/container exists, but **this requires**:
  - permission to create/update Storage Account in the chosen backend RG
  - permission to create blob containers

If you want strict separation and predictability:
- Pre-create the backend RG + Storage Account + container manually once.
- Grant the CI identity access at that scope.

---

## 5) DNS Model When `ameide.io` Stays in the Old Subscription (Manual)

### The key constraint
Terraform in the **new subscription** cannot safely manage the **parent zone** `ameide.io` unless the CI identity also has access to the old subscription and Terraform is configured for cross-subscription DNS operations.

This affects:
- `argocd.ameide.io` A record updates (cluster-level)
- NS delegation records for `dev.ameide.io` and `staging.ameide.io` if those zones move

### Option A (recommended): make DNS updates CI-managed cross-subscription
Manual prerequisites:
- Grant the CI principal at least **DNS Zone Contributor** on the parent DNS zone scope (old subscription).

Terraform/code prerequisite (future work):
- Add a second `azurerm` provider alias for the DNS subscription and use it for:
  - `data.azurerm_dns_zone.parent`
  - parent zone A records (argocd)
  - NS delegation records

### Option B (fastest now): keep DNS operations manual
Manual steps each time you deploy a new cluster/public IP:
1. In the **old subscription** (parent zone), update:
   - `argocd.ameide.io` → the new cluster’s ArgoCD public IP
2. If moving `dev.ameide.io` and `staging.ameide.io` zones to the new subscription:
   - Create child zones in the new subscription (Terraform can do this).
   - In the old subscription, create/update the NS delegation records:
     - `dev` NS → name servers from the new `dev.ameide.io` zone
     - `staging` NS → name servers from the new `staging.ameide.io` zone

### Important: CI verifiers assume DNS works
The Azure apply workflow includes TLS/443/SSO verifiers. They will fail until:
- the parent record for `argocd.ameide.io` points at the new cluster
- delegated zones (dev/staging) resolve correctly

---

## 6) CI Secrets Source Key Vault (Manual)

We intentionally do **not** put secret values into Terraform inputs, because they land in `tfstate`.

Instead, CI reads secret values from a “source” Key Vault (`TF_ENV_SECRETS_KEYVAULT_NAME`) and then **copies** them into the cluster Key Vault created by Terraform.

### 6.1 Create a globally unique Key Vault name
Key Vault names are global. `ameide-ci-secrets` may already be taken.

- Create a new Key Vault in the **new subscription** with a unique name (example pattern):
  - `ameidecisecrets<hash-or-suffix>`
- Enable RBAC authorization.
- Grant the CI principal `Key Vault Secrets User` on this vault.

### 6.2 Populate required secret names (values are org-specific)
Create secrets (names must match exactly):
- `ghcr-username`
- `ghcr-token`
- `pypi-token`
- `npm-token`
- `buf-token`
- `langsmith-api-key`
- `repository-database-url`
- `database-ssl`
- `database-pool-max`
- `database-pool-idle-timeout-ms`
- `database-pool-connection-timeout-ms`
- `go-proto-version`
- `go-sdk-version`

Then update GitHub variable:
- `TF_ENV_SECRETS_KEYVAULT_NAME` → the created vault name

---

## 7) Global Name Collision Checks (Manual)

Before running CI apply in a new subscription, check that globally-scoped names are available:

- Cluster Key Vault name derived from `TF_VAR_name_prefix`:
  - default in code: `ameidekv`
  - if unavailable or soft-deleted with purge protection, pick a new `TF_VAR_name_prefix`.
- Storage accounts:
  - backup accounts derived from `name_prefix` + env, e.g. `ameidedevbackup`
  - if unavailable, change `TF_VAR_name_prefix`

Also check for Key Vault soft-delete:
- If a vault name is “taken” due to soft delete, you may need to **purge** it (requires privileged access) or choose a new name.

---

## 8) Run CI Deployment

Once prerequisites are satisfied:

1. Run `.github/workflows/terraform-azure-plan.yaml` (optional)
2. Run `.github/workflows/terraform-azure-apply.yaml` with:
   - `confirm=apply-azure`
   - `verify_sso=true` (only once DNS and secrets are in place)

If you are using Option B (manual DNS), set `verify_sso=false` until DNS is updated, then re-run with `verify_sso=true`.

---

## 9) Post-Deploy Verification (What “Done” Means)

CI apply is considered successful only if:
- Terraform apply succeeds
- ArgoCD is installed and synced (root ApplicationSets applied)
- Public IPs are associated to LoadBalancer Services
- TLS certificates reach `Ready`
- Envoy/edge listeners expose `443`
- SSO verifiers pass

