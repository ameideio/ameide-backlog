# 452: Vault RBAC Isolation

> **Note:** This document (452) covers Vault RBAC. For observability namespace isolation, see [452-observability-namespace-isolation.md](452-observability-namespace-isolation.md).

> **Cross-References (Deployment Architecture Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | Per-environment Vault deployment |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Phase 150/155 (vault-core/bootstrap) |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Chart at `sources/charts/third_party/hashicorp/vault/` |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | Client-patcher writes secrets to Vault |

## Status: Implemented

## Problem

HashiCorp Vault's agent-injector creates cluster-scoped resources (ClusterRoleBinding, MutatingWebhookConfiguration) that conflict when multiple Vault instances are deployed across environments. Similar to cert-manager (see [448](448-cert-manager-workload-identity.md)), the "last sync wins" behavior causes:

1. **ClusterRoleBindings only reference one namespace** - staging/prod vault-agent-injector pods lose RBAC permissions
2. **MutatingWebhookConfiguration points to wrong service** - pod injection fails in other environments

**Symptoms:**
- `vault-agent-injector-binding` only references the last-synced environment's ServiceAccount
- `vault-agent-injector-cfg` webhook points to one namespace's service
- Pod injection silently fails in environments that weren't synced last

## Solution

Apply per-environment naming via `fullnameOverride` to create unique cluster-scoped resources:

| Environment | fullnameOverride | ClusterRoleBinding | MutatingWebhookConfig |
|-------------|------------------|--------------------|-----------------------|
| dev | `vault-core-dev` | `vault-core-dev-agent-injector-binding-ameide-dev` | `vault-core-dev-agent-injector-cfg` |
| staging | `vault-core-staging` | `vault-core-staging-agent-injector-binding-ameide-staging` | `vault-core-staging-agent-injector-cfg` |
| production | `vault-core-prod` | `vault-core-prod-agent-injector-binding-ameide-prod` | `vault-core-prod-agent-injector-cfg` |

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ ameide-dev namespace                                                             │
│ ├── StatefulSet: vault-core-dev                                                 │
│ ├── Deployment: vault-core-dev-agent-injector                                   │
│ ├── Service: vault-core-dev (ClusterIP:8200)                                    │
│ ├── Service: vault-core-dev-agent-injector-svc (ClusterIP:443)                  │
│ └── Certificate: vault-core-dev-injector-tls                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ameide-staging namespace                                                         │
│ ├── StatefulSet: vault-core-staging                                             │
│ ├── Deployment: vault-core-staging-agent-injector                               │
│ ├── Service: vault-core-staging (ClusterIP:8200)                                │
│ └── ...                                                                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ameide-prod namespace                                                            │
│ ├── StatefulSet: vault-core-prod                                                │
│ ├── Deployment: vault-core-prod-agent-injector                                  │
│ ├── Service: vault-core-prod (ClusterIP:8200)                                   │
│ └── ...                                                                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│ Cluster-scoped resources (unique per environment)                                │
│ ├── ClusterRole: vault-core-dev-agent-injector-clusterrole                      │
│ ├── ClusterRole: vault-core-staging-agent-injector-clusterrole                  │
│ ├── ClusterRole: vault-core-prod-agent-injector-clusterrole                     │
│ ├── ClusterRoleBinding: vault-core-dev-agent-injector-binding-ameide-dev        │
│ ├── ClusterRoleBinding: vault-core-staging-agent-injector-binding-ameide-staging│
│ ├── ClusterRoleBinding: vault-core-prod-agent-injector-binding-ameide-prod      │
│ ├── MutatingWebhookConfiguration: vault-core-dev-agent-injector-cfg             │
│ ├── MutatingWebhookConfiguration: vault-core-staging-agent-injector-cfg         │
│ └── MutatingWebhookConfiguration: vault-core-prod-agent-injector-cfg            │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Implementation Details

### Per-Environment Values Files

**`sources/values/env/{env}/foundation/foundation-vault-core.yaml`:**
```yaml
# Per-environment naming for complete RBAC isolation
fullnameOverride: vault-core-{env}  # vault-core-dev, vault-core-staging, vault-core-prod

injector:
  # Per-environment TLS certificate from vault-webhook-certs
  certs:
    secretName: vault-core-{env}-injector-tls
  webhook:
    annotations:
      cert-manager.io/inject-ca-from: ameide-{env}/vault-core-{env}-injector-tls
```

### Dynamic Service References

Components that reference Vault's service must derive the name from the namespace:

**SecretStore** (`sources/values/_shared/foundation/foundation-vault-secret-store.yaml`):
```yaml
{{- $envSuffix := trimPrefix "ameide-" .Release.Namespace }}
server: "http://vault-core-{{ $envSuffix }}.{{ .Release.Namespace }}.svc.cluster.local:8200"
```

**vault-bootstrap CronJob** (`sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml`):
```yaml
{{- $envSuffix := trimPrefix "ameide-" .Release.Namespace }}
- name: VAULT_ADDR
  value: "http://vault-core-{{ $envSuffix }}.{{ .Release.Namespace }}.svc.cluster.local:8200"
```

### TLS Certificate Configuration

Each environment needs its own TLS certificate for the agent-injector webhook:

**`sources/values/env/{env}/foundation/foundation-vault-webhook-certs.yaml`:**
```yaml
servingCertificate:
  name: vault-core-{env}-injector-tls
  secretName: vault-core-{env}-injector-tls
  commonName: vault-core-{env}-agent-injector-svc.ameide-{env}.svc
  dnsNames:
    - vault-core-{env}-agent-injector-svc
    - vault-core-{env}-agent-injector-svc.ameide-{env}.svc
    - vault-core-{env}-agent-injector-svc.ameide-{env}.svc.cluster.local
```

## Verification

```bash
# Check per-environment ClusterRoleBindings
kubectl get clusterrolebinding -o custom-columns='NAME:.metadata.name,SA:.subjects[*].name,NS:.subjects[*].namespace' | grep vault-core

# Check per-environment MutatingWebhookConfigurations
kubectl get mutatingwebhookconfiguration | grep vault-core

# Verify each webhook points to correct namespace service
for env in dev staging prod; do
  kubectl get mutatingwebhookconfiguration vault-core-$env-agent-injector-cfg \
    -o jsonpath='{.webhooks[0].clientConfig.service.namespace}/{.webhooks[0].clientConfig.service.name}'
  echo
done

# Verify vault status in each environment
for ns in ameide-dev ameide-staging ameide-prod; do
  env="${ns#ameide-}"
  kubectl exec -n $ns vault-core-$env-0 -- vault status | grep -E "Initialized|Sealed"
done

# Verify SecretStores are valid
kubectl get secretstores -A -o wide
```

## Migration Notes

**WARNING:** Changing `fullnameOverride` creates a NEW StatefulSet with a new PVC. This is a **disruptive change** that:
1. Creates new vault pods with new persistent storage
2. Requires vault re-initialization
3. Loses existing vault data unless migrated

For production migrations:
1. Backup vault data before applying changes
2. Export secrets from old vault
3. Apply changes and wait for new vault to initialize
4. Re-import secrets or run vault-bootstrap to sync from Azure Key Vault

## Files Modified

| File | Purpose |
|------|---------|
| `sources/values/env/dev/foundation/foundation-vault-core.yaml` | Per-env fullnameOverride and injector TLS |
| `sources/values/env/staging/foundation/foundation-vault-core.yaml` | Per-env fullnameOverride and injector TLS |
| `sources/values/env/production/foundation/foundation-vault-core.yaml` | Per-env fullnameOverride and injector TLS |
| `sources/values/_shared/foundation/foundation-vault-core.yaml` | Removed hardcoded TLS refs |
| `sources/values/_shared/foundation/foundation-vault-secret-store.yaml` | Dynamic service name from namespace |
| `sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml` | Dynamic VAULT_ADDR from namespace |
| `sources/values/env/dev/foundation/foundation-vault-webhook-certs.yaml` | Per-env TLS certificate |
| `sources/values/env/staging/foundation/foundation-vault-webhook-certs.yaml` | Per-env TLS certificate |
| `sources/values/env/production/foundation/foundation-vault-webhook-certs.yaml` | Per-env TLS certificate |

## Related

- [446: Namespace Isolation](446-namespace-isolation.md) - Environment isolation architecture
- [447: Waves v3 Cluster-Scoped Operators](447-waves-v3-cluster-scoped-operators.md) - Dual ApplicationSet model
- [448: Cert-Manager Workload Identity](448-cert-manager-workload-identity.md) - Same pattern for cert-manager
- [451: Secrets Management](451-secrets-management.md) - Vault secrets flow architecture

## Changelog

- **2025-12-05**: Implemented per-environment Vault RBAC isolation with `fullnameOverride` pattern
