# 451 – Secrets Management Architecture

**Status**: Implemented
**Created**: 2025-12-05
**Related**: [444-terraform.md](444-terraform.md), [449-per-environment-infrastructure.md](449-per-environment-infrastructure.md), [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md), [452-vault-rbac-isolation.md](452-vault-rbac-isolation.md), [462-secrets-origin-classification.md](462-secrets-origin-classification.md)

---

## Overview

This document describes the end-to-end secrets flow from developer `.env` files to Kubernetes Secrets in environment namespaces. The architecture uses Azure Key Vault as the external source of truth for **external/third-party secrets**, HashiCorp Vault as the internal secret store, and ExternalSecrets Operator to sync secrets to Kubernetes.

> **Important:** This flow applies only to **external/third-party secrets** (API keys, registry tokens, etc.).
> **Cluster-managed secrets** (database credentials, Keycloak client secrets) have different authorities.
> See [462-secrets-origin-classification.md](462-secrets-origin-classification.md) for the full taxonomy.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Developer Workstation                                │
│                                                                              │
│   .env / .env.local                                                         │
│   ┌────────────────────┐                                                    │
│   │ GHCR_TOKEN=ghp_xxx │                                                    │
│   │ GHCR_USER=myuser   │                                                    │
│   │ LANGFUSE_SECRET=xx │                                                    │
│   └────────────────────┘                                                    │
│              │                                                               │
│              ▼                                                               │
│   infra/scripts/deploy.sh                                                   │
│   ├── parse_env_to_tfvars()                                                 │
│   │   └── Sanitize: GHCR_TOKEN → ghcr-token                                │
│   └── terraform apply -var-file=env_secrets.tfvars.json                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Azure                                           │
│                                                                              │
│   Azure Key Vault (ameidekv)                                                │
│   ┌────────────────────────────┐                                            │
│   │ ghcr-token = ghp_xxx       │                                            │
│   │ ghcr-user = myuser         │                                            │
│   │ langfuse-secret = xx       │                                            │
│   └────────────────────────────┘                                            │
│              │                                                               │
│              │ Workload Identity (OIDC)                                      │
│              │ ameide-vault-bootstrap-mi                                     │
│              │ Federated credentials:                                        │
│              │ ├── vault-bootstrap-dev                                       │
│              │ ├── vault-bootstrap-staging                                   │
│              │ └── vault-bootstrap-prod                                      │
│              ▼                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Per-Environment (ameide-dev)                      │   │
│  │                                                                      │   │
│  │  vault-bootstrap CronJob                                            │   │
│  │  ├── Fetches secrets from Azure Key Vault                           │   │
│  │  ├── Overwrites fixture defaults with real values                   │   │
│  │  └── Writes to HashiCorp Vault: secret/ghcr-token                   │   │
│  │                                                                      │   │
│  │  HashiCorp Vault (foundation-vault-core)                            │   │
│  │  ┌────────────────────────────┐                                     │   │
│  │  │ secret/ghcr-token          │                                     │   │
│  │  │ secret/ghcr-user           │                                     │   │
│  │  │ secret/langfuse-secret     │                                     │   │
│  │  └────────────────────────────┘                                     │   │
│  │              │                                                       │   │
│  │              │ Kubernetes Auth                                       │   │
│  │              ▼                                                       │   │
│  │  SecretStore (ameide-vault)                                         │   │
│  │  └── Points to foundation-vault-core.ameide-dev.svc.cluster.local   │   │
│  │              │                                                       │   │
│  │              ▼                                                       │   │
│  │  ExternalSecrets                                                    │   │
│  │  ├── ghcr-pull-sync → ghcr-pull (dockerconfigjson)                  │   │
│  │  ├── grafana-admin-credentials-sync → grafana-admin-credentials     │   │
│  │  └── ... (all other secrets)                                        │   │
│  │              │                                                       │   │
│  │              ▼                                                       │   │
│  │  Kubernetes Secrets                                                 │   │
│  │  ┌────────────────────────────┐                                     │   │
│  │  │ ghcr-pull (image pull)     │                                     │   │
│  │  │ grafana-admin-credentials  │                                     │   │
│  │  │ minio-root-credentials     │                                     │   │
│  │  └────────────────────────────┘                                     │   │
│  │              │                                                       │   │
│  │              ▼                                                       │   │
│  │  Pods (using secrets)                                               │   │
│  │  ├── imagePullSecrets: ghcr-pull                                    │   │
│  │  └── env: GRAFANA_ADMIN_PASSWORD                                    │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  (Identical architecture in ameide-staging and ameide-prod)                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. deploy.sh - Secret Parsing

**File**: `infra/scripts/deploy.sh`

The `parse_env_to_tfvars()` function:
1. Reads `.env` and `.env.local` files
2. Sanitizes keys: `GHCR_TOKEN` → `ghcr-token` (lowercase, underscores/dots to dashes)
3. Creates temporary `.tfvars.json` file with `env_secrets` variable
4. Terraform creates `azurerm_key_vault_secret` resources

```bash
# Key sanitization rules (Azure Key Vault requirements):
# - Lowercase only
# - Replace _ with -
# - Replace . with -
sanitized_key="$(echo "${key}" | tr '[:upper:]' '[:lower:]' | sed -e 's/[_.]/-/g')"
```

### 2. Terraform - Key Vault Secrets

**File**: `infra/terraform/azure/main.tf`

```hcl
resource "azurerm_key_vault_secret" "env_secrets" {
  for_each = nonsensitive(toset(keys(var.env_secrets)))

  name         = each.key
  value        = var.env_secrets[each.key]
  key_vault_id = module.keyvault.id
}
```

### 3. Terraform - Vault Bootstrap Identity

**File**: `infra/terraform/azure/main.tf`

Per-environment federated credentials allow vault-bootstrap to authenticate:

```hcl
vault_bootstrap_subjects = {
  dev     = "system:serviceaccount:ameide-dev:foundation-vault-bootstrap-vault-bootstrap"
  staging = "system:serviceaccount:ameide-staging:foundation-vault-bootstrap-vault-bootstrap"
  prod    = "system:serviceaccount:ameide-prod:foundation-vault-bootstrap-vault-bootstrap"
}

module "vault_bootstrap_identity" {
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

### 4. vault-bootstrap CronJob

**File**: `sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml`

The CronJob runs in each environment namespace and:
1. Uses Azure Workload Identity to authenticate to Key Vault
2. Fetches secrets and overwrites fixture defaults
3. Writes to HashiCorp Vault KV store

Key configuration:
```yaml
azure:
  keyVault:
    uri: "https://ameidekv.vault.azure.net/"
    secretPrefix: ""  # No prefix - matches sanitized .env keys
```

### 5. SecretStore (Per-Environment)

**File**: `sources/values/_shared/foundation/foundation-vault-secret-store.yaml`

Each environment has its own SecretStore pointing to local Vault:

```yaml
# ameide-dev
SecretStore/ameide-vault → foundation-vault-core.ameide-dev.svc.cluster.local:8200

# ameide-staging
SecretStore/ameide-vault → foundation-vault-core.ameide-staging.svc.cluster.local:8200

# ameide-prod
SecretStore/ameide-vault → foundation-vault-core.ameide-prod.svc.cluster.local:8200
```

### 6. ExternalSecrets

ExternalSecrets sync from Vault to Kubernetes Secrets. Example:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: ghcr-pull-sync
  namespace: ameide-dev
spec:
  secretStoreRef:
    kind: SecretStore
    name: ameide-vault
  target:
    name: ghcr-pull
    template:
      type: kubernetes.io/dockerconfigjson
  data:
    - secretKey: .dockerconfigjson
      remoteRef:
        key: ghcr-token
        property: value
```

---

## Secret Key Mapping (External/Third-Party Only)

> **Note:** This table shows **external/third-party secrets** that flow through Azure KV.
> **Cluster-managed secrets** (database passwords, Keycloak client secrets) do NOT use this flow.
> See [462-secrets-origin-classification.md](462-secrets-origin-classification.md) for the cluster-managed patterns.

| .env Variable | Azure Key Vault | HashiCorp Vault | K8s Secret |
|---------------|-----------------|-----------------|------------|
| `GHCR_TOKEN` | `ghcr-token` | `secret/ghcr-token` | `ghcr-pull` |
| `GHCR_USER` | `ghcr-user` | `secret/ghcr-user` | `ghcr-pull` |
| `LANGFUSE_SECRET_KEY` | `langfuse-secret-key` | `secret/langfuse-secret-key` | `langfuse-secrets` |
| `ANTHROPIC_API_KEY` | `anthropic-api-key` | `secret/anthropic-api-key` | `inference-api-credentials` |
| `OPENAI_API_KEY` | `openai-api-key` | `secret/openai-api-key` | `inference-api-credentials` |

### Secrets NOT in Azure KV (Cluster-Managed)

These secrets are generated in-cluster and should NOT be added to `.env` or Azure KV:

| Secret | Authority | See |
|--------|-----------|-----|
| `postgres-ameide-auth`, `*-db-credentials` | CNPG Operator | [412](./412-cnpg-owned-postgres-greds.md) |
| `platform-app-master-client` | Keycloak (via client-patcher) | [462](./462-secrets-origin-classification.md) |
| `minio-root-credentials` | Helm `randAlphaNum` | [462](./462-secrets-origin-classification.md) |
| `grafana-admin-credentials` | Helm `randAlphaNum` | [462](./462-secrets-origin-classification.md) |

---

## Troubleshooting

### Secrets Not Syncing

1. **Check vault-bootstrap logs**:
   ```bash
   kubectl logs -n ameide-dev -l job-name=foundation-vault-bootstrap-vault-bootstrap-<id>
   ```

2. **Verify Azure Key Vault secrets exist**:
   ```bash
   az keyvault secret list --vault-name ameidekv -o table
   ```

3. **Check federated identity credentials**:
   ```bash
   az identity federated-credential list \
     --identity-name ameide-vault-bootstrap-mi \
     --resource-group Ameide -o table
   ```

4. **Verify Vault has real values** (not placeholders):
   ```bash
   ROOT_TOKEN=$(kubectl get secret -n ameide-dev vault-auto-credentials -o jsonpath='{.data.root-token}' | base64 -d)
   kubectl exec -n ameide-dev foundation-vault-core-0 -- sh -c "VAULT_TOKEN=$ROOT_TOKEN vault kv get secret/ghcr-token"
   ```

5. **Check ExternalSecret status**:
   ```bash
   kubectl get externalsecrets -n ameide-dev
   kubectl describe externalsecret ghcr-pull-sync -n ameide-dev
   ```

### Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `ImagePullBackOff` | ghcr-pull secret has placeholder | Check vault-bootstrap federated identity |
| vault-bootstrap fails silently | Missing `AZURE_FEDERATED_TOKEN_FILE` | Verify workload identity webhook running |
| SecretStore not Ready | Vault not initialized | Wait for vault-bootstrap to complete |
| ExternalSecret shows `SecretSyncError` | Vault path incorrect | Check `secretPrefix` in values |
| SecretStore `InvalidProviderConfig` | Vault sealed | Unseal Vault (see below) |
| SecretStore `permission denied` (403) | Kubernetes auth not configured | Run vault-bootstrap or reconfigure manually |

### Vault Sealed (503 Error)

If Vault is sealed, the SecretStore shows `InvalidProviderConfig` with error:
```
unable to log in with Kubernetes auth: Error making API request.
Code: 503. Errors: * Vault is sealed
```

**Resolution:**
```bash
# Get unseal key (replace <env> with dev/staging/prod)
UNSEAL_KEY=$(kubectl get secret -n ameide-<env> vault-auto-credentials -o jsonpath='{.data.unseal-key}' | base64 -d)

# Unseal vault
kubectl exec -n ameide-<env> vault-core-<env>-0 -- vault operator unseal "$UNSEAL_KEY"

# Verify
kubectl exec -n ameide-<env> vault-core-<env>-0 -- vault status
```

### Kubernetes Auth Permission Denied (403 Error)

If SecretStore shows `InvalidProviderConfig` with error:
```
Code: 403. Errors: * permission denied
```

This indicates Vault's Kubernetes auth is not properly configured.

**Resolution:**
```bash
# Get root token
ROOT_TOKEN=$(kubectl get secret -n ameide-<env> vault-auto-credentials -o jsonpath='{.data.root-token}' | base64 -d)

# Reconfigure kubernetes auth
kubectl exec -n ameide-<env> vault-core-<env>-0 -- sh -c "VAULT_TOKEN=$ROOT_TOKEN vault write auth/kubernetes/config kubernetes_host='https://kubernetes.default.svc' disable_iss_validation=true"

# Force SecretStore reconciliation
kubectl annotate secretstore ameide-vault -n ameide-<env> force-refresh=$(date +%s) --overwrite
```

### Force ExternalSecrets Refresh

After fixing SecretStore issues, ExternalSecrets may be cached as failed:
```bash
# Refresh all ExternalSecrets in a namespace
for es in $(kubectl get externalsecret -n ameide-<env> -o name); do
  kubectl annotate "$es" -n ameide-<env> force-refresh="$(date +%s)" --overwrite
done
```

---

## Files Reference

| Component | Path |
|-----------|------|
| Deploy script | `infra/scripts/deploy.sh` |
| Terraform secrets | `infra/terraform/azure/main.tf` (lines 280-290) |
| Terraform identity | `infra/terraform/azure/main.tf` (vault_bootstrap_identity) |
| vault-bootstrap CronJob | `sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml` |
| vault-bootstrap values (shared) | `sources/values/_shared/foundation/foundation-vault-bootstrap.yaml` |
| vault-bootstrap values (dev) | `sources/values/dev/foundation/foundation-vault-bootstrap.yaml` |
| SecretStore template | `sources/values/_shared/foundation/foundation-vault-secret-store.yaml` |

---

## Validation

```bash
# 1. Verify Azure Key Vault secrets
az keyvault secret list --vault-name ameidekv --query "[].name" -o tsv

# 2. Verify federated credentials
az identity federated-credential list \
  --identity-name ameide-vault-bootstrap-mi \
  --resource-group Ameide \
  --query "[].{name:name, subject:subject}" -o table

# 3. Verify Vault has real values
kubectl exec -n ameide-dev foundation-vault-core-0 -- \
  sh -c "VAULT_TOKEN=\$(cat /vault/secrets/root-token) vault kv get secret/ghcr-token"

# 4. Verify ExternalSecrets are synced
kubectl get externalsecrets -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,READY:.status.conditions[0].status

# 5. Verify K8s secrets exist
kubectl get secrets -n ameide-dev -l app.kubernetes.io/managed-by=external-secrets
```
