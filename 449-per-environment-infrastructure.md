# 449 – Per-Environment Infrastructure

**Status**: Implemented
**Created**: 2025-12-04
**Updated**: 2025-12-05
**Related**: [446-namespace-isolation.md](446-namespace-isolation.md), [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), [448-per-environment-service-verification.md](448-per-environment-service-verification.md), [451-secrets-management.md](451-secrets-management.md)

---

## Problem Statement

Infrastructure was designed for a single-environment cluster. With namespace isolation (446/447), workloads run in per-environment namespaces (`ameide-dev`, `ameide-staging`, `ameide-prod`). Azure resources need corresponding per-environment instances.

---

## Resource Classification

### Cluster-Scoped (Shared)

| Resource | Namespace/Scope | Rationale |
|----------|-----------------|-----------|
| AKS Cluster | N/A | Single cluster, multi-environment |
| Key Vault | N/A | Shared secrets across environments (see [451](451-secrets-management.md)) |
| Log Analytics | N/A | Centralized observability |
| ArgoCD Public IP | `argocd` | Single ArgoCD instance |
| Vault Bootstrap Identity | Per-env federated | vault-bootstrap runs in each env namespace |
| DNS Identity | Multiple | Cert-manager shares DNS access |
| DNS Zones | N/A | Shared DNS infrastructure |

### Per-Environment

| Resource | Pattern | Rationale |
|----------|---------|-----------|
| Storage Account | `ameide{env}backup` | Backup isolation |
| Backup Identity | `ameide-{env}-backup-mi` | CNPG per-namespace |
| Envoy Public IP | `ameide-{env}-envoy-pip` | Ingress isolation |

---

## Implementation Status

### Terraform (`infra/terraform/azure/`) ✅

| Resource | Status |
|----------|--------|
| Storage Account | ✅ Per-environment (`for_each` over `env_namespaces`) |
| Backup Identity | ✅ Per-environment with correct federated credentials |
| Envoy IPs | ✅ Per-environment |

### Bicep (`infra/bicep/`) ✅

| Resource | Status |
|----------|--------|
| Storage Account | ✅ Per-environment (`[for ns in envNamespaces: ...]`) |
| Backup Identity | ✅ Per-environment with correct federated credentials |
| Envoy IPs | ✅ Per-environment |

---

## Implementation Details

### Key Vault (Cluster-Scoped)

Key Vault remains shared across environments. Secrets are namespaced by naming convention within the vault.

### Per-Environment Backup Infrastructure

```hcl
resource "azurerm_storage_account" "backup" {
  for_each = var.enable_backup_storage ? toset(local.environments) : toset([])

  name                     = lower(replace("${var.name_prefix}${each.key}backup", "-", ""))
  location                 = var.location
  resource_group_name      = data.azurerm_resource_group.main.name
  account_tier             = "Standard"
  account_replication_type = each.key == "prod" ? "GRS" : "LRS"
  min_tls_version          = "TLS1_2"

  blob_properties {
    versioning_enabled = true
  }

  tags = merge(local.base_tags, {
    ameide_io_environment = each.key
  })
}

module "backup_identity" {
  for_each = var.enable_backup_storage ? toset(local.environments) : toset([])
  source   = "../modules/azure-identity"

  name                = lower("${var.name_prefix}-${each.key}-backup-mi")
  location            = var.location
  resource_group_name = data.azurerm_resource_group.main.name

  federated_credentials = var.enable_backup_federated_identity ? {
    cnpg-backup = {
      issuer  = module.aks.oidc_issuer_url
      subject = "system:serviceaccount:ameide-${each.key}:postgres-ameide"
    }
  } : {}

  role_assignments = {
    storage-blob-contributor = {
      scope = azurerm_storage_account.backup[each.key].id
      role  = "Storage Blob Data Contributor"
    }
  }
}
```

### Outputs

```hcl
output "backup" {
  description = "Backup configuration per environment"
  value = {
    for env in local.env_namespaces : env => {
      storage_account_name  = var.enable_backup_storage ? azurerm_storage_account.backup[env].name : ""
      blob_endpoint         = var.enable_backup_storage ? azurerm_storage_account.backup[env].primary_blob_endpoint : ""
      identity_client_id    = var.enable_backup_storage ? module.backup_identity[env].client_id : ""
      identity_principal_id = var.enable_backup_storage ? module.backup_identity[env].principal_id : ""
    }
  }
}
```

### Federated Identity Subjects

Each environment's backup identity has a federated credential for the CNPG service account:

```hcl
cnpg_backup_subjects = {
  dev     = "system:serviceaccount:ameide-dev:postgres-ameide"
  staging = "system:serviceaccount:ameide-staging:postgres-ameide"
  prod    = "system:serviceaccount:ameide-prod:postgres-ameide"
}
```

### Vault Bootstrap Identity (Updated 2025-12-05)

The vault-bootstrap CronJob runs in per-environment namespaces (`ameide-dev`, `ameide-staging`, `ameide-prod`) and needs Azure Workload Identity to fetch secrets from Key Vault. This requires federated credentials for each namespace:

```hcl
# infra/terraform/azure/main.tf
vault_bootstrap_subjects = {
  dev     = "system:serviceaccount:ameide-dev:foundation-vault-bootstrap-vault-bootstrap"
  staging = "system:serviceaccount:ameide-staging:foundation-vault-bootstrap-vault-bootstrap"
  prod    = "system:serviceaccount:ameide-prod:foundation-vault-bootstrap-vault-bootstrap"
}

module "vault_bootstrap_identity" {
  source = "../modules/azure-identity"

  name = "ameide-vault-bootstrap-mi"
  # ...

  federated_credentials = {
    for name, subject in local.vault_bootstrap_subjects : "vault-bootstrap-${name}" => {
      issuer  = module.aks.oidc_issuer_url
      subject = subject
    }
  }

  role_assignments = {
    keyvault-secrets-user = {
      scope = module.keyvault.id
      role  = "Key Vault Secrets User"
    }
  }
}
```

**Important**: Without per-environment federated credentials, vault-bootstrap cannot authenticate to Azure Key Vault and falls back to placeholder secrets. See [451-secrets-management.md](451-secrets-management.md) for the full secrets flow.

---

## Migration

1. Apply infrastructure changes (creates per-env storage accounts and identities)
2. Update globals.yaml with per-environment backup values
3. Sync ArgoCD
4. Validate CNPG backups work in all environments

---

## Validation

```bash
# Storage Accounts (should show 3: ameidedevbackup, ameidestagingbackup, ameideprodbackup)
az storage account list -g Ameide \
  --query "[?contains(name,'backup')].{name:name,env:tags.ameide_io_environment}" -o table

# Federated Credentials (should show correct service account per env)
for env in dev staging prod; do
  echo "=== ${env} ==="
  az identity federated-credential list \
    --identity-name "ameide-${env}-backup-mi" \
    --resource-group Ameide \
    --query "[].{name:name, subject:subject}" -o table
done

# Terraform outputs
cd infra/terraform/azure && terraform output backup
```
