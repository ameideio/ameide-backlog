# 450 – ArgoCD Service Issues Inventory

**Status**: Partially Resolved (CI/CD blocking staging/prod)
**Created**: 2025-12-04
**Updated**: 2025-12-05
**Commit**: `6c805de fix(446): remove tolerations overrides, inherit from globals`
**Related**: [442-environment-isolation.md](442-environment-isolation.md), [445-argocd-namespace-isolation.md](445-argocd-namespace-isolation.md), [446-namespace-isolation.md](446-namespace-isolation.md), [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), [451-secrets-management.md](451-secrets-management.md), [456-ghcr-mirror.md](456-ghcr-mirror.md)

---

## Summary

Total applications: 200
- **Healthy/Synced**: ~83
- **Degraded**: 97 (across dev/staging/production)
- **Missing**: 6
- **OutOfSync**: 20+
- **Progressing**: 12

---

## P0 – Blocking Issues (Root Cause Chain)

### 1. ~~Vault Pods Running with Outdated Revision~~ ✅ RESOLVED

**Status**: ✅ RESOLVED - Pods deleted and recreated with correct tolerations

All Vault pods now running on correct environment nodes:
```
ameide-dev       foundation-vault-core-0   Running   aks-dev-21365390-vmss000001
ameide-staging   foundation-vault-core-0   Running   aks-staging-92300291-vmss000002
ameide-prod      foundation-vault-core-0   Running   aks-prod-23578083-vmss000000
```

### 1b. ~~Dev Node Volume Limit~~ ✅ RESOLVED

**Status**: ✅ RESOLVED - Second dev node now Ready

---

### 2. ~~ClusterSecretStore Architecture Mismatch~~ ✅ RESOLVED

**Status**: ✅ RESOLVED - Refactored to per-environment SecretStores (Option B)

**Original Issue**: ClusterSecretStore pointed to non-existent `vault` namespace, but Vault is deployed per-environment per backlog/446.

**Fix Applied**:
1. Replaced cluster-wide `ClusterSecretStore` with per-environment `SecretStore`
2. Each SecretStore points to local Vault: `foundation-vault-core.{{ .Release.Namespace }}.svc.cluster.local`
3. Updated 97 files to reference `SecretStore` instead of `ClusterSecretStore`
4. Deleted old `ClusterSecretStore` from cluster

**New Architecture** (aligned with 446):
```
ameide-dev namespace:
  └── SecretStore/ameide-vault → foundation-vault-core.ameide-dev.svc.cluster.local:8200

ameide-staging namespace:
  └── SecretStore/ameide-vault → foundation-vault-core.ameide-staging.svc.cluster.local:8200

ameide-prod namespace:
  └── SecretStore/ameide-vault → foundation-vault-core.ameide-prod.svc.cluster.local:8200
```

**Files Changed**:
- `sources/values/_shared/foundation/foundation-vault-secret-store.yaml` - Main SecretStore template
- `sources/values/_shared/foundation/foundation-vault-bootstrap.yaml` - Per-env Vault address
- `sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml` - Use Release.Namespace
- `sources/charts/shared-values/tests/secrets-smoke.yaml` - Fix ClusterSecretStore refs
- All ExternalSecret references updated from `ClusterSecretStore` to `SecretStore`

**Additional Fix (2025-12-05)**:
- Removed `namespace` from serviceAccountRef (webhook validation error)
- Use `default` SA in each environment namespace
- Updated vault-bootstrap to accept SAs from `ameide-dev,ameide-staging,ameide-prod`

**Current State**: SecretStores created and synced:
```
kubectl get secretstores -A
NAMESPACE        NAME           AGE   STATUS   CAPABILITIES   READY
ameide-dev       ameide-vault   ...
ameide-prod      ameide-vault   ...
ameide-staging   ameide-vault   ...
```

**Next**: Vault initialization/unsealing required for SecretStores to become Ready

---

### 3. PVCs Pending

**Symptom**: Multiple PersistentVolumeClaims stuck in Pending

| PVC | Namespace | Storage Class | Reason |
|-----|-----------|---------------|--------|
| `data-foundation-vault-core-0` | ameide-dev | default | Pod not scheduled |
| `data-0-kafka-kafka-pool-0` | ameide-dev | managed-csi-premium | Pod not scheduled |
| `data-clickhouse-*` | ameide-dev | managed-csi | Pod not scheduled |
| `postgres-ameide-1` | ameide-dev | managed-csi-premium | Pod not scheduled |
| `storage-platform-loki-0` | ameide-dev | managed-csi | Pod not scheduled |

**Root Cause**: PVCs waiting for pods to be scheduled (WaitForFirstConsumer)

---

## P1 – Pod Failures (Blocked by P0)

### CreateContainerConfigError (Missing Secrets)

| Pod | Namespace | Missing Secret |
|-----|-----------|----------------|
| `data-minio-*` | ameide-dev | `minio-root-credentials` |
| `platform-grafana-*` | ameide-dev | `grafana-admin-credentials` |
| `pgadmin-*` | ameide-dev | `pgadmin-bootstrap-credentials` |
| `plausible-plausible-*` | ameide-dev | `plausible-secret` |

### ~~ImagePullBackOff (Missing ghcr-pull-secret)~~ ✅ RESOLVED

| Pod | Namespace | Image |
|-----|-----------|-------|
| `transformation-*` | ameide-dev | `ghcr.io/ameideio/transformation:dev` |

**Root Cause (Multi-Layer)**:
1. ClusterSecretStore broken → fixed with per-environment SecretStores
2. vault-bootstrap couldn't fetch secrets from Azure Key Vault → federated identity credentials were missing for per-environment namespaces

**Fix Applied (2025-12-05)**:
1. Created per-environment federated identity credentials for vault-bootstrap identity:
   - `vault-bootstrap-dev` → `system:serviceaccount:ameide-dev:foundation-vault-bootstrap-vault-bootstrap`
   - `vault-bootstrap-staging` → `system:serviceaccount:ameide-staging:foundation-vault-bootstrap-vault-bootstrap`
   - `vault-bootstrap-prod` → `system:serviceaccount:ameide-prod:foundation-vault-bootstrap-vault-bootstrap`
2. Updated Terraform to manage these federated credentials automatically
3. Removed `env-` prefix from Azure Key Vault secret names (aligned with Helm values)

**Result**: Real credentials now flow: `.env` → Azure Key Vault → HashiCorp Vault → ExternalSecrets → K8s Secrets

See [451-secrets-management.md](451-secrets-management.md) for complete secrets architecture

---

## P2 – OutOfSync Applications

### ~~Environment-Specific Apps~~ ✅ RESOLVED

| App | Status | Resolution |
|-----|--------|------------|
| `*-foundation-cert-manager` | ✅ DELETED | Orphaned apps deleted - cert-manager is cluster-scoped per 446 |
| `*-foundation-coredns-config` | Syncing | Will auto-sync |
| `*-foundation-cnpg-monitoring` | Synced | Fixed |
| `*-foundation-vault-bootstrap` | Synced | Fixed |
| `*-platform-envoy-gateway` | Syncing | Will auto-sync |
| `*-platform-grafana` | Syncing | Will auto-sync |
| `*-platform-loki` | Syncing | Will auto-sync |
| `*-platform-prometheus` | Syncing | Will auto-sync |

**Per-environment cert-manager apps removed** (2025-12-05):
- `dev-foundation-cert-manager`, `staging-foundation-cert-manager`, `production-foundation-cert-manager`
- These were orphaned applications with no component file (deleted in commit `92efae5`)
- Per 446, cert-manager should be cluster-scoped only (`cluster-cert-manager` in `cert-manager` namespace)
- Orphaned resources cleaned up from all 3 environment namespaces

### Cluster-Scoped Apps

| App | Reason |
|-----|--------|
| `argocd-cert-manager` | ArgoCD-managed cert-manager |
| `cluster-cert-manager` | Cluster cert-manager operator |

---

## P3 – Progressing Applications

Apps still reconciling (may self-heal once P0 fixed):

- `{env}-data-clickhouse` - ClickHouse installation
- `{env}-data-kafka-cluster` - Kafka cluster startup
- `{env}-data-temporal-namespace-bootstrap` - Temporal namespaces
- `{env}-platform-keycloak` - Keycloak startup
- `{env}-platform-postgres-clusters` - CNPG cluster initialization

---

## Controller Noise – DiffFromCache Errors

**Symptom**: `kubectl logs -n argocd argocd-application-controller-0 --since=30m | grep 'DiffFromCache error'` emits dozens of messages such as `DiffFromCache error: error getting managed resources for app production-platform-envoy-crds: cache: key is missing`.

**Root Cause**: Per the Argo CD operator manual (`docs/operator-manual/server-commands/argocd-application-controller.md`), the application controller stores managed-resource diffs in a one-hour cache (`--app-state-cache-expiration`, default `1h0m0s`). With ~200 apps reconciling every three minutes, entries naturally expire every hour and the next reconciliation logs an error when the cache miss occurs—even though the controller immediately recomputes the diff and continues.

**Fix Applied**: Extended the cache TTL to six hours by setting `controller.env[ARGOCD_APP_STATE_CACHE_EXPIRATION]=6h` in `sources/values/common/argocd.yaml`. Helm installs/`tools/bootstrap/bootstrap-v2.sh` now render the updated environment variable so the controller retains app-state cache entries between hourly polls.

**Validation**:
1. `helm upgrade --install argocd ... -f sources/values/common/argocd.yaml` (or rerun bootstrap) to roll out the new env var.
2. `kubectl logs -n argocd argocd-application-controller-0 --since=2h | grep 'DiffFromCache error'` should show at most one entry immediately after rollout; subsequent hourly cache sweeps no longer spam errors.

---

## Node Topology

| Node Pool | Count | Taint | Capacity |
|-----------|-------|-------|----------|
| system | 2 | `CriticalAddonsOnly=true:NoSchedule` | 1.9 CPU, 6GB RAM |
| dev | 1 | `ameide.io/environment=dev:NoSchedule` | 3.8 CPU, 14GB RAM |
| staging | 1 | `ameide.io/environment=staging:NoSchedule` | 3.8 CPU, 14GB RAM |
| prod | 3 | `ameide.io/environment=production:NoSchedule` | 3.8 CPU, 14GB RAM each |

---

## Resolution Priority

1. ~~**Fix Vault tolerations in values**~~ ✅ Already configured correctly
2. ~~**Wait for dev autoscaler**~~ ✅ Second dev node now Ready
3. ~~**Delete outdated vault pods**~~ ✅ Pods recreated, now Running
4. ~~**Fix SecretStore architecture**~~ ✅ Refactored to per-environment SecretStores
5. ~~**Commit and push changes**~~ ✅ Pushed - ArgoCD synced
6. ~~**Vault initializes**~~ ✅ Vault pods running, SecretStores Ready
7. ~~**SecretStores connect**~~ ✅ ExternalSecrets syncing
8. ~~**Secrets available**~~ ✅ Most secrets created (ghcr-pull, minio, etc.)
9. ~~**Delete orphaned per-env cert-manager**~~ ✅ Apps and resources deleted
10. ~~**Fix tolerations overrides**~~ ✅ Removed empty overrides from 11 apps shared values
11. ~~**Apps auto-syncing**~~ ✅ Pods now scheduling on correct nodes
12. ~~**Fix Vault sealed/K8s auth issues**~~ ✅ All environments operational

**Current Status**: ⚠️ **PARTIALLY RESOLVED** - Dev healthy, staging/prod have CI/CD-related issues.

### Remaining Issues (2025-12-05)

| Issue | Environment | Root Cause | Resolution |
|-------|-------------|------------|------------|
| `www-ameide-platform` ImagePullBackOff | staging, prod | `main` tag not pushed to GHCR | CI/CD pipeline needs to push `main` tag |
| `plausible-seed` ImagePullBackOff | staging, prod | Missing `main` tag | CI/CD pipeline issue |
| `workflows-runtime` CrashLoopBackOff | staging, prod | Likely missing `main` tag | CI/CD pipeline issue |
| Bootstrap jobs Pending | staging, prod | Helm hooks need ArgoCD re-sync | Delete jobs and sync apps |

**Image Tagging Strategy**:
- `dev` environment → `dev` tag (working)
- `staging`/`production` environments → `main` tag (not pushed by CI/CD yet)

The GitOps configuration is correct. The issue is that the source repositories' CI/CD pipelines are not pushing `main` tagged images to GHCR.

### Tolerations Fix (2025-12-05)

**Root Cause**: Shared values files explicitly set `nodeSelector: {}` and `tolerations: []`,
overriding the globals.yaml inherited values.

**Fix Applied** (`6c805de`):
- Removed explicit empty values from 11 apps shared values files:
  - `agents.yaml`, `agents-runtime.yaml`, `graph.yaml`, `inference.yaml`
  - `inference-gateway.yaml`, `platform.yaml`, `threads.yaml`
  - `workflows.yaml`, `workflows-runtime.yaml`
  - `www-ameide.yaml`, `www-ameide-platform.yaml`
- Deleted orphaned `foundation-cert-manager.yaml` values files
- Now tolerations/nodeSelector inherited from `globals.yaml`:
  ```yaml
  nodeSelector:
    ameide.io/pool: dev  # or staging/prod
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"  # or staging/production
      effect: "NoSchedule"
  ```

**Verification**:
```bash
# New pods now assigned to correct nodes:
kubectl get pods -n ameide-dev -o wide | grep agents
# agents-785f678b6c-pw52f   aks-dev-21365390-vmss000001
```

### Vault Sealed/K8s Auth Fix (2025-12-05)

**Symptoms discovered**:
- Staging: Vault sealed (503 error)
- Production: Vault Kubernetes auth permission denied (403 error)
- SecretStores showed `InvalidProviderConfig` in both environments

**Root Causes**:
1. **Staging**: Vault pod restarted and was not auto-unsealed
2. **Production**: Kubernetes auth configuration needed `disable_iss_validation=true`

**Fixes Applied**:
```bash
# 1. Unseal staging Vault
UNSEAL_KEY=$(kubectl get secret -n ameide-staging vault-auto-credentials -o jsonpath='{.data.unseal-key}' | base64 -d)
kubectl exec -n ameide-staging vault-core-staging-0 -- vault operator unseal "$UNSEAL_KEY"

# 2. Reconfigure Kubernetes auth in both environments
for env in staging prod; do
  ROOT_TOKEN=$(kubectl get secret -n ameide-$env vault-auto-credentials -o jsonpath='{.data.root-token}' | base64 -d)
  kubectl exec -n ameide-$env vault-core-$env-0 -- sh -c "VAULT_TOKEN=$ROOT_TOKEN vault write auth/kubernetes/config kubernetes_host='https://kubernetes.default.svc' disable_iss_validation=true"
done

# 3. Force SecretStore reconciliation
kubectl annotate secretstore ameide-vault -n ameide-staging force-refresh=$(date +%s) --overwrite
kubectl annotate secretstore ameide-vault -n ameide-prod force-refresh=$(date +%s) --overwrite

# 4. Force ExternalSecrets refresh
for ns in ameide-staging ameide-prod; do
  for es in $(kubectl get externalsecret -n $ns -o name); do
    kubectl annotate "$es" -n $ns force-refresh="$(date +%s)" --overwrite
  done
done
```

**Result**:
```
NAMESPACE        NAME           STATUS   READY
ameide-dev       ameide-vault   Valid    True
ameide-prod      ameide-vault   Valid    True
ameide-staging   ameide-vault   Valid    True
```

All 21 ExternalSecrets in each environment now syncing successfully.

See [451-secrets-management.md](451-secrets-management.md) for detailed troubleshooting runbook.

### ArgoCD OIDC Client Secret (2025-12-07)

**Status**: ✅ RESOLVED via client-patcher

**Issue**: ArgoCD/Dex OIDC authentication relies on the `argocd-dex-client-secret` in Vault. Previously this was a static fixture value.

**Resolution** (per [462-secrets-origin-classification.md](462-secrets-origin-classification.md)):
1. ArgoCD's `argocd` client is configured in the Keycloak `ameide` realm
2. `client-patcher` Job extracts the Keycloak-generated secret via Admin API
3. Secret written to Vault at `secret/argocd-dex-client-secret`
4. ExternalSecret syncs to `argocd-secret` for Dex consumption

**Configuration**:
```yaml
# Per-env platform-keycloak-realm.yaml
clientPatcher:
  secretExtraction:
    clients:
      - clientId: argocd
        vaultKey: argocd-dex-client-secret
```

**Verification**:
```bash
# Check client-patcher Job completed
kubectl get jobs -n ameide-<env> -l app.kubernetes.io/name=keycloak-realm

# Verify secret in Vault
vault kv get secret/argocd-dex-client-secret

# Test ArgoCD OIDC login
argocd login <server> --sso
```

See [426-keycloak-config-map.md §3.2](426-keycloak-config-map.md) for client-patcher architecture.

---

## Validation Commands

```bash
# Check Vault pods
kubectl get pods -l app.kubernetes.io/name=vault -A

# Check per-environment SecretStores (after sync)
kubectl get secretstores -A

# Check ExternalSecrets status
kubectl get externalsecrets -A | grep -v "True"

# Check pending pods
kubectl get pods -A --field-selector=status.phase=Pending

# Check CreateContainerConfigError pods
kubectl get pods -A | grep CreateContainerConfigError
```
