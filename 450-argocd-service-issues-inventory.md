# 450 – ArgoCD Service Issues Inventory

**Status**: Partially Resolved (CI/CD blocking staging/prod)
**Created**: 2025-12-04
**Updated**: 2025-12-15
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

## Update (2025-12-14): Local GitOps health fixes (devcontainer + k3d)

- Fixed `local-platform-keycloak-realm` being blocked by a failing PreSync hook: `platform-keycloak-realm-client-patcher` is now reproducible (no runtime GitHub tool downloads; Keycloak Admin REST + Kubernetes API patch for rotation ConfigMap).
- Fixed `local-platform-gateway` sync failure on cross-namespace `ReferenceGrant` rendering: ensure core-group `group: ""` is rendered as a string (quoted), not YAML null.
- Verified the local set converges after sync: `local-platform-gateway`, `local-inference`, `local-inference-gateway`, `local-data-pgadmin` reach `Synced/Healthy`.
- Fixed local “red pod noise” from provisioning/bootstrap jobs:
  - MinIO provisioning Job was stuck `ImagePullBackOff` on `ghcr.io/ameideio/mirror/bitnami-os-shell:latest` due to missing `imagePullSecrets` and a single-arch mirror tag.
  - Vault bootstrap Job had a transient failure due to runtime `kubectl` download TLS flakiness; bootstrap jobs should avoid runtime downloads (track under 519).

## Update (2025-12-15): Local data-plane smoke “Last Sync Failed” (ClickHouse name + Redis operator crashloop)

Observed Argo apps that were `Synced/Healthy` but still had a stale **`operationState.phase=Failed`** due to PostSync hook Jobs reaching backoff limits.

- **App:** `local-data-data-plane-smoke`
  - **Failing hooks:** `data-data-plane-smoke-helm-test-jobs-clickhouse`, `data-data-plane-smoke-helm-test-jobs-redis-cache`
- **ClickHouse smoke failure:** the test script hardcoded `ClickHouseInstallation` name `clickhouse`, but the installed CHI is `data-clickhouse`.
  - Symptom: `ClickHouseInstallation 'clickhouse' not found in namespace ameide-local`
  - Root cause: test assumed a fixed name rather than discovering from labels or values.
- **Redis smoke failure:** `RedisFailover/redis` existed, but the Redis pods never became Ready because the Spotahome `redis-operator` was CrashLooping.
  - Symptom: `kubectl wait ... --selector app.kubernetes.io/part-of=redis-failover --timeout=600s` timed out.
  - Operator logs: leader election lease renewal failed with `client rate limiter Wait returned an error: context deadline exceeded`, causing the operator to exit repeatedly.
  - Resulting state: RedisFailover reconciliation was incomplete (no master created; replica config `slaveof 127.0.0.1 6379`).

Remediation approach (GitOps-aligned, no band-aids):
1. Make ClickHouse smoke discover the installation name (or take it from values) instead of hardcoding `clickhouse`.
2. Stabilize `redis-operator` so it stays Running and can reconcile RedisFailover deterministically (avoid lease-renewal crashes under k3d load).
3. Ensure smoke apps always have at least one tracked non-hook manifest so Argo auto-sync can clear stale failed operations (already applied for `helm-test-jobs`).

Follow-up (local):
- If `redis-operator` continues to lose its lease due to apiserver latency, pin it to the k3d control-plane node and run it at `system-cluster-critical` priority to reduce scheduling/CPU jitter during lease renewals.
- If the published operator tag is missing (Spotahome does not publish `v1.3.0` on quay.io), prefer a known-good tag (e.g. `v1.2.4`) or a vetted RC (`v1.3.0-rc1`) for local until a stable release is available.
- If lease updates still time out, bypass the `kubernetes` ClusterIP path for in-cluster clients by pointing controllers at the apiserver endpoint directly (set `KUBERNETES_SERVICE_HOST/PORT` to the `default/kubernetes` endpoint host/port in k3d).
- If leader-election renewals remain fragile under local apiserver write latency, stop depending on the operator for local:
  - Run `data-redis-failover` in a local `standalone` mode (no `RedisFailover` CR).
  - Scale the `cluster-redis-operator` Deployment to `replicas: 0` for local.
  - Override local consumers (e.g. `www-ameide-platform`) to use `redis://redis-master:6379/0` (no Sentinel) explicitly.

## Update (2025-12-15): `local-foundation-external-secrets-ready` “Last Sync Failed” (readiness gate drift)

Observed an Argo app that was `Synced/Healthy` but still showed **`operationState.phase=Failed`** due to a Sync hook Job hitting its backoff limit.

- **App:** `local-foundation-external-secrets-ready`
  - **Failing hook:** `foundation-external-secrets-ready-helm-test-jobs-*`

Root cause (configuration drift):
- The “ready” hook script was checking legacy resource names (`deployment/foundation-external-secrets*`) that do not exist with the current `external-secrets` Helm chart install (`deployment/external-secrets`, `deployment/external-secrets-webhook`, `deployment/external-secrets-cert-controller`).
- The ready hook runs in the environment namespace but checks resources in `external-secrets`, so it also requires cluster-wide RBAC (otherwise `kubectl -n external-secrets ...` is forbidden).

Remediation approach (GitOps-aligned, no band-aids):
1. Update the ready check to validate the *actual* external-secrets deployment names and webhook Service/Endpoints created by the current chart.
2. Set the ready check’s `helm-test-jobs` RBAC to `clusterWide: true` (or move the check into `external-secrets`; prefer explicit RBAC so the check remains per-env but can validate cluster-scoped substrate).

Backlog linkage:
- Reinforces [382-argocd-dependency-hardening.md](300-400/382-argocd-dependency-hardening.md): readiness gates must be accurate and properly ordered, otherwise they produce stale failed operations that hide the real state.

## Update (2025-12-15): Local cluster controllers CrashLooping (Envoy Gateway leader election + kube-state-metrics RBAC mismatch)

Observed cluster-scoped and per-env controllers stuck `Progressing` because their Deployments were CrashLooping:

- **App:** `cluster-gateway`
  - **Pod:** `argocd/envoy-gateway-*` CrashLoopBackOff
  - **Symptom:** leader election lease reads failed with `context deadline exceeded` against the in-cluster apiserver service (`https://10.43.0.1:443/.../leases/...`), then the controller exits (`leader election lost`).
- **App:** `local-platform-prometheus`
  - **Pod:** `ameide-local/platform-prometheus-kube-state-metrics-*` CrashLoopBackOff
  - **Symptom:** RBAC forbids listing cluster-scoped resources (`nodes`, `namespaces`, `storageclasses`, `persistentvolumes`, `volumeattachments`, webhook configs, CSRs), but the container is configured with `--resources=...` that includes those cluster-scoped collectors.
  - **Also observed:** liveness probe flapping (`/livez` timeouts / HTTP 503) during informer cache warmup under local apiserver latency, causing repeated restarts even when RBAC is correct.

Remediation approach (GitOps-aligned, no band-aids):
1. **Local controller stability:** for controllers where we own the chart (e.g. `cluster-ameide-operators`), disable leader election in local when running single replicas (avoid fragile lease renewals under local apiserver latency).
2. **Namespace-isolated kube-state-metrics:** configure `kube-state-metrics` collectors to namespaced-only resources and set `rbac.useClusterRole=false` so RBAC matches the intended per-namespace observability model.
   - Local: enable a `startupProbe` and relax liveness/readiness thresholds to avoid self-inflicted CrashLoopBackOff while caches sync.
3. **Gateway controller locality/priority (local-only):** if Envoy Gateway remains sensitive to the ClusterIP apiserver path, prefer policy-shaped placement (pin to the control-plane node + higher priority) over manual restarts.

## Update (2025-12-14): ComparisonError flapping + operator leader-election instability (local k3d)

Observed “unhealthy” Argo apps even when the underlying resources were fine (or briefly fine):

- **CRD apps intermittently show `Unknown`** with `ComparisonError`:
  - Example: `cluster-crds-strimzi`, `cluster-crds-redis`.
  - Condition message: `failed to get resource health ... context deadline exceeded`.
  - This is controller-side health evaluation timing out against the apiserver (not a missing health script).
- **Cluster operators intermittently CrashLoop on leader election renewal**:
  - Example: Spotahome `redis-operator` and `cloudnative-pg` operator.
  - Logs show lease renew/update failing with `context deadline exceeded` / `client rate limiter Wait returned an error`.

Key contributing factors found:
- **Values layering footgun**: `sources/values/cluster/globals.yaml` was setting `namespace: argocd`, leaking into third-party charts that honor `.Values.namespace`, causing controllers to be deployed in the wrong namespace and increasing contention/noise (tracked under 519; fixed by removing the key).
- **Local cluster constraints**: k3d is CPU/memory constrained and the combination of Argo server-side diff + many apps/controllers can push apiserver latency high enough that controllers lose their leases.

Remediation approach (GitOps-aligned):
1. **Reduce Argo controller pressure on local clusters** using supported `argocd-cmd-params-cm` values (via the ArgoCD Helm install/upgrade), rather than manual refreshes.
2. **Tune high-churn operators** (k8s client QPS/burst + concurrency) so leader election is resilient under load.
3. **Remove Helm hook “stable secret generation”** where possible (e.g. GitLab `shared-secrets`), replacing with Vault KV → ESO → Secret so sync is idempotent and doesn’t block on long-running hook batches.

## Update (2025-12-15): Local primitive smoke failure (agent-echo-v0)

- **App:** `local-agent-echo-v0-smoke`
  - **Failing hook:** `agent-echo-v0-smoke-helm-test-jobs-agent-echo-v0` (gRPC `grpcurl` connect refused)
  - **Observed:** the `echo-v0-agent` pod listens on an IPv6 socket and refuses IPv4 pod/service traffic (`dial tcp <podIP>:50051: connect: connection refused`).
  - **Constraint:** Agent image/config is owned outside this repo; no reliable in-repo toggle was found to force an IPv4 listener.

Remediation approach (GitOps-aligned):
1. Treat this as an application portability bug: the agent must listen on `0.0.0.0:50051` (or dual-stack with IPv4 enabled) for IPv4-only clusters like k3d/k3s.
2. Until the image is fixed, exclude `agent-echo-v0` and its smoke app from the local component allowlist (prefer “do not generate the Application” over “install + fail”).

## Update (2025-12-14): Local GitLab OutOfSync noise (Argo diff + orphaned Helm hook)

Observed `local-platform-gitlab` reporting `OutOfSync` while workloads remained `Healthy`.

Contributing factors:
- **Orphaned Helm pre-upgrade hook artifacts**: `platform-gitlab-gitlab-upgrade-check` (ConfigMap) persisted with `argocd.argoproj.io/hook-finalizer` even after `gitlab.upgradeCheck.enabled=false`, preventing normal pruning and leaving an “extraneous-but-tracked” resource.
- **Client-side diff sensitivity to defaulted fields**: when server-side diff is disabled for local stability, Argo comparisons can surface Kubernetes-defaulted fields (e.g., gRPC probe `service: ""`, `imagePullPolicy: IfNotPresent`) as spurious drift on StatefulSets.
- **Non-GitOps “debug pod” leftovers**: manual Pods `tmp-ro-help*` in the `argocd` namespace (one `ImagePullBackOff`) add red-noise unrelated to app health.

Remediation approach (GitOps-aligned):
1. **Prefer disabling Helm hooks** (or re-wrapping vendor charts) for GitOps-managed installs; treat hook finalizers/orphans as a fleet policy violation (track under 519).
2. **Expand app-local `ignoreDifferences`** to ignore known defaulted fields for StatefulSets when server-side diff is disabled.
3. **One-time cleanup**: remove hook finalizer and delete the orphaned hook ConfigMap; delete stray debug Pods and add policy guardrails to prevent recreating them in `argocd`.

## Update (2025-12-14): Synced/Healthy but “Last Sync Failed” (hook ordering + bootstrap race)

Observed Argo applications that are currently `Synced/Healthy` but still show `operationState.phase=Failed` from an earlier reconcile:

- `local-platform-langfuse-bootstrap`: ExternalSecrets briefly reported `could not get secret data from provider` during the first sync, before Vault bootstrap had seeded keys and ESO could materialize K8s Secrets.
- `local-agent-echo-v0-smoke`: Helm-test Job hit backoff (service not ready quickly enough for the configured test/backoff window).

Remediation approach (GitOps-aligned):
1. Make bootstrap/smoke hook Jobs **wait on prerequisites** (Secrets and readiness) rather than failing fast.
2. Ensure chart `enabled: false` toggles are **semantically correct** (no `default true` footguns that ignore explicit `false`).
3. Keep “first sync” deterministic by reducing race windows between **Vault bootstrap → ESO → consuming hook Jobs**.
4. Avoid “hook-only Applications” for long-lived GitOps signals: add at least one small, non-hook tracked resource (e.g. a ConfigMap checksum) so Argo can detect drift and auto-sync can clear stale failed `operationState`.
5. Fix local service bind defaults: several workloads were observed listening on IPv6 wildcard only (`:::PORT`) and refusing IPv4 ClusterIP traffic; smoke tests must not depend on such endpoints until they bind `0.0.0.0` (or the workloads are rebuilt to listen on IPv4).
6. Temporal extended smoke `temporal-schema` hook can fail due to the chosen `psql` runner image; local mitigation is to use an image+script combination that reliably reaches `postgres-ameide-rw`, with a follow-up to provide a pinned, multi-arch `psql` runner image (no runtime package downloads).

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
- `{env}-data-temporal` - TemporalCluster/TemporalNamespace (operator-managed)
- `{env}-platform-keycloak` - Keycloak startup
- `{env}-platform-postgres-clusters` - CNPG cluster initialization

---

## Controller Noise – DiffFromCache Errors

**Symptom**: `kubectl logs -n argocd argocd-application-controller-0 --since=30m | grep 'DiffFromCache error'` emits dozens of messages such as `DiffFromCache error: error getting managed resources for app production-platform-envoy-crds: cache: key is missing`.

**Root Cause**: Per the Argo CD operator manual (`docs/operator-manual/server-commands/argocd-application-controller.md`), the application controller stores managed-resource diffs in a one-hour cache (`--app-state-cache-expiration`, default `1h0m0s`). With ~200 apps reconciling every three minutes, entries naturally expire every hour and the next reconciliation logs an error when the cache miss occurs—even though the controller immediately recomputes the diff and continues.

**Fix Applied**: Extended the cache TTL to six hours by setting `controller.env[ARGOCD_APP_STATE_CACHE_EXPIRATION]=6h` in `sources/values/common/argocd.yaml`. Helm installs or the GitOps bootstrap (`ameide-gitops/bootstrap/bootstrap.sh`) now render the updated environment variable so the controller retains app-state cache entries between hourly polls.

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
