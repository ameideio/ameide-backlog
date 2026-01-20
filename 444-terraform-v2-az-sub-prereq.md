# 444 – Terraform Infrastructure (v2) – Azure Subscription Prerequisites (Runbook)

**Created**: 2026-01-10  
**Status**: Draft / runbook

This document lists the **manual, one-time prerequisites** for deploying the Ameide AKS environment via **GitHub CI** into a **new Azure subscription**, while keeping the **parent DNS zone `ameide.io`** in the existing subscription:

- **Parent DNS subscription (kept)**: `92f1f4d0-00d9-4ffb-bce3-f2c3be49d748`
- **Target AKS subscription (new)**: `AZURE_SUBSCRIPTION_ID` (GitHub Actions variable)

The intent is that **Terraform owns infra** (cluster, identities, Key Vault, storage, etc.) and **CI owns bootstrap + verification**, while we keep certain “account bootstrap” items manual because they are cross-cutting or sensitive.

---

## Scope and Assumptions

- The AKS deployment is driven by GitHub Actions workflows in:
  - `.github/workflows/terraform-azure-apply.yaml`
  - `.github/workflows/terraform-azure-plan.yaml`
  - `.github/workflows/terraform-azure-destroy.yaml`
- Terraform workspace: `infra/terraform/azure`
- Argo CD uses **multi-source** Applications and pulls CI-generated runtime facts from a **dedicated repo**:
  - `https://github.com/ameideio/ameide-runtime-facts.git` (generated artifacts; CI-only writes)
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
- `DNS_PARENT_ZONE_SUBSCRIPTION_ID` = `92f1f4d0-00d9-4ffb-bce3-f2c3be49d748`
- `DNS_PARENT_ZONE_RESOURCE_GROUP` = the RG that contains `ameide.io` **in the parent DNS subscription** (currently `Ameide`)
- `TF_ENV_SECRETS_KEYVAULT_NAME` = name of the **CI secrets source** Key Vault (see below)

Notes:
- `DNS_PARENT_ZONE_SUBSCRIPTION_ID` is required for the `ameide.io` DNS-01 / AzureDNS flow; charts intentionally fail fast if it’s missing to avoid silently targeting the wrong subscription.

---

## 1) Entra / GitHub OIDC Bootstrap (Manual)

### 1.1 Ensure the GitHub OIDC app exists

- Ensure the Entra application/service principal referenced by `AZURE_CLIENT_ID` exists in `AZURE_TENANT_ID`.
- Ensure the app has a **federated credential** for GitHub Actions OIDC for this repo (subject typically ties to repo + environment/branch).

### 1.2 Grant the CI identity sufficient Azure RBAC in the new subscription

Terraform needs to create:
- AKS, networking, storage, identities, Key Vault, role assignments.

**Recommended minimal roles (practical + “no drift”):**
- On the **target AKS subscription** (or the target RG scope if you keep everything in one RG):
  - `Contributor`
  - `User Access Administrator` (required because Terraform creates role assignments)
- On the **parent DNS zone** (`ameide.io`) in the **old subscription**:
  - `DNS Zone Contributor` (zone scope)
  - `User Access Administrator` (zone scope; required because Terraform creates DNS role assignments for managed identities)
- On the **Terraform state backend** (Azure Storage):
  - `Storage Blob Data Contributor` (data plane) on the storage account or tfstate container
- On the **CI secrets source** Key Vault:
  - `Key Vault Secrets User` (read secrets)

Notes:
- Assigning `Owner` works, but it is broader than needed and not recommended as the steady state.
- Terraform also grants the current Terraform runner identity `Key Vault Secrets Officer` on the **cluster** Key Vault it creates (so CI can seed secrets post-apply).

### 1.3 AKS API access model (CI bootstrap)

- CI uses `az aks get-credentials` **without** `--admin` and authenticates via Entra ID (`kubelogin`).
- Local (admin) accounts are disabled by default in the AKS module, so “admin kubeconfig” is not relied on.
- Terraform grants the Terraform runner identity `Azure Kubernetes Service RBAC Cluster Admin` on the cluster (so the CI identity can bootstrap Argo CD).

---

## 2) Azure Resource Provider Registration (Manual)

Azure subscriptions must have resource providers registered before resources can be created. Terraform is configured to register core providers in the **new** subscription as needed; if provider registration is restricted in your tenant, pre-register the required providers before running CI apply.

Practical minimum set for this repo’s Azure footprint:
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
- Workflows ensure the backend storage account/container exists using **Entra ID / OIDC auth only**.
- Workflows **do not** use (or fall back to) Storage Account access keys.

Required permissions for the CI identity:
- Management plane: create storage account in the backend RG (e.g., `Contributor`)
- Data plane: create/read blobs and acquire leases (locking) in the tfstate container:
  - `Storage Blob Data Contributor` on the storage account or on the specific container

Optional hardening after first successful apply:
- Disable Shared Key authorization on the backend storage account (Entra-only).

If you want strict separation and predictability:
- Pre-create the backend RG + Storage Account + container manually once.
- Grant the CI identity access at that scope.

---

## 5) DNS Model When `ameide.io` Stays in the Old Subscription (Manual)

### The key constraint
The **parent zone** `ameide.io` stays in the old subscription, but the AKS cluster and child zones can live in the new subscription. For this to be predictable (“no drift”), CI must be able to update DNS **cross-subscription**.

This affects:
- `argocd.ameide.io` A record updates (cluster-level)
- NS delegation records for `dev.ameide.io` and `staging.ameide.io` if those zones move

### Option A (default/recommended): CI-managed cross-subscription DNS (implemented)
Prerequisites:
- `DNS_PARENT_ZONE_SUBSCRIPTION_ID` points at the subscription that contains `ameide.io`.
- CI identity has `DNS Zone Contributor` on the `ameide.io` zone scope in that subscription.

Behavior:
- Terraform uses a second `azurerm` provider configuration (pointed at `DNS_PARENT_ZONE_SUBSCRIPTION_ID`) for:
  - reading the parent zone
  - creating NS delegation records for child zones
  - creating/updating `argocd.ameide.io`
  - creating/updating apex/wildcard records for production (`ameide.io`, `*.ameide.io`)

### Option B (break-glass): manual DNS edits
If CI cannot be granted access to the old subscription, you must manage these manually each time you redeploy:
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

Instead, a “seed/source” Key Vault (`TF_ENV_SECRETS_KEYVAULT_NAME`) is treated as the root-of-truth for secret *values* (seeded out-of-band).

GitOps then performs the **seed → cluster** copy **in-cluster** (ArgoCD-managed), keeping orchestration out of GitHub Actions.

### 6.1 Create a globally unique Key Vault name
Key Vault names are global. `ameide-ci-secrets` may already be taken.

- Create a new Key Vault in the **new subscription** with a unique name (example pattern):
  - `ameidecisecrets<hash-or-suffix>`
- Enable RBAC authorization.
- Grant the CI principal `Key Vault Secrets User` on this vault (read).

Notes:
- The **cluster** Key Vault is created by Terraform.
- Terraform grants the in-cluster `vault-bootstrap` managed identity `Key Vault Secrets Officer` on both:
  - the seed/source vault (read/write for controlled automation like token rotation)
  - the cluster vault (write for seed sync; read for vault-bootstrap)

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
- `coder-github-oauth-client-id` (Coder GitHub External Auth OAuth app)
- `coder-github-oauth-client-secret` (Coder GitHub External Auth OAuth app)
- `coder-bootstrap-admin-username` (seed initial Coder owner; skip `/setup`)
- `coder-bootstrap-admin-email` (seed initial Coder owner; skip `/setup`)
- `coder-bootstrap-admin-password` (seed initial Coder owner; skip `/setup`)
- `coder-bootstrap-token` (one-time seed; used by in-cluster rotator to mint `coder-cli-session-token`)
- `coder-cli-session-token` (managed by the in-cluster rotator; required for deterministic `coder` CLI in workspaces)
- `codex-auth-json-b64-0` (base64(auth.json) for Codex slot 0)
- `codex-auth-json-b64-1` (base64(auth.json) for Codex slot 1)
- `codex-auth-json-b64-2` (base64(auth.json) for Codex slot 2)
- `backstage-gitlab-token` (GitLab API token for Backstage integration; mapped into Vault)

---

## 7) Runtime Facts Repo (Manual)

Argo CD cannot reference multiple revisions of the **same** Git repository in a single multi-source Application, so runtime facts must live in a separate repo.

Create/ensure:
- GitHub repo `ameideio/ameide-runtime-facts` exists (private is fine).
- The token used for Argo CD repo access (same as `ghcr-token` today) has read access to that repo.

Notes:
- CI publishes runtime facts into `runtime-facts/` in that repo on each successful apply.
- Humans should never edit this repo; treat it as a generated artifact output.

Then update GitHub variable:
- `TF_ENV_SECRETS_KEYVAULT_NAME` → the created vault name

---

## 8) Global Name Collision Checks (Manual)

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

## 9) Run CI Deployment

Once prerequisites are satisfied:

1. Run `.github/workflows/terraform-azure-plan.yaml` (optional)
2. Run `.github/workflows/terraform-azure-apply.yaml` with:
   - `confirm=apply-azure`

The apply workflow is considered successful only if the **verifiers pass** (TLS/443 + ArgoCD SSO + Platform SSO). There is intentionally no “skip SSO” option.

---

## 10) Post-Deploy Verification (What “Done” Means)

CI apply is considered successful only if:
- Terraform apply succeeds
- ArgoCD is installed and synced (root ApplicationSets applied)
- Public IPs are associated to LoadBalancer Services
- TLS certificates reach `Ready`
- Envoy/edge listeners expose `443`
- SSO verifiers pass

---

## Appendix: Runtime Facts Boundary (Generated, Non-Secret)

- `runtime-facts` is a CI-generated branch, regenerated from Terraform outputs after each apply.
- Treat it as an artifact: never hand-edit it, and keep it limited to non-secret “routing facts” (IPs, client IDs, resource IDs, subscription IDs).
