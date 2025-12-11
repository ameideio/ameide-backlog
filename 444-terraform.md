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
| Azure Marketplace SaaS | Can consume existing Managed App, cannot publish definition | Required to author/publish definition |
| Multi-cloud (AWS, GCP) | Native | Azure only |
| Local development | Flexible | Azure-only, not used here |
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
  local         k3d + Terraform (designed, not yet implemented)
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
│   │   │   └── k3d-cluster/       # Local k3d cluster (TF-9)
│   │   ├── azure/                 # Azure workspace
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── terraform.tfvars
│   │   └── local/                 # Local k3d workspace (TF-9)
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── versions.tf
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

> **Auth requirement**: Requires Azure CLI login (`az login`) with access to the tfstate storage account and **Storage Blob Data Contributor** role on the container. For CI with GitHub OIDC, can swap to `use_oidc = true`.

> **Current status**: Remote backend designed (TF-14) but not yet wired into `azure/versions.tf`. The azure workspace currently uses local state in devcontainer. Use the example `backend "azurerm"` block above when turning on remote state.

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
        └── Installs: ArgoCD + ApplicationSet (same as cloud)
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

**Design decisions:**
- **Provider**: `moio/k3d` v0.0.12 (shell fallback only if provider fails)
- **Secrets**: Same Azure Key Vault chain as cloud targets
- **Registry**: GHCR (no local registry)
- **Traefik**: Disabled (we use Envoy Gateway)
- **Kubeconfig**: Auto-updated via provider (`update_default_kubeconfig = true`)

**What stays the same:**
- Secrets chain: `.env → AKV → vault-bootstrap → Vault → K8s Secrets`
- Bootstrap flow: ArgoCD + ApplicationSets
- Image registry: GHCR

**Known issues:**
- Port mappings (80/443) on `ameide-network` will collide if multiple local clusters are running
- k3d versions may have startup bugs (see [k3d#1481](https://github.com/k3d-io/k3d/issues/1481)) - pin a known-good version in dev tooling

## Deploy Script

Location: `infra/scripts/deploy.sh`

```bash
./infra/scripts/deploy.sh azure -e dev        # Terraform + bootstrap
./infra/scripts/deploy.sh azure-bicep -e dev  # Bicep + bootstrap
./infra/scripts/deploy.sh local -e dev        # k3d + bootstrap (TF-9)
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
- [ ] Local k3d configuration
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
- [x] **TF-10**: Enable remote state backend → designed with `use_azuread_auth`; example in 444, not yet wired into `azure/versions.tf`
- [x] **TF-11**: Normalize output names → 23 outputs now use consistent snake_case across Terraform and Bicep
- [x] **TF-12**: Add per-env Envoy IPs to Bicep → `main.bicep` now creates 4 IPs (ArgoCD + 3 env-specific)
- [x] **TF-13**: Create consistency test script → `infra/scripts/test-iac-consistency.sh`
- [x] **TF-14**: Auto-create backend storage → `deploy.sh` can auto-create RG, storage account, and container; backend block not yet in `azure/versions.tf` → **2025-12-04**
- [x] **TF-15**: Unify .env loading → `deploy.sh` now loads `.env`/`.env.local` like `deploy-bicep.sh` → **2025-12-04**
- [x] **TF-16**: Add error trap with line numbers → Better debugging aligned with `deploy-bicep.sh` → **2025-12-04**
- [x] **TF-17**: Add Key Vault soft-delete recovery → Pre-deployment recovery aligned with Bicep → **2025-12-04**
- [x] **TF-18**: Add ArgoCD CLI context refresh → Auto-login after bootstrap → **2025-12-04**
- [x] **TF-19**: Parse .env secrets to Azure Key Vault → `deploy.sh` now seeds secrets via `env_secrets` Terraform variable → **2025-12-05**
- [x] **TF-20**: Federated identity credentials per environment → vault-bootstrap can authenticate from ameide-dev/staging/prod namespaces → **2025-12-05**
- [x] **TF-22**: Add explicit DNS subdomain records → explicit A records (www, platform, api, etc.) override wildcards and must be Terraform-managed → **2025-12-05**
- [ ] **TF-21**: Clean up legacy `env-*` prefixed secrets from Azure Key Vault (19 secrets) → now using non-prefixed secrets with `secretPrefix: ""`
- [ ] **TF-8**: AWS EKS module (future)
- [ ] **TF-9**: Local k3d setup → **designed 2025-12-10**, implementation pending

## TF-9: Local k3d Target (Design)

**Goal**: `deploy.sh local` with same aligned experience as Azure - Terraform manages full lifecycle.

### Files to Create

| File | Purpose |
|------|---------|
| `infra/terraform/modules/k3d-cluster/main.tf` | k3d cluster resource via moio/k3d |
| `infra/terraform/modules/k3d-cluster/variables.tf` | name, servers, agents, network, ports, k3s_image |
| `infra/terraform/modules/k3d-cluster/outputs.tf` | cluster name |
| `infra/terraform/local/main.tf` | Module instantiation |
| `infra/terraform/local/variables.tf` | Local workspace vars |
| `infra/terraform/local/outputs.tf` | cluster_name, cluster_context |
| `infra/terraform/local/versions.tf` | Local state (no remote backend) |
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

### deploy.sh local case

```bash
local)
    echo "Deploying local k3d cluster..."
    docker info > /dev/null 2>&1 || { echo "Docker not running"; exit 1; }
    docker network inspect ameide-network >/dev/null 2>&1 || \
        docker network create ameide-network
    terraform -chdir=infra/terraform/local init
    terraform -chdir=infra/terraform/local apply -auto-approve
    ./bootstrap/bootstrap.sh \
        --config bootstrap/configs/local.yaml \
        --install-argo \
        --apply-root-apps
    ;;
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
