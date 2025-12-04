# 444 – Terraform Infrastructure

**Created**: 2025-12-04

> **Related documents:**
> - [439-deploy-infrastructure.md](439-deploy-infrastructure.md) – Deployment flow and scripts
> - [443-tenancy-models.md](443-tenancy-models.md) – Multi-tenant architecture
> - [148-azure-managed-offering.md](148-azure-managed-offering.md) – Azure Marketplace (Bicep required)

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
  local         k3d + Terraform (future)
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
│   │   │   └── azure-identity/    # Managed identities + federation
│   │   └── azure/                 # Azure workspace
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── terraform.tfvars
│   ├── bicep/                     # Azure Marketplace ONLY
│   │   └── managed-application/
│   └── scripts/
│       ├── deploy.sh              # Main entry point
│       └── deploy-bicep.sh        # Bicep deployment
└── sources/                       # Helm charts + values
```

### State Management

```hcl
# Azure backend (infra/terraform/azure/versions.tf)
terraform {
  backend "azurerm" {
    resource_group_name  = "ameide-tfstate"
    storage_account_name = "ameidetfstate"
    container_name       = "tfstate"
    key                  = "azure.tfstate"
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

## Backlog

- [x] Design Terraform architecture
- [x] **TF-1**: Create azure-aks module → `infra/terraform/modules/azure-aks/`
- [x] **TF-2**: Create azure-keyvault module → `infra/terraform/modules/azure-keyvault/`
- [x] **TF-3**: Create azure-dns module → `infra/terraform/modules/azure-dns/`
- [x] **TF-4**: Create azure-identity module → `infra/terraform/modules/azure-identity/`
- [x] **TF-5**: Create Azure workspace → `infra/terraform/azure/`
- [x] **TF-6**: Validate parity with Bicep → 33 resources imported
- [x] **TF-7**: Refactor deploy.sh → `infra/scripts/deploy.sh` with bootstrap integration
- [ ] **TF-8**: AWS EKS module (future)
- [ ] **TF-9**: Local k3d setup (future)
