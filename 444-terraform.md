# 444 – Terraform Infrastructure

**Created**: 2025-12-04
**Updated**: 2025-12-11

> **Related documents:**
> - [439-deploy-infrastructure.md](439-deploy-infrastructure.md) – Deployment flow and scripts
> - [443-tenancy-models.md](443-tenancy-models.md) – Multi-tenant architecture
> - [148-azure-managed-offering.md](148-azure-managed-offering.md) – Azure Marketplace (Bicep required)
> - [449-per-environment-infrastructure.md](449-per-environment-infrastructure.md) – Per-environment Azure resources
> - [451-secrets-management.md](451-secrets-management.md) – Secrets flow from .env to Kubernetes

## Overview

Terraform-first infrastructure provisioning for multi-cloud deployment. Bicep retained only for Azure Marketplace SaaS offer compliance.

## Why Terraform + Bicep

| Requirement | Terraform | Bicep |
|-------------|-----------|-------|
| Azure Marketplace SaaS | Can consume existing Managed App, cannot publish definition | Required to author/publish definition |
| Multi-cloud (AWS, GCP) | Native | Azure only |
| Local development | Flexible | Azure-only, not used here |
| State management | Backend needed | None |
| Developer experience | Consistent | Azure-specific |

**Decision**: Terraform as primary IaC, Bicep only for Marketplace.

## Command Interface

```bash
./infra/scripts/deploy.sh [target]

Targets:
  azure            Terraform Azure (default, always renders dev/staging/prod namespaces)
  azure-bicep      Bicep for Azure Marketplace only
  aws              Terraform AWS (future)
  local            k3d + Terraform (implemented)
  bootstrap-azure  ArgoCD bootstrap for an existing Azure cluster
  bootstrap-local  ArgoCD bootstrap for an existing local k3d cluster
```

Targets no longer accept a `-e` flag because a single cluster hosts all namespaces. Use the dedicated `bootstrap-*` targets to run only the ArgoCD step.

`deploy.sh` and `deploy-bicep.sh` both source `infra/scripts/lib/env.sh`, which loads `.env` and `.env.local` via `set -a` before invoking Terraform or Helm. The CLI contract is now:

1. Populate `.env`/`.env.local` with the required API tokens (e.g. `GITHUB_TOKEN`); no manual `export` is necessary.
2. Run `./infra/scripts/deploy.sh <target>` – the helper guarantees child processes inherit those variables, and Terraform’s `data.external.env` shim will fall back to `.env` if the variable is absent from the shell.
3. After apply, Terraform writes the latest outputs to `artifacts/terraform-outputs/<target>.json`, and the local target also starts an ArgoCD port-forward plus prints the credentials banner.

## Architecture

### Directory Structure

```
ameide-gitops/
├── argocd/                        # ArgoCD configuration (shared)
│   ├── applicationsets/           # Root ApplicationSets
│   ├── projects/                  # AppProjects
│   └── repos/                     # Repository credentials
├── bootstrap/                     # ArgoCD bootstrap scripts
│   ├── bootstrap.sh               # Main entry point
│   ├── configs/                   # Environment-specific configs
│   └── lib/                       # Library functions
├── infra/
│   ├── terraform/
│   │   ├── modules/               # Reusable modules
│   │   │   ├── azure-aks/         # AKS cluster
│   │   │   ├── azure-keyvault/    # Key Vault + secrets
│   │   │   ├── azure-dns/         # DNS zones + records
│   │   │   ├── azure-identity/    # Managed identities + federation
│   │   │   ├── k3d-cluster/       # Local k3d cluster (TF-9)
│   │   │   └── argocd-bootstrap/  # ArgoCD install + GitOps (TF-23/24/27-29)
│   │   ├── azure/                 # Azure workspace
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── terraform.tfvars
│   │   └── local/                 # Local k3d workspace (TF-9)
│   │       ├── main.tf            # Includes data.external for .env reading
│   │       ├── variables.tf
│   │       ├── outputs.tf         # Includes argocd_credentials
│   │       └── versions.tf
│   ├── bicep/                     # Azure Marketplace ONLY
│   │   └── managed-application/
│   └── scripts/
│       ├── deploy.sh              # Main entry point
│       ├── deploy-bicep.sh        # Bicep deployment
│       └── lib/
│           └── env.sh             # Shared env loading helper (TF-26)
└── sources/                       # Helm charts + values
```

### State Management

Terraform now uses an **azurerm remote backend**. The backend definition lives in `infra/terraform/azure/versions.tf` and points at the shared state account:

```hcl
backend "azurerm" {
  resource_group_name  = "ameide-tfstate"
  storage_account_name = "ameidetfstate"
  container_name       = "tfstate"
  key                  = "azure.tfstate"
  use_azuread_auth     = true
}
```

`infra/scripts/deploy.sh` (TF-10/TF-14) ensures the resource group, storage account, and container exist before it runs `terraform init`. Defaults can be overridden via environment variables:

| Variable | Default | Purpose |
|----------|---------|---------|
| `TF_BACKEND_RG` | `ameide-tfstate` | Resource group for remote state |
| `TF_BACKEND_SA` | `ameidetfstate` | Storage account hosting the container |
| `TF_BACKEND_CONTAINER` | `tfstate` | Blob container name |
| `TF_BACKEND_KEY` | `azure.tfstate` | Blob key used by the backend |
| `TF_BACKEND_LOCATION` | `<tfvars location>` | Location used when auto-creating the RG/SA |

Set any of the above before invoking `deploy.sh azure` to target alternate infra (e.g. per-tenant state stores). The backend uses Azure AD auth, so developers simply `az login` prior to running Terraform.

## Deployment Flow

```
deploy.sh azure
    ├── terraform -chdir=infra/terraform/azure apply
    │   └── Creates: AKS, KeyVault, DNS, managed identities
    └── bootstrap/bootstrap.sh
        └── Installs: ArgoCD + ApplicationSet

deploy.sh azure-bicep
    ├── az deployment group create (Bicep)
    │   └── Creates: Same resources via ARM
    └── bootstrap/bootstrap.sh (or Bicep deployment script)

deploy.sh local
    ├── docker network create ameide-network (if not exists)
    ├── terraform -chdir=infra/terraform/local apply
    │   └── Creates: k3d cluster via moio/k3d provider
    └── bootstrap/bootstrap.sh
        ├── Seeds `vault-bootstrap-local-secrets` via `infra/scripts/seed-local-secrets.sh`
        ├── Installs: ArgoCD + ApplicationSet (same as cloud)
        └── Starts ArgoCD port-forward + prints admin credentials
```

## Terraform Modules

### azure-aks

Provisions AKS cluster with environment-specific node pools.

```hcl
module "aks" {
  source = "../modules/azure-aks"

  name                       = local.cluster_name
  environment                = var.environment
  location                   = var.location
  resource_group_name        = data.azurerm_resource_group.main.name
  kubernetes_version         = var.kubernetes_version
  aad_tenant_id              = var.aad_tenant_id
  admin_group_object_ids     = var.admin_group_object_ids
  user_assigned_identity_id  = module.aks_identity.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  linux_admin_public_key     = var.linux_admin_public_key

  # Pool settings are individual variables, not a map
  system_pool_vm_size = "Standard_D4as_v6"
  system_pool_count   = 2

  dev_pool_vm_size   = "Standard_D2as_v6"
  dev_pool_min_count = 2
  dev_pool_max_count = 5

  staging_pool_vm_size   = "Standard_D2as_v6"
  staging_pool_min_count = 2
  staging_pool_max_count = 5

  prod_pool_vm_size   = "Standard_D4as_v6"
  prod_pool_min_count = 3
  prod_pool_max_count = 8
}
```

The module always enables workload identity/oidc and exposes taints/labels internally, so consumers do not need feature flags for those settings.

### azure-identity

Managed identities with workload identity federation for Kubernetes.

```hcl
module "dns_identity" {
  source = "../modules/azure-identity"

  name                = "ameide-dns-mi"
  resource_group_name = azurerm_resource_group.main.name

  federated_credentials = {
    cert-manager = {
      issuer  = module.aks.oidc_issuer_url
      subject = "system:serviceaccount:argocd:argocd-cert-manager"
    }
  }

  role_assignments = {
    dns_contributor = {
      scope = azurerm_dns_zone.main.id
      role  = "DNS Zone Contributor"
    }
  }
}
```

`federated_credentials` keys map to direct OIDC subjects; the module does not accept `namespace`/`service_account` fields.

### k3d-cluster

Local Kubernetes cluster via [moio/k3d](https://github.com/moio/terraform-provider-k3d) Terraform provider (v0.0.12).

```hcl
module "k3d" {
  source = "../modules/k3d-cluster"

  name         = "ameide"
  servers      = 1
  agents       = 2
  network_name = "ameide-network"
  k3s_image    = "rancher/k3s:v1.31.3-k3s1"  # Pinned to avoid startup bugs

  port_mappings = [
    { host = 443, container = 443 },   # HTTPS ingress
    { host = 80, container = 80 },     # HTTP ingress
  ]
}
```

`deploy.sh local` already provisions this workspace: it ensures Docker is running, creates the `ameide-network` bridge if needed, applies Terraform, and then runs the normal bootstrap flow.

**Design decisions:**
- **Provider**: `moio/k3d` v0.0.12 (shell fallback only if provider fails)
- **Secrets**: The local Terraform workspace reads `.env` (or exported env vars) through `data.external` to supply `GITHUB_TOKEN`; infrastructure does not create Azure resources when targeting local-only flows.
- **Registry**: GHCR (no local registry)
- **Traefik**: Disabled (we use Envoy Gateway)
- **Kubeconfig**: Auto-updated via provider (`update_default_kubeconfig = true`)

**What stays the same:**
- Bootstrap flow: ArgoCD + ApplicationSets
- Image registry: GHCR

**Known issues:**
- Port mappings (80/443) on `ameide-network` will collide if multiple local clusters are running
- k3d versions may have startup bugs (see [k3d#1481](https://github.com/k3d-io/k3d/issues/1481)) - pin a known-good version in dev tooling

## Deploy Script

Location: `infra/scripts/deploy.sh`

```bash
./infra/scripts/deploy.sh azure           # Terraform + bootstrap
./infra/scripts/deploy.sh azure-bicep     # Bicep + bootstrap
./infra/scripts/deploy.sh local           # k3d + bootstrap
./infra/scripts/deploy.sh bootstrap-azure # Bootstrap only (Azure)
./infra/scripts/deploy.sh bootstrap-local # Bootstrap only (local)
```

The script orchestrates:
1. Terraform/Bicep infrastructure provisioning
2. AKS credential retrieval
3. ArgoCD bootstrap via `bootstrap/bootstrap.sh --install-argo --apply-root-apps`

## Migration from Bicep

### Phase 1: Terraform Azure Module

1. Create `infra/terraform/modules/azure-aks/`
2. Create `infra/terraform/modules/azure-keyvault/`
3. Create `infra/terraform/modules/azure-dns/`
4. Create `infra/terraform/modules/azure-identity/`
5. Create `infra/terraform/azure/` workspace

### Phase 2: Validate Parity

1. Deploy Terraform to new resource group
2. Compare outputs with Bicep deployment
3. Verify ArgoCD bootstrap works identically

### Phase 3: Switch Default

1. Update `scripts/deploy.sh` to default to Terraform
2. Move Bicep to `azure-bicep` target
3. Update README and documentation

### Phase 4: Future Targets

- [ ] AWS EKS module
- [ ] GCP GKE module (if needed)

## Terraform vs Bicep Feature Mapping

| Bicep Resource | Terraform Resource |
|----------------|-------------------|
| `Microsoft.ContainerService/managedClusters` | `azurerm_kubernetes_cluster` |
| `Microsoft.KeyVault/vaults` | `azurerm_key_vault` |
| `Microsoft.Network/dnsZones` | `azurerm_dns_zone` |
| `Microsoft.ManagedIdentity/userAssignedIdentities` | `azurerm_user_assigned_identity` |
| `Microsoft.ManagedIdentity/.../federatedIdentityCredentials` | `azurerm_federated_identity_credential` |
| `Microsoft.OperationalInsights/workspaces` | `azurerm_log_analytics_workspace` |
| `Microsoft.Resources/deploymentScripts` | `null_resource` + `local-exec` |

## Outputs

Terraform (`infra/terraform/azure/outputs.tf`) now exposes the same normalized keys as Bicep, including per-environment maps that bootstrap consumes:

```hcl
output "envoy_public_ips" {
  value = {
    for ns in local.env_namespaces : ns => {
      address     = azurerm_public_ip.envoy[ns].ip_address
      resource_id = azurerm_public_ip.envoy[ns].id
    }
  }
}

output "dns_identity_client_ids" {
  value = {
    for env, identity in module.dns_identity : env => identity.client_id
  }
}

output "backup" {
  value = {
    for env in local.env_namespaces : env => {
      storage_account_name  = var.enable_backup_storage ? azurerm_storage_account.backup[env].name : ""
      blob_endpoint         = var.enable_backup_storage ? azurerm_storage_account.backup[env].primary_blob_endpoint : ""
      identity_client_id    = var.enable_backup_storage ? module.backup_identity[env].client_id : ""
      identity_principal_id = var.enable_backup_storage ? module.backup_identity[env].principal_id : ""
    }
  }
}

output "environment_dns_zone" {
  value = local.environment_dns_zone
}
```

`infra/scripts/test-iac-consistency.sh` validates that the Terraform and Bicep schemas remain aligned. `deploy.sh azure` writes Terraform outputs to `artifacts/terraform-outputs/azure.json` for bootstrap consumption.

## Secrets Seeding from .env

The `deploy.sh` script automatically seeds secrets from `.env` and `.env.local` to Azure Key Vault during the **Azure** Terraform deployment. The `local` target does not push anything into Key Vault; instead, `infra/terraform/local/main.tf` reads `GITHUB_TOKEN` via a `data.external` helper that falls back to `.env`. See [451-secrets-management.md](451-secrets-management.md) for the complete cloud secret flow.

### Local-only bootstrap

`foundation-vault-bootstrap` now has `secretSource.type` (`azure` or `local`). Cloud environments keep the Azure Key Vault path, while the `local` overlay switches to a Kubernetes Secret mount (`vault-bootstrap-local-secrets`) and sets `LOCAL_SECRET_FILE` for the CronJob.

`deploy.sh local` seeds that secret via `infra/scripts/seed-local-secrets.sh`, which reuses the `.env` sanitization logic and applies the secret in the `ameide-local` namespace before ArgoCD bootstraps the chart. Local clusters no longer require any Azure resources.

The Helm template also guards against missing chart overrides. When `secretSource=local`, the CronJob automatically mounts the JSON file and defaults `EXTERNAL_SECRETS_NAMESPACES` to the release namespace. When `secretSource=azure`, it falls back to Azure Key Vault and uses the federated workload identity outputs from Terraform. In either case the CronJob now injects the Kubernetes audience (`https://kubernetes.default.svc`) when it creates Vault roles, matching the requirements introduced in Vault 1.21.

> **TF-21**: During Azure deploys, `deploy.sh` proactively deletes legacy `env-*` Key Vault secrets so only the sanitized hyphenated keys remain. This keeps the vault clean once the CronJob migrates to the new naming scheme.

### How It Works

1. **Parse .env files** → `deploy.sh` reads `.env` and `.env.local`
2. **Sanitize keys** → `GHCR_TOKEN` becomes `ghcr-token` (lowercase, underscores to dashes)
3. **Create tfvars.json** → Temporary file with `env_secrets` variable
4. **Terraform apply** → `azurerm_key_vault_secret` resources created
5. **vault-bootstrap** → CronJob fetches from Key Vault, writes to HashiCorp Vault
6. **ExternalSecrets** → Syncs from Vault to Kubernetes Secrets

```bash
# Example .env file
GHCR_TOKEN=ghp_xxx
GHCR_USER=myuser
LANGFUSE_SECRET_KEY=xxx

# Results in Azure Key Vault secrets:
# - ghcr-token
# - ghcr-user
# - langfuse-secret-key
```

### Terraform Configuration

```hcl
# infra/terraform/azure/main.tf
resource "azurerm_key_vault_secret" "env_secrets" {
  for_each = nonsensitive(toset(keys(var.env_secrets)))

  name         = each.key   # Already sanitized by deploy.sh
  value        = var.env_secrets[each.key]
  key_vault_id = module.keyvault.id
}
```

> **Security note**: `azurerm_key_vault_secret` values are stored in plain text in Terraform state/plan. This is mitigated by pushing secrets into HashiCorp Vault afterwards (secrets don't persist in state long-term). Future: Terraform ≥1.11 may support write-only args for providers.

### Helm Values for vault-bootstrap

```yaml
# sources/values/dev/foundation/foundation-vault-bootstrap.yaml
azure:
  workloadIdentity:
    enabled: true
    clientId: "<vault-bootstrap-mi-client-id>"
    tenantId: "<tenant-id>"
  keyVault:
    uri: "https://ameidekv.vault.azure.net/"
    secretPrefix: ""   # No prefix - matches sanitized .env keys
```

## Backlog

- [x] Design Terraform architecture
- [x] **TF-1**: Create azure-aks module → `infra/terraform/modules/azure-aks/`
- [x] **TF-2**: Create azure-keyvault module → `infra/terraform/modules/azure-keyvault/`
- [x] **TF-3**: Create azure-dns module → `infra/terraform/modules/azure-dns/`
- [x] **TF-4**: Create azure-identity module → `infra/terraform/modules/azure-identity/`
- [x] **TF-5**: Create Azure workspace → `infra/terraform/azure/`
- [x] **TF-6**: Validate parity with Bicep → 33 resources imported
- [x] **TF-7**: Refactor deploy.sh → `infra/scripts/deploy.sh` with bootstrap integration
- [x] **TF-10**: Enable remote state backend → azurerm backend block added to `infra/terraform/azure/versions.tf`; uses `ameide-tfstate/ameidetfstate/tfstate` by default → **2025-12-11**
- [x] **TF-11**: Normalize output names → 23 outputs now use consistent snake_case across Terraform and Bicep
- [x] **TF-12**: Add per-env Envoy IPs to Bicep → `main.bicep` now creates 4 IPs (ArgoCD + 3 env-specific)
- [x] **TF-13**: Create consistency test script → `infra/scripts/test-iac-consistency.sh`
- [x] **TF-14**: Auto-create backend storage → `deploy.sh` now provisions the tfstate resource group, storage account, and container (override via `TF_BACKEND_*` env vars) → **2025-12-11**
- [x] **TF-15**: Unify .env loading → `deploy.sh` now loads `.env`/`.env.local` like `deploy-bicep.sh` → **2025-12-04**
- [x] **TF-16**: Add error trap with line numbers → Better debugging aligned with `deploy-bicep.sh` → **2025-12-04**
- [x] **TF-17**: Add Key Vault soft-delete recovery → Pre-deployment recovery aligned with Bicep → **2025-12-04**
- [x] **TF-18**: Add ArgoCD CLI context refresh → Auto-login after bootstrap → **2025-12-04**
- [x] **TF-19**: Parse .env secrets to Azure Key Vault → `deploy.sh` now seeds secrets via `env_secrets` Terraform variable → **2025-12-05**
- [x] **TF-20**: Federated identity credentials per environment → vault-bootstrap can authenticate from ameide-dev/staging/prod namespaces → **2025-12-05**
- [x] **TF-22**: Add explicit DNS subdomain records → explicit A records (www, platform, api, etc.) override wildcards and must be Terraform-managed → **2025-12-05**
- [x] **TF-21**: Clean up legacy `env-*` prefixed secrets from Azure Key Vault (19 secrets) → deploy script deletes/purges `env-*` secrets before seeding sanitized keys → **2025-12-11**
- [x] **TF-23**: Add `secretSource=local` mode to `foundation-vault-bootstrap` so k3d clusters can bootstrap without Azure Key Vault → **2025-12-10**
- [x] **TF-24**: Create `infra/scripts/seed-local-secrets.sh` and wire it into `deploy.sh local` to populate `vault-bootstrap-local-secrets` → **2025-12-10**
- [ ] **TF-8**: AWS EKS module (future)
- [x] **TF-9**: Local k3d setup → Implemented via `infra/terraform/local/` and `deploy.sh local` target → **2025-12-10**
- [x] **TF-27**: Auto-read `.env` from Terraform → `data.external.env` reads `GITHUB_TOKEN` directly from `.env`/`.env.local`, no manual `export` required → **2025-12-11**
- [x] **TF-28**: ArgoCD port-forward on terraform apply → Terraform auto-starts port-forward to `localhost:8443` after ArgoCD install → **2025-12-11**
- [x] **TF-29**: Expose ArgoCD credentials in terraform output → `terraform output -json argocd_credentials` shows actual password (sensitive) → **2025-12-11**
- [x] **TF-26**: Unified env loading helper → `infra/scripts/lib/env.sh` provides `load_env_files` with `set -a` for both `deploy.sh` and `deploy-bicep.sh` → **2025-12-11**

## TF-9: Local k3d Target (Reference)

Implementation lives in `infra/terraform/local/` and the `deploy.sh local` target; the notes below capture the delivered structure.

**Goal (shipped)**: `deploy.sh local` with the same aligned experience as Azure - Terraform manages full lifecycle.

### Files Created

| File | Purpose |
|------|---------|
| `infra/terraform/modules/k3d-cluster/main.tf` | k3d cluster resource via moio/k3d |
| `infra/terraform/modules/k3d-cluster/variables.tf` | name, servers, agents, network, ports, k3s_image |
| `infra/terraform/modules/k3d-cluster/outputs.tf` | cluster name |
| `infra/terraform/modules/argocd-bootstrap/main.tf` | ArgoCD install, GitOps manifests, port-forward |
| `infra/terraform/modules/argocd-bootstrap/variables.tf` | overlay, namespace, github_token, port_forward_port |
| `infra/terraform/modules/argocd-bootstrap/outputs.tf` | argocd_url, argocd_admin_password |
| `infra/terraform/local/main.tf` | Module instantiation + .env reading |
| `infra/terraform/local/variables.tf` | Local workspace vars |
| `infra/terraform/local/outputs.tf` | cluster_name, cluster_context, argocd_credentials |
| `infra/terraform/local/versions.tf` | Local state (no remote backend) |
| `infra/scripts/lib/env.sh` | Shared env loading helper with `set -a` |
| `bootstrap/configs/local.yaml` | Bootstrap config for k3d |

### k3d-cluster Module

```hcl
# modules/k3d-cluster/main.tf
terraform {
  required_providers {
    k3d = {
      source  = "moio/k3d"
      version = "0.0.12"
    }
  }
}

resource "k3d_cluster" "this" {
  name    = var.name
  servers = var.servers
  agents  = var.agents
  network = var.network_name
  image   = var.k3s_image  # Pin k3s version to avoid startup bugs

  dynamic "port" {
    for_each = var.port_mappings
    content {
      host_port      = port.value.host
      container_port = port.value.container
      node_filters   = ["loadbalancer"]
    }
  }

  k3s {
    extra_args {
      arg          = "--disable=traefik"
      node_filters = ["server:*"]
    }
  }

  kubeconfig {
    update_default_kubeconfig = true
    switch_current_context    = true
  }
}
```

### argocd-bootstrap Module

Installs ArgoCD via Helm CLI, applies GitOps manifests, and optionally starts port-forward.

```hcl
# modules/argocd-bootstrap/main.tf
module "argocd" {
  source = "../modules/argocd-bootstrap"

  gitops_root       = var.gitops_root
  overlay           = "local"               # Kustomize overlay (local, azure)
  namespace         = "ameide-local"        # Environment namespace
  github_token      = local.github_token    # For private repo access
  port_forward_port = 8443                  # 0 to disable
}
```

**Outputs:**
- `argocd_namespace` → "argocd"
- `argocd_url` → "https://localhost:8443" (if port-forward enabled)
- `argocd_admin_password` → Actual password (sensitive)
- `environment_namespace` → The namespace created

**Features (TF-27 to TF-29):**
- Reads `GITHUB_TOKEN` from environment or `.env` file via `data.external`
- Auto-starts port-forward on `localhost:8443` (detached process)
- Exposes actual ArgoCD admin password via `terraform output -json argocd_credentials`

#### Terraform outputs & artifacts

Both the Azure and local workspaces emit the same core outputs:

| Output | Description |
|--------|-------------|
| `argocd_namespace` | Namespace hosting the control plane (`argocd`) |
| `argocd_url` | URL served by the live ArgoCD instance (port-forward for local) |
| `argocd_credentials` | Struct containing username and password (sensitive) |
| `cluster_name` / `cluster_context` | Identifiers consumed by bootstrap scripts |
| `environment_namespace` | Namespace that backs the environment (`ameide-<env>`) |

Running `terraform output -json argocd_credentials` returns the banner credentials printed by `deploy.sh`. The CLI also serializes the full output set to `artifacts/terraform-outputs/<target>.json` so other tooling (bootstrap, smoke tests) can consume it without re-running Terraform.

### deploy.sh local case

```bash
local_k3d() {
  log "Deploying local k3d cluster with Terraform"
  local tf_dir="${REPO_ROOT}/infra/terraform/local"

  if ! docker info > /dev/null 2>&1; then
    fail "Docker is not running. Please start Docker and try again."
  fi

  if ! docker network inspect ameide-network >/dev/null 2>&1; then
    log "Creating Docker network: ameide-network"
    docker network create ameide-network
  fi

  cd "${tf_dir}"
  terraform init -upgrade
  terraform apply -auto-approve
  cd "${REPO_ROOT}"

  log "k3d cluster created. Context switched to k3d-ameide"
}
```

### bootstrap/configs/local.yaml

```yaml
# Aligned with existing dev.yaml schema
env: local

cluster:
  context: k3d-ameide
  type: k3d

gitops:
  root: .
  env: dev

bicepOutputsFile: ""  # No Bicep for local
envFile: .env         # Reuse existing .env for local secrets

secrets:
  repo:
    strategy: env     # No workload identity in k3d
    tokenEnv: GITHUB_TOKEN

  docker:
    strategy: env
    registry: ghcr.io
    usernameEnv: GHCR_USER
    tokenEnv: GHCR_TOKEN
```
