# 444 – Terraform Infrastructure

**Created**: 2025-12-04
**Updated**: 2025-12-10

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
| Azure Marketplace SaaS | Not supported | Required |
| Multi-cloud (AWS, GCP) | Native | Azure only |
| Local development | Flexible | N/A |
| State management | Backend needed | None |
| Developer experience | Consistent | Azure-specific |

**Decision**: Terraform as primary IaC, Bicep only for Marketplace.

## Command Interface

```bash
./infra/scripts/deploy.sh [target] -e [environment]

Targets:
  azure         Terraform Azure (default)
  azure-bicep   Bicep for Azure Marketplace only
  aws           Terraform AWS (future)
  local         k3d + Terraform local dev cluster
  bootstrap     ArgoCD bootstrap only (any existing cluster)
```

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
│   │   │   └── k3d-cluster/       # Local k3d cluster module
│   │   ├── azure/                 # Azure workspace
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── terraform.tfvars
│   │   └── local/                 # Local k3d workspace
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   ├── bicep/                     # Azure Marketplace ONLY
│   │   └── managed-application/
│   └── scripts/
│       ├── deploy.sh              # Main entry point
│       └── deploy-bicep.sh        # Bicep deployment
└── sources/                       # Helm charts + values
```

### State Management

The Terraform backend storage is **auto-created** by `deploy.sh` if it doesn't exist:

```bash
# Backend storage configuration (can be overridden via environment variables)
TF_STATE_RG="ameide-tfstate"           # Resource group
TF_STATE_SA="ameidetfstate"            # Storage account
TF_STATE_CONTAINER="tfstate"           # Blob container
TF_STATE_LOCATION="westeurope"         # Location

# deploy.sh automatically:
# 1. Creates resource group if missing
# 2. Creates storage account with TLS 1.2, no public blob access
# 3. Creates container with Azure AD auth
# 4. Assigns Storage Blob Data Contributor role to current user
```

```hcl
# Azure backend (infra/terraform/azure/versions.tf)
terraform {
  backend "azurerm" {
    resource_group_name  = "ameide-tfstate"
    storage_account_name = "ameidetfstate"
    container_name       = "tfstate"
    key                  = "azure.tfstate"
    use_azuread_auth     = true  # No storage keys needed
  }
}
```

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
```

## Terraform Modules

### azure-aks

Provisions AKS cluster with environment-specific node pools.

```hcl
module "aks" {
  source = "../modules/azure-aks"

  name                = "ameide"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location

  node_pools = {
    system = {
      vm_size    = "Standard_D4as_v6"
      node_count = 2
      taints     = ["CriticalAddonsOnly=true:NoSchedule"]
    }
    dev = {
      vm_size   = "Standard_D2as_v6"
      min_count = 0
      max_count = 2
      labels    = { "ameide.io/pool" = "dev" }
      taints    = ["ameide.io/environment=dev:NoSchedule"]
    }
    staging = {
      vm_size   = "Standard_D2as_v6"
      min_count = 0
      max_count = 2
      labels    = { "ameide.io/pool" = "staging" }
      taints    = ["ameide.io/environment=staging:NoSchedule"]
    }
    prod = {
      vm_size   = "Standard_D4as_v6"
      min_count = 3
      max_count = 8
      labels    = { "ameide.io/pool" = "prod" }
      taints    = ["ameide.io/environment=production:NoSchedule"]
    }
  }

  enable_workload_identity = true
  enable_oidc_issuer       = true
}
```

### azure-identity

Managed identities with workload identity federation for Kubernetes.

```hcl
module "dns_identity" {
  source = "../modules/azure-identity"

  name                = "ameide-dns-mi"
  resource_group_name = azurerm_resource_group.main.name

  federated_credentials = {
    cert-manager = {
      issuer    = module.aks.oidc_issuer_url
      namespace = "cert-manager"
      service_account = "cert-manager"
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

## Deploy Script

Location: `infra/scripts/deploy.sh`

```bash
./infra/scripts/deploy.sh azure -e dev        # Terraform + bootstrap
./infra/scripts/deploy.sh azure-bicep -e dev  # Bicep + bootstrap
./infra/scripts/deploy.sh bootstrap           # Bootstrap only
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
- [x] Local k3d configuration
- [ ] GCP GKE module (if needed)

## Local k3d Deployment

The `local` target creates a k3d cluster for local development using Terraform.

### Usage

```bash
./infra/scripts/deploy.sh local
```

### What It Does

1. **Creates Docker network** `ameide-network` (for devcontainer connectivity)
2. **Runs Terraform** (`infra/terraform/local/`) to create k3d cluster:
   - 1 server + 2 agents
   - Ports 80/443 mapped to host
   - Traefik disabled (uses Envoy Gateway)
3. **Bootstraps ArgoCD** with local config (`bootstrap/configs/local.yaml`)
4. **Applies root ApplicationSet** for GitOps

### Local ArgoCD Configuration

The local environment uses `environments/local/argocd/values/argocd-values.yaml` which:
- Uses public Redis image (avoids GHCR auth chicken-and-egg)
- Disables imagePullSecrets (no private registry auth needed initially)
- Single replicas (resource efficiency)
- Disables ServiceMonitors and NetworkPolicy

Once ExternalSecrets deploys `ghcr-pull` secret, ArgoCD self-manages to GHCR mirror.

### k3d Terraform Module

```hcl
# infra/terraform/local/main.tf
module "k3d" {
  source = "../modules/k3d-cluster"

  name         = "ameide"
  servers      = 1
  agents       = 2
  network_name = "ameide-network"
  k3s_image    = var.k3s_image

  port_mappings = [
    { host = 443, container = 443 },
    { host = 80, container = 80 },
  ]
}
```

### Idempotent Deployment

Running `deploy.sh local` multiple times is safe:
- Terraform detects existing cluster (no-op if unchanged)
- Helm upgrades ArgoCD if already installed
- Bootstrap Application is applied idempotently

### Local Environment File Structure

The local environment follows the same two-location pattern as other environments:

```
# Bootstrap-time ArgoCD values (used by bootstrap/bootstrap.sh)
environments/local/argocd/values/argocd-values.yaml

# GitOps-managed component values (used by ApplicationSets)
sources/values/local/
├── globals.yaml      # Environment globals
├── foundation/       # Foundation component values
├── platform/         # Platform component values
└── apps/             # Application values
```

### ArgoCD Access (Local)

ArgoCD runs in insecure mode (HTTP) for local development - TLS would be terminated at gateway.

```bash
# Port-forward is started automatically by deploy.sh
# Access: http://localhost:8443
# Username: admin
# Password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### DevContainer Network Configuration

The devcontainer must be on the `ameide-network` Docker network to communicate with k3d:

```jsonc
// .devcontainer/devcontainer.json
"runArgs": ["--init", "--network=ameide-network"],
"forwardPorts": [8443],
"portsAttributes": {
  "8443": { "label": "ArgoCD UI" }
}
```

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

Both Terraform and Bicep produce identical outputs for bootstrap:

```hcl
output "aks_cluster_name" { value = module.aks.name }
output "aks_oidc_issuer" { value = module.aks.oidc_issuer_url }
output "keyvault_uri" { value = module.keyvault.vault_uri }
output "dns_identity_client_id" { value = module.dns_identity.client_id }
output "envoy_public_ip" { value = azurerm_public_ip.envoy.ip_address }
```

Outputs written to `artifacts/terraform-outputs/azure.json` for bootstrap consumption.

## Secrets Seeding from .env

The `deploy.sh` script automatically seeds secrets from `.env` and `.env.local` to Azure Key Vault during Terraform deployment. See [451-secrets-management.md](451-secrets-management.md) for the complete flow.

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
- [x] **TF-10**: Enable remote state backend → `infra/terraform/azure/versions.tf` (use_azuread_auth)
- [x] **TF-11**: Normalize output names → 23 outputs now use consistent snake_case across Terraform and Bicep
- [x] **TF-12**: Add per-env Envoy IPs to Bicep → `main.bicep` now creates 4 IPs (ArgoCD + 3 env-specific)
- [x] **TF-13**: Create consistency test script → `infra/scripts/test-iac-consistency.sh`
- [x] **TF-14**: Auto-create backend storage → `deploy.sh` now auto-creates RG, storage account, and container → **2025-12-04**
- [x] **TF-15**: Unify .env loading → `deploy.sh` now loads `.env`/`.env.local` like `deploy-bicep.sh` → **2025-12-04**
- [x] **TF-16**: Add error trap with line numbers → Better debugging aligned with `deploy-bicep.sh` → **2025-12-04**
- [x] **TF-17**: Add Key Vault soft-delete recovery → Pre-deployment recovery aligned with Bicep → **2025-12-04**
- [x] **TF-18**: Add ArgoCD CLI context refresh → Auto-login after bootstrap → **2025-12-04**
- [x] **TF-19**: Parse .env secrets to Azure Key Vault → `deploy.sh` now seeds secrets via `env_secrets` Terraform variable → **2025-12-05**
- [x] **TF-20**: Federated identity credentials per environment → vault-bootstrap can authenticate from ameide-dev/staging/prod namespaces → **2025-12-05**
- [x] **TF-22**: Add explicit DNS subdomain records → explicit A records (www, platform, api, etc.) override wildcards and must be Terraform-managed → **2025-12-05**
- [ ] **TF-21**: Clean up legacy `env-*` prefixed secrets from Azure Key Vault (19 secrets) → now using non-prefixed secrets with `secretPrefix: ""`
- [ ] **TF-8**: AWS EKS module (future)
- [x] **TF-9**: Local k3d setup → `infra/terraform/local/` + `infra/terraform/modules/k3d-cluster/` + `environments/local/argocd/values/` → **2025-12-10**
