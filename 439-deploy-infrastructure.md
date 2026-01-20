# 439 – Deploy Infrastructure

**Status:** ✅ Implemented (core infra complete, CI/CD automation pending)
**Created**: 2025-12-04

> **Related documents:**
> - [443-tenancy-models.md](443-tenancy-models.md) – **Tenancy model overview (single source of truth)**
> - [367-bootstrap-v2.md](367-bootstrap-v2.md) – Bootstrap CLI design and GitOps initialization
> - [148-azure-managed-offering.md](148-azure-managed-offering.md) – Azure Managed Application packaging for Marketplace
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) – Environment naming, namespace conventions, GitOps structure
> - [438-cert-manager-dns01-azure-workload-identity.md](438-cert-manager-dns01-azure-workload-identity.md) – DNS identity and cert-manager workload identity
> - [435-remote-first-development.md](435-remote-first-development.md) – Remote-first development workflow
> - [712-traffic-manager-first-redesign.md](712-traffic-manager-first-redesign.md) – Edge-first traffic manager redesign (Front Door + BYOC certs in DNS subscription)

## Overview

This document describes the end-to-end infrastructure deployment for AMEIDE environments. It covers the Bicep templates, deployment scripts, and the flow from Azure resource provisioning through GitOps bootstrap.

## Target Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Azure Subscription                                  │
│                                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                 │
│  │ Resource Group  │  │   DNS Zones     │  │  Key Vault      │                 │
│  │    Ameide       │  │                 │  │                 │                 │
│  │                 │  │  ameide.io      │  │  Secrets        │                 │
│  │  ┌───────────┐  │  │  ├─ dev.        │  │  ├─ env-*       │                 │
│  │  │    AKS    │  │  │  ├─ staging.    │  │  ├─ oauth       │                 │
│  │  │  ameide   │  │  │  └─ (apex)      │  │  └─ creds       │                 │
│  │  └───────────┘  │  └─────────────────┘  └─────────────────┘                 │
│  │                 │                                                            │
│  │  ┌───────────┐  │  ┌─────────────────┐  ┌─────────────────┐                 │
│  │  │    ACR    │  │  │ Managed Identity│  │  Log Analytics  │                 │
│  │  │  ameide   │  │  │                 │  │                 │                 │
│  │  └───────────┘  │  │  ├─ ameide-mi   │  │  30-day retain  │                 │
│  │                 │  │  └─ ameide-dns  │  │                 │                 │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                 │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              AKS Cluster: ameide                                 │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         Cluster-Scoped Resources                         │   │
│  │                                                                          │   │
│  │  Namespace: argocd          Namespace: cert-manager                     │   │
│  │  ├─ ArgoCD Server           ├─ cert-manager controller                  │   │
│  │  ├─ ApplicationSets         └─ DNS-01 solver (workload identity)        │   │
│  │  └─ Projects                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐          │
│  │ Namespace:        │  │ Namespace:        │  │ Namespace:        │          │
│  │ ameide-dev        │  │ ameide-staging    │  │ ameide-prod       │          │
│  │                   │  │                   │  │                   │          │
│  │ Domain:           │  │ Domain:           │  │ Domain:           │          │
│  │ dev.ameide.io     │  │ staging.ameide.io │  │ ameide.io         │          │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Deployment Modes

The same Bicep + bootstrap stack supports three deployment modes corresponding to the tenancy models defined in [443-tenancy-models.md](443-tenancy-models.md):

| Mode | Description | Bicep Usage | Subscription |
|------|-------------|-------------|--------------|
| **Shared SaaS** | Single multi-tenant cluster `ameide` per region | `deploy-managed-app.sh staging/production` | AMEIDE subscription |
| **Namespace-per-tenant** | Tenants share the regional `ameide` cluster | No new AKS; GitOps creates `tenant-<id>-prod` namespaces | AMEIDE subscription |
| **Private cloud (per-tenant)** | Dedicated cluster per tenant (optionally in their subscription) | `deploy-managed-app.sh` with tenant-specific params | Customer subscription |

### Shared SaaS Deployment

Standard deployment for internal environments:
```bash
./scripts/deploy-managed-app.sh staging
./scripts/deploy-managed-app.sh production
```

### Namespace-per-Tenant Deployment

No infrastructure changes required. See [GitOps: Tenants on a Shared Cluster](#gitops-tenants-on-a-shared-cluster) below.

### Private Cloud Deployment

For dedicated tenant clusters, use tenant-specific parameters:
```bash
./scripts/deploy-managed-app.sh production \
  --deployment-mode privateTenant \
  --tenant-slug acme \
  --resource-group Ameide-Acme
```

> **Private cloud note**: For `deploymentMode=privateTenant`, deployment typically occurs in the
> customer's Azure subscription using Azure Managed Applications.
> See [148-azure-managed-offering.md](148-azure-managed-offering.md) for Marketplace packaging.

## Repository Structure

```
ameide-gitops/
├── infra/
│   ├── bicep/
│   │   └── managed-application/
│   │       ├── main.bicep                # Root template
│   │       ├── main.bicepparam           # Parameter file
│   │       └── modules/                  # AKS, KV, DNS, identities, records
│   ├── terraform/
│   │   └── azure/                        # Terraform outputs parity with Bicep
│   └── scripts/
│       └── sync-globals.sh               # Sync IaC outputs → globals.yaml
├── scripts/
│   ├── deploy-managed-app.sh             # Main deployment orchestrator
│   ├── setup-service-principal.sh        # SP with certificate auth
│   └── ensure-aks-admin-key.sh           # SSH key for AKS
├── bootstrap/
│   ├── bootstrap.sh                      # GitOps bootstrap CLI
│   ├── lib/
│   │   ├── cluster.sh                    # Cluster readiness checks
│   │   ├── argocd.sh                     # ArgoCD installation
│   │   ├── secrets.sh                    # Secret seeding
│   │   └── ...
│   └── configs/
│       ├── dev.yaml                      # Dev bootstrap config
│       ├── staging.yaml                  # Staging bootstrap config
│       └── production.yaml               # Production bootstrap config
└── sources/values/
    ├── dev/globals.yaml                  # Dev infrastructure values
    ├── staging/globals.yaml              # Staging infrastructure values
    └── production/globals.yaml           # Production infrastructure values
```

## Deployment Flow

### Phase 1: Prerequisites

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Azure Authentication                                          │
│    az login                                                      │
│    az account set --subscription <subscription-id>               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Service Principal (optional, for CI/CD)                       │
│    scripts/setup-service-principal.sh                            │
│                                                                  │
│    Creates:                                                      │
│    ├─ X.509 certificate (2-year validity)                       │
│    ├─ SP with Contributor + RBAC Administrator roles            │
│    └─ Profile JSON: creds/ameide-deploy-sp.json                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. SSH Key for AKS                                               │
│    scripts/ensure-aks-admin-key.sh                               │
│                                                                  │
│    Creates:                                                      │
│    ├─ RSA-4096 keypair (AKS requires RSA, not ed25519)          │
│    └─ Output: creds/aks-admin/aks-admin.pub                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Environment Variables                                         │
│    .env file with:                                               │
│    ├─ GITHUB_TOKEN (required for GitOps repo access)            │
│    ├─ DCKR_USERNAME, DCKR_PAT (Docker Hub pull)                 │
│    └─ Application secrets to materialize in Key Vault           │
└─────────────────────────────────────────────────────────────────┘
```

### Phase 2: Infrastructure Deployment (Bicep)

```
┌─────────────────────────────────────────────────────────────────┐
│ scripts/deploy-managed-app.sh <env>                              │
│                                                                  │
│ Example: ./scripts/deploy-managed-app.sh staging                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ az deployment group create                                       │
│   --resource-group Ameide                                        │
│   --template-file bicep/managed-application/main.bicep           │
│   --parameters environment=<env> ...                             │
│                                                                  │
│ Creates:                                                         │
│ ├─ User-Assigned Managed Identity (ameide-<env>-mi)             │
│ ├─ Log Analytics Workspace (30-day retention)                   │
│ ├─ Key Vault (RBAC, purge protection)                           │
│ ├─ AKS Cluster (workload identity, OIDC issuer)                 │
│ ├─ Container Registry (Standard/Premium)                        │
│ ├─ DNS Zone + child zones (dev/staging)                         │
│ └─ DNS Managed Identity with federated credentials              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Outputs captured to: artifacts/bicep-outputs/<env>.json          │
│                                                                  │
│ Key outputs:                                                     │
│ ├─ aksResourceId                                                 │
│ ├─ aksOidcIssuer                                                 │
│ ├─ keyVaultUri                                                   │
│ ├─ envoyPublicIpAddress                                          │
│ ├─ dnsManagedIdentityClientId                                    │
│ ├─ dnsManagedIdentityPrincipalId                                 │
│ └─ containerRegistryLoginServer                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ infra/scripts/sync-globals.sh <env>                              │
│                                                                  │
│ Updates: sources/values/<env>/globals.yaml                       │
│                                                                  │
│ Synced values:                                                   │
│ ├─ azure.envoyPublicIpAddress                                    │
│ ├─ azure.dnsManagedIdentityClientId                              │
│ ├─ azure.dnsManagedIdentityPrincipalId                           │
│ ├─ azure.keyVaultUri                                             │
│ └─ azure.vaultBootstrapIdentityClientId                          │
└─────────────────────────────────────────────────────────────────┘
```

**Consumption note (GitOps):**
- `azure.envoyPublicIpAddress` is consumed by the platform Gateway chart as `Gateway.spec.addresses` (static IP request), not by setting `EnvoyProxy.spec.provider.kubernetes.envoyService.loadBalancerIP`.

### Phase 3: GitOps Bootstrap

```
┌─────────────────────────────────────────────────────────────────┐
│ bootstrap/bootstrap.sh                                           │
│   --config bootstrap/configs/<env>.yaml                          │
│   --bicep-outputs artifacts/bicep-outputs/<env>.json             │
│   --install-argo                                                 │
│   --apply-repo-creds                                             │
│   --apply-projects-ops                                           │
│   --apply-root-apps                                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Cluster Readiness Check                                       │
│    └─ Polls /readyz until API server responds                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Install ArgoCD                                                │
│    └─ Helm install argo-cd with environment values              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Apply Repository Credentials                                  │
│    └─ kubectl apply argocd repo secrets (GITHUB_TOKEN)          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Apply Projects & Ops                                          │
│    └─ kubectl apply argocd/projects/*.yaml                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Apply Root Application                                        │
│    └─ kubectl apply argocd/applicationsets/ameide.yaml          │
│                                                                  │
│    Single ApplicationSet generates apps for all environments:   │
│    ├─ dev-<component>      → namespace: ameide-dev              │
│    ├─ staging-<component>  → namespace: ameide-staging          │
│    └─ prod-<component>     → namespace: ameide-prod             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. Wait for Sync (optional)                                      │
│    └─ --wait-tier foundation,data,platform,apps                 │
└─────────────────────────────────────────────────────────────────┘
```

### GitOps: Tenants on a Shared Cluster

For namespace-per-tenant deployments, we **do not** run Bicep again. Instead:

1. **Create a namespace per tenant** (e.g., `tenant-acme-prod`) with labels:
   ```yaml
   metadata:
     name: tenant-acme-prod
     labels:
       ameide.io/environment: production
       ameide.io/tenant-kind: namespace
       ameide.io/tenant-id: acme
   ```

2. **Extend the ApplicationSet** `argocd/applicationsets/ameide.yaml` `list` generator with a new element:
   ```yaml
   - env: production
     tenantKind: namespace
     tenantId: acme
     namespace: tenant-acme-prod
   ```

3. **Add per-tenant values** under `sources/values/tenants/acme/...`:
   ```
   sources/values/tenants/
   └── acme/
       ├── globals.yaml        # Tenant-specific config
       └── apps/               # Tenant-specific overrides
   ```

This keeps infrastructure provisioning (Bicep) and per-tenant workloads (GitOps) cleanly separated.

> **Reference:** See [443-tenancy-models.md](443-tenancy-models.md) for complete tenancy documentation.

## Bicep Modules Reference

### main.bicep

**Purpose:** Root orchestration template for managed application deployment.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `environment` | string | Yes | `staging` or `production` |
| `location` | string | No | Azure region (default: resource group location) |
| `namePrefix` | string | Yes | Resource naming prefix (min 3 chars) |
| `aadTenantId` | string | Yes | Azure AD tenant ID |
| `linuxAdminPublicKey` | string | Yes | SSH public key for AKS admin |
| `adminGroupObjectIds` | array | Yes | Azure AD group IDs for cluster admin |
| `dnsZoneName` | string | No | DNS zone name (empty to skip DNS) |
| `aksVersion` | string | No | Kubernetes version (default: 1.32.7) |
| `gitOpsRepoUrl` | string | Yes | GitOps repository URL |
| `gitOpsRevision` | string | No | Git branch/tag (default: main) |
| `gitOpsArgocdPath` | string | No | Path to ArgoCD manifests |
| `gitOpsRepoPassword` | securestring | No | Repository token for private repos |
| `deploymentMode` | string | No | `shared` \| `namespaceTenant` \| `privateTenant` (default: `shared`) |
| `tenantSlug` | string | No | Tenant identifier for cluster naming (used when `deploymentMode=privateTenant`) |

**Outputs (Normalized snake_case format - see [444-terraform.md](444-terraform.md) for parity):**

| Output | Description |
|--------|-------------|
| `aks_cluster_name` | AKS cluster name |
| `aks_resource_id` | AKS cluster resource ID |
| `aks_oidc_issuer` | OIDC issuer URL for workload identity |
| `keyvault_name` | Key Vault name |
| `keyvault_uri` | Key Vault URI |
| `argocd_public_ip_address` | Static IP for ArgoCD ingress |
| `envoy_public_ips` | Map of per-environment Envoy IPs (dev/staging/prod) |
| `dns_identity_client_id` | Client ID for DNS automation |
| `vault_bootstrap_identity_client_id` | Vault bootstrap identity |
| `backup_storage_account_name` | Backup storage account (optional) |
| `backup_identity_client_id` | Backup identity client ID (optional) |

> **Consistency testing**: Run `./infra/scripts/test-iac-consistency.sh schema` to verify output parity with Terraform.

### modules/aks.bicep

**Purpose:** AKS cluster with Azure AD integration and workload identity.

**Key Configuration:**

| Setting | Staging | Production |
|---------|---------|------------|
| Node VM Size | Standard_D2as_v6 | Standard_D4as_v6 |
| Node Count | 1 | 3-4 (autoscale) |
| Upgrade Channel | rapid | stable |
| Network Policy | Azure | Azure |
| RBAC | Azure AD | Azure AD |
| Workload Identity | Enabled | Enabled |
| OIDC Issuer | Enabled | Enabled |

### modules/dnsIdentity.bicep

**Purpose:** Managed identity for cert-manager DNS-01 challenges with workload identity federation.

**Resources Created:**
1. User-assigned managed identity (`ameide-dns-mi`)
2. Federated identity credential for cert-manager service account
3. DNS Zone Contributor role on parent and child zones
4. Child DNS zones (dev.ameide.io, staging.ameide.io)
5. NS delegation records

**Cross-reference:** See [438-cert-manager-dns01-azure-workload-identity.md](438-cert-manager-dns01-azure-workload-identity.md) for detailed architecture.

### modules/keyvault.bicep

**Purpose:** Key Vault with RBAC authorization and diagnostics.

**Key Configuration:**
- RBAC authorization (no access policies)
- Purge protection enabled (90-day soft-delete)
- Diagnostic settings: AuditEvent logs + AllMetrics
- SKU: standard (staging) / premium (production)

## Scripts Reference

### deploy-managed-app.sh

**Usage:**
```bash
./scripts/deploy-managed-app.sh <env> [deployment-name]
```

**Parameters:**
- `<env>`: Environment (`dev`, `staging`, `production`)
- `[deployment-name]`: Optional Azure deployment name override

**Environment Variables:**
- `GITHUB_TOKEN` (required): GitHub token for GitOps repo access
- `AZ_RESOURCE_GROUP` (optional): Resource group (default: Ameide)
- `AMEIDE_SP_PROFILE` (optional): Service principal profile for CI/CD

**Operations:**
1. Validates Azure CLI authentication
2. Ensures AKS admin SSH key exists
3. Parses .env files for Key Vault secrets
4. Handles soft-deleted Key Vault recovery
5. Executes Bicep deployment
6. Captures outputs to JSON
7. Updates globals.yaml
8. Refreshes kubectl context

### setup-service-principal.sh

**Usage:**
```bash
./scripts/setup-service-principal.sh
```

**Environment Variables:**
- `AZ_RESOURCE_GROUP`: Target resource group (default: Ameide)
- `AMEIDE_DEPLOY_SP_YEARS`: Certificate validity (default: 2)
- `AMEIDE_SP_ROTATE_DAYS`: Rotation threshold (default: 30)
- `AMEIDE_FORCE_SP_ROTATION`: Force rotation even if valid

**Output:** `creds/ameide-deploy-sp.json`

**Roles Assigned:**
- Contributor (scoped to resource group)
- RBAC Administrator (scoped to resource group)
- Key Vault Contributor (subscription level)

### ensure-aks-admin-key.sh

**Usage:**
```bash
./scripts/ensure-aks-admin-key.sh
```

**Output:** `creds/aks-admin/aks-admin.pub`

**Notes:**
- AKS requires RSA keys (ed25519 not supported)
- Generates RSA-4096 keypair
- Reuses existing keypair if present

### sync-globals.sh

**Usage:**
```bash
./infra/scripts/sync-globals.sh <env>
```

**Input:** `artifacts/bicep-outputs/<env>.json`

**Output:** Updates `sources/values/<env>/globals.yaml`

## Environment Matrix

| Environment | Namespace | Domain | DNS Zone | Status |
|-------------|-----------|--------|----------|--------|
| dev | ameide-dev | dev.ameide.io | dev.ameide.io | Active |
| staging | ameide-staging | staging.ameide.io | staging.ameide.io | Active |
| production | ameide-prod | ameide.io | ameide.io (apex) | Reserved |

**Cross-reference:** See [434-unified-environment-naming.md](434-unified-environment-naming.md) for complete environment matrix.

## Quick Start

### Deploy Staging Environment

```bash
# 1. Authenticate
az login
az account set --subscription <subscription-id>

# 2. Ensure prerequisites
./scripts/ensure-aks-admin-key.sh

# 3. Set required environment variables
export GITHUB_TOKEN=<your-token>
# Or add to .env file

# 4. Deploy infrastructure
./scripts/deploy-managed-app.sh staging

# 5. Bootstrap GitOps
./bootstrap/bootstrap.sh \
  --config bootstrap/configs/staging.yaml \
  --bicep-outputs artifacts/bicep-outputs/staging.json \
  --install-argo \
  --apply-repo-creds \
  --apply-projects-ops \
  --apply-root-apps

# 6. Verify
kubectl --context ameide-staging get pods -n argocd
argocd app list
```

### CI/CD Deployment

```bash
# 1. Create service principal (once)
./scripts/setup-service-principal.sh

# 2. In CI pipeline, authenticate with SP
az login --service-principal \
  --username $(jq -r .appId creds/ameide-deploy-sp.json) \
  --password creds/ameide-deploy-sp.pem \
  --tenant $(jq -r .tenant creds/ameide-deploy-sp.json)

# 3. Deploy
export AMEIDE_SP_PROFILE=creds/ameide-deploy-sp.json
./scripts/deploy-managed-app.sh staging
```

## Backlog Items

### Completed

- [x] Bicep modules for AKS, Key Vault, DNS, identities
- [x] deploy-managed-app.sh orchestration script
- [x] Bootstrap CLI with config file support
- [x] Bicep outputs → globals.yaml sync
- [x] DNS identity with workload identity federation
- [x] Single parametrized ApplicationSet
- [x] Bicep modules for backup storage (`storageAccount.bicep`, `backupIdentity.bicep`) → **2025-12-04**
- [x] **DEPLOY-8**: Per-environment Envoy IPs in Bicep (4 IPs: ArgoCD + 3 env-specific) → **2025-12-04**
- [x] **DEPLOY-9**: Normalized output schema (23 snake_case outputs matching Terraform) → **2025-12-04**
- [x] **DEPLOY-10**: IaC consistency test script (`infra/scripts/test-iac-consistency.sh`) → **2025-12-04**
- [x] **DEPLOY-11**: Terraform remote state backend enabled (`use_azuread_auth`) → **2025-12-04**
- [x] **DEPLOY-12**: Auto-create Terraform backend storage (`deploy.sh` creates RG/SA/container if missing) → **2025-12-04**
- [x] **DEPLOY-13**: Unify `.env` loading across both deployment scripts → **2025-12-04**
- [x] **DEPLOY-14**: Add error trap with line numbers to `deploy.sh` → **2025-12-04**
- [x] **DEPLOY-15**: Add Key Vault soft-delete recovery to `deploy.sh` → **2025-12-04**
- [x] **DEPLOY-16**: Add ArgoCD CLI context refresh to `deploy.sh` → **2025-12-04**

### In Progress

- [ ] **DEPLOY-1**: Add CI/CD pipeline for automated deployments
- [ ] **DEPLOY-2**: Implement Bicep what-if validation in PR checks (can use `test-iac-consistency.sh what-if`)
- [ ] **DEPLOY-3**: Add health checks after bootstrap (ArgoCD sync status)

### Pending

- [ ] **DEPLOY-4**: Production bootstrap config
- [ ] **DEPLOY-5**: Automated secret rotation for Key Vault
- [ ] **DEPLOY-6**: Disaster recovery runbook
- [ ] **DEPLOY-7**: Cost optimization (reserved instances, spot nodes)

## Validation Checklist

### Post-Bicep Deployment

- [ ] AKS cluster is running: `az aks show --name ameide --resource-group Ameide`
- [ ] Key Vault is accessible: `az keyvault show --name <vault-name>`
- [ ] DNS zones have NS records: `az network dns zone show --name <zone>`
- [ ] Managed identity has correct roles: `az role assignment list --assignee <principal-id>`

### Post-Bootstrap

- [ ] ArgoCD pods are running: `kubectl -n argocd get pods`
- [ ] ApplicationSet is created: `kubectl -n argocd get applicationset`
- [ ] Applications are syncing: `argocd app list`
- [ ] Certificates are issued: `kubectl get certificate -A`

## Troubleshooting

### Common Issues

| Issue | Cause | Resolution |
|-------|-------|------------|
| `Key Vault soft-deleted` | Previous deployment deleted vault | Script auto-recovers or purges |
| `OIDC issuer not found` | AKS not fully provisioned | Wait for AKS to be Running state |
| `DNS challenge failed` | Federated identity not configured | Check dnsIdentity.bicep outputs |
| `Image pull failed` | ACR not attached or no GHCR token | Verify imagePullSecrets and ACR role |

### Debug Commands

```bash
# Check AKS OIDC issuer
az aks show --name ameide --resource-group Ameide --query oidcIssuerProfile

# Check Key Vault access
az keyvault secret list --vault-name <vault-name>

# Check DNS zone delegation
dig NS dev.ameide.io

# Check cert-manager logs
kubectl -n cert-manager logs -l app=cert-manager

# Check ArgoCD sync status
argocd app get <app-name> --show-operation
```

## References

- [Azure AKS Documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [Bicep Documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [cert-manager Azure DNS](https://cert-manager.io/docs/configuration/acme/dns01/azuredns/)
- [Azure Workload Identity](https://azure.github.io/azure-workload-identity/)
