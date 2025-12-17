# 450 – ArgoCD Service Issues Inventory

**Status**: Partially Resolved (CI/CD blocking staging/prod)
**Created**: 2025-12-04
**Updated**: 2025-12-16
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

## Update (2025-12-16): `platform-control-plane-smoke` false negatives (Envoy Gateway namespace moved)

Observed a platform smoke Job failing due to an outdated assumption that Envoy Gateway runs per-environment.

- **Symptom:** smoke Job waits for `deployment/envoy-gateway` in the environment namespace and fails even though the cluster is healthy.
- **Root cause:** Envoy Gateway control-plane is now cluster-shared and runs in `argocd`; per-environment resources are `Gateway`/`HTTPRoute`/`EnvoyProxy` in `ameide-{env}`.
- **Fix direction:** update the smoke script to check `deployment/envoy-gateway` in `argocd`, while still checking `Gateway/HTTPRoute` in the environment namespace.

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
3. **Envoy Gateway stability (local-only):** stop depending on leader election leases for a single-replica local controller.
   - Preferred: disable leader election for Envoy Gateway when `replicas=1` (the process currently exits cleanly on `leader election lost`, causing probe failures and CrashLoopBackOff even though config is otherwise valid).
   - If leader election must remain enabled, bypass the `kubernetes.default.svc` ClusterIP path for apiserver calls by pointing the controller at the apiserver endpoint directly (`KUBERNETES_SERVICE_HOST/PORT`), or increase local apiserver resources/timeouts so lease updates do not exceed the controller’s `timeout=5s` window.
   - Implementation note: `gateway-helm` v1.3.0 renders the controller config from a merged “base + user” map; ensure user overrides actually win for `provider.kubernetes.*` (otherwise `leaderElection.disable` can be silently dropped during templating).

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

## Update (2025-12-15): Local operator lease renewals causing Argo `Progressing` (NiFiKop + Strimzi)

Observed cluster-scoped operator Deployments stuck `Progressing` because the controller process **exits on leader election loss** under local apiserver latency.

- **App:** `cluster-nifikop`
  - **Resource:** `Deployment/argocd/nifikop`
  - **Symptom:** `Waiting for rollout to finish: 0 of 1 updated replicas are available...`
  - **Operator logs:** lease renewals fail with `context deadline exceeded` / `client rate limiter Wait returned an error`, followed by `unable to start manager: leader election lost`.
- **App:** `cluster-strimzi-operator`
  - **Resource:** `Deployment/strimzi-system/strimzi-cluster-operator`
  - **Symptom:** `Waiting for rollout to finish: 0 of 1 updated replicas are available...`
  - **Operator logs:** Kubernetes client requests get interrupted during reconciliation after leadership loss, and the process exits (CrashLoopBackOff).

Remediation approach (GitOps-aligned, local-only):
1. Prefer disabling leader election when `replicas=1` for local clusters (avoid “lease renew = liveness gate”).
   - Strimzi: set the chart’s `leaderElection.enable=false` in local cluster values.
   - NiFiKop: add a chart value to conditionally omit `-leader-elect`, then set it off in local cluster values.
2. Keep the default topology for real clusters (where multiple replicas or higher apiserver capacity makes leader election meaningful).

## Update (2025-12-15): `local-www-ameide-platform` not Ready (dev-server startup + Redis auth mismatch)

- **App:** `local-www-ameide-platform`
  - **Resource:** `Deployment/ameide-local/www-ameide-platform`
  - **Symptom:** rollout stuck with probes timing out; container frequently exits `1`.
  - **Observed root causes:**
    1. **Non-deterministic startup**: `pnpm run dev` runs via Corepack and may download `pnpm` at runtime; Next dev also attempts to mutate `tsconfig.json` and can fail inside an immutable image.
    2. **Redis auth mismatch**: local `redis-failover` runs in `standalone` mode and always enforces `--requirepass` from `redis-auth` even when `auth.enabled=false`. The app connects via `REDIS_URL` and does not successfully authenticate, causing `NOAUTH Authentication required` loops.
    3. **Image/tag mismatch**: the `dev` image tag does not include a production `.next` build, so `next start` fails with `production-start-no-build-id`. The Argo baseline must use a production-built tag (e.g. `main`) or a dedicated runtime tag that includes the build output.
    4. **Multi-arch gap**: the `main` image tag is `amd64`-only (local arm64 pulls fail with `no match for platform in manifest`). If we expect local to run a production-like entrypoint, the runtime tag must be multi-arch (or local must use an arm64-capable tag/registry).

Remediation approach (GitOps-aligned, no band-aids):
1. Treat Argo-managed local baseline as production-like: run a deterministic server entrypoint (no runtime package-manager downloads, no dev server).
2. Fix the local Redis standalone chart behavior so `auth.enabled=false` truly runs Redis without `requirepass` (and keep `auth.enabled=true` for environments that need it).
3. Until a **multi-arch production image** exists (tag that contains a prebuilt `.next` and runs `next start`), disable the Argo baseline for local and use the dedicated Tilt release per [424-tilt-www-ameide-separate-release.md](424-tilt-www-ameide-separate-release.md).

## Update (2025-12-16): `local-www-ameide-platform` OutOfSync after local disable (Argo auto-sync safety on empty desired state)

- **App:** `local-www-ameide-platform`
  - **Symptom:** Application stays `OutOfSync` while resources are otherwise `Healthy`.
  - **Argo condition:** `Skipping sync attempt... auto-sync will wipe out all resources`
  - **Root cause:** Local overrides intentionally disable the chart (renders **zero** desired manifests). With automated prune enabled, ArgoCD refuses to auto-sync unless `syncPolicy.automated.allowEmpty=true` is set.

Remediation approach (vendor-aligned, GitOps-idempotent):
1. Keep the chart disabled for local until a deterministic multi-arch runtime image exists (per the previous update).
2. Avoid “empty desired state + prune” by having the chart render a deterministic “disabled marker” resource (e.g., a ConfigMap) when `enabled=false`, so Argo can prune previously-created resources without requiring `allowEmpty`.
3. Keep the disable scoped to local (either omit the component from the local component set, or keep the App visible-but-off via the marker resource).

## Update (2025-12-16): Transient `Progressing` from strict probe timeouts under local load (Strimzi + Loki + Alloy)

- **Env:** `local` (k3d)
- **Observed symptoms:**
  - `cluster-strimzi-operator` appeared `Progressing` intermittently and the `strimzi-cluster-operator` pod restarted due to probe failures.
  - `local-platform-loki` and `local-platform-alloy-logs` emitted periodic readiness probe failures (`context deadline exceeded`) while pods remained Running.
  - Other local system workloads showed the same “probe budget too strict under transient stalls” pattern:
    - `ameide-local/keycloak-0` readiness probe failures (`/health/ready` timeouts) with frequent restarts.
    - `ameide-local/platform-prometheus-kube-state-metrics-*` liveness failures returning HTTP 503, leading to repeated restarts.
    - `argocd/argocd-application-controller-*` and `argocd/argocd-repo-server-*` occasionally timing out on `healthz` probes under load.
- **Root cause:** Several control-plane components used HTTP probe defaults with `timeoutSeconds: 1`, which is too aggressive for a constrained local cluster during apiserver/client stalls. Kubelet interprets transient slow responses as Unhealthy and restarts pods (liveness) or marks them NotReady (readiness), which can surface as Argo “Progressing”/flapping even when the steady-state is functional.

Remediation approach (vendor-aligned, GitOps-idempotent):
1. Prefer vendor-supported configuration to **increase probe budgets** (e.g., `timeoutSeconds` and `failureThreshold`) for local, and ensure charts expose those knobs.
2. For charts that do **not** expose probe timeout/failure threshold, patch the vendored chart minimally to add those fields with defaults matching upstream behavior, then override locally.
3. Add explicit resource requests for observability components so they are not starved under contention.
4. Reduce local apiserver stalls at the source by **reserving the k3d control-plane node** (taint `k3d-ameide-server-0` `NoSchedule` when agents exist) so heavy data-plane workloads don’t compete with the control-plane on the same node.
5. Keycloak operator nuance: `Keycloak.spec.{livenessProbe,readinessProbe}` only supports a small subset of fields (no `timeoutSeconds`/`httpGet`), so full probe tuning must be expressed via `Keycloak.spec.unsupported.podTemplate` (container `livenessProbe/readinessProbe/startupProbe`) instead.

### Follow-up (local): kube-state-metrics CrashLoopBackOff from hardcoded `/livez`

- **Pod:** `ameide-local/platform-prometheus-kube-state-metrics-*`
  - **Symptom:** `CrashLoopBackOff` with restarts climbing (liveness probe repeatedly returns HTTP 503 for `/livez`).
  - **Contributing factor:** runs as `BestEffort` (no CPU/memory requests), making it a prime victim during local contention.
- **Root cause:** upstream `kube-state-metrics` chart hardcodes probe paths (`/healthz`, `/livez`, `/readyz`) in templates, so local cannot switch liveness away from `/livez` without a minimal vendored-chart patch.

Remediation approach (vendor-aligned, reproducible):
1. Patch the vendored `kube-state-metrics` chart template to allow `livenessProbe.httpGet.path` override (default remains `/livez`).
2. In local values, set liveness path to `/healthz` (process-alive) and keep readiness on `/readyz` (functional), so transient apiserver stalls don’t cause a self-inflicted CrashLoop.
3. Set resource requests for kube-state-metrics in local to avoid BestEffort starvation.

### Follow-up (local): `postgres-password-reconcile` CronJob failures (runtime `apk add`)

- **CronJob:** `ameide-local/postgres-password-reconcile` (from `platform-postgres-clusters`)
  - **Symptom:** intermittent failed Jobs (exit code 2) and noisy failed pods.
  - **Root cause:** the job installs dependencies at runtime (`apk add postgresql-client curl jq`), which is not deterministic and can fail under transient network/DNS/apiserver stalls.

Remediation approach (vendor-aligned, reproducible):

## Update (2025-12-16): ArgoCD UI shows persistent `Progressing` due to local apiserver stalls (k3d/k3s + Kine SQLite)

- **Env:** `local` (k3d/k3s)
- **Symptom:** ArgoCD UI continues to show several Applications stuck in `Progressing` even after underlying workloads converge; cluster queries like `kubectl -n argocd get applications` intermittently time out or return stream errors.
- **Hypothesis (vendor-aligned):** local k3s default datastore (Kine→SQLite) can become slow under high churn, causing kube-apiserver write latency (status PATCHes, Lease renewals, Job updates). When ArgoCD cannot consistently read/write status via the API server, UI health can appear stale.
- **Compounding failure (observed):** `argocd-repo-server` can enter `CrashLoopBackOff` because its liveness probe uses an aggressive `timeoutSeconds: 1`; under local stalls, `GET /healthz?full=true` exceeds the budget and kubelet restarts the container, degrading chart rendering and making Application status even noisier/staler.

Remediation approach (reproducible, idempotent, Terraform-first):
1. Treat the local cluster as disposable infrastructure and **recreate it via Terraform** (do not manually delete namespaces/resources as “cleanup”).
2. After recreate, re-verify ArgoCD health with bounded requests (use `--request-timeout=30s` for `kubectl` calls) to confirm the API server is responsive and Application status updates are flowing.

Commands (local):
- `infra/scripts/tf-local.sh destroy -auto-approve`
- `infra/scripts/tf-local.sh apply -auto-approve`

Verification:
- `kubectl get nodes --request-timeout=30s`
- `kubectl -n argocd get pods --request-timeout=30s`
- `kubectl -n argocd get applications --request-timeout=30s` (should return quickly; any remaining `Progressing` should have an actionable resource health message)
1. Prefer CloudNativePG `managed.roles` (already configured) for role + password reconciliation.
2. Keep `passwordReconcile` as an escape-hatch feature, but default it **off** (including local) unless we are actively migrating password sources; avoid long-running “self-heal” CronJobs that rely on runtime package installs.

### Follow-up (local): `cluster-gateway` stuck `Degraded` (Envoy Gateway controller unschedulable on tainted control-plane node)

- **App:** `cluster-gateway`
  - **Resource:** `Deployment/argocd/envoy-gateway`
  - **Symptom:** Deployment exceeds progress deadline; pod remains `Pending` indefinitely; downstream `HTTPRoute` resources stay `Progressing` ("Waiting for parent Gateway status").
  - **Observed cause:** local values pin the controller to the control-plane node via `nodeSelector`, but the control-plane node is tainted (`node-role.kubernetes.io/control-plane:NoSchedule` and `node-role.kubernetes.io/master:NoSchedule`) and the Deployment does not include tolerations.

Remediation approach (vendor-aligned, reproducible):
1. Keep the local-only control-plane pin (it reduces lease-renewal flakiness under k3d load), but add explicit tolerations for the control-plane/master taints.
2. Scope the change to local-only values (`sources/values/cluster/local/gateway.yaml`) so hosted clusters keep their vendor defaults.

### Follow-up (local): `local-platform-loki` stuck `Progressing` (sidecar ImagePullBackOff after local mirror fallback)

- **App:** `local-platform-loki`
  - **Resource:** `StatefulSet/ameide-local/platform-loki`
  - **Symptom:** 1/2 containers Ready; pod shows `ImagePullBackOff` for the sidecar container; Application remains `Progressing`.
  - **Observed cause:** local overrides switch Loki’s main image from the GHCR mirror to the upstream multi-arch image (arm64-safe), but the chart still uses the GHCR-mirrored `k8s-sidecar` image. Local also drops `imagePullSecrets`, so the private mirror sidecar cannot be pulled.

Remediation approach (vendor-aligned, reproducible):
1. In local values (`sources/values/env/local/observability/platform-loki.yaml`), override the sidecar image to an upstream multi-arch `k8s-sidecar` image (no mirror dependency), consistent with the local Loki fallback.
2. Keep the shared/default sidecar image on the mirror for hosted clusters, gated by the mirror policy tracked in backlog/456.

### Follow-up (local): NiFiKop-managed NiFi pods stuck `Error` (placeholder secret value)

- **App:** `local-data-nifi`
  - **Resource:** `NifiCluster/ameide-local/nifi`
  - **Symptom:** NiFi node pods like `nifi-0-node*` terminate with exit code 1; cluster remains in `ClusterRollingUpgrading` and `podIsReady=false`.
  - **Observed config smell:** the NiFi chart currently uses a committed placeholder `config.sensitivePropsKey: CHANGEME`, which is both insecure and (depending on NiFi version) can be invalid/too short and prevent startup.

Remediation approach (GitOps-idempotent, secrets-aligned):
1. Disable NiFi for local by default (extended data-plane component) until the sensitive properties key is sourced from Vault/ESO (no committed placeholders).
2. Add a chart guardrail: if NiFi is enabled and the sensitive key is missing/placeholder, fail fast at render time.

### Follow-up (local): vault-bootstrap CronJob leaves stale failed Jobs/Pods

- **CronJob:** `ameide-local/foundation-vault-bootstrap-vault-bootstrap`
  - **Symptom:** old failed job pods remain visible for hours (noise) even after subsequent successful runs.
  - **Root cause:** `failedJobsHistoryLimit=1` keeps the latest failed Job indefinitely without a TTL; local users interpret this as “still broken” even when the controller is otherwise progressing.

Remediation approach (GitOps-idempotent):
1. Add `ttlSecondsAfterFinished` to the CronJob’s `jobTemplate.spec` so completed/failed Jobs are garbage-collected automatically.
2. In local, set `failedJobsHistoryLimit=0` so the steady-state cluster does not retain failed Jobs.

### Follow-up (local): Vault bootstrap stuck `Init:0/1` when local secrets seed is missing

- **Env:** `local` (k3d)
- **Pod:** `ameide-local/foundation-vault-bootstrap-vault-bootstrap-*`
  - **Symptom:** pod stays `Init:0/1` / `PodInitializing` and all `ExternalSecret` resources remain `SecretSyncedError`.
  - **Observed error:** `MountVolume.SetUp failed ... secret "vault-bootstrap-local-secrets" not found`
  - **Downstream impact:** Vault remains sealed → `SecretStore/ameide-vault` is `InvalidProviderConfig` (`Vault is sealed`) → `ghcr-pull` and all app credentials never materialize.

Remediation approach (Terraform-first, reproducible):
1. Ensure the local provisioning entrypoint (`infra/scripts/tf-local.sh apply`, and/or the Terraform bootstrap module) **seeds** `vault-bootstrap-local-secrets` from `.env` via `infra/scripts/seed-local-secrets.sh` **before** applying the GitOps overlay.
2. Keep the seed idempotent (apply/patch the Secret, do not require manual deletes) so repeated `apply` runs always converge the same way.

### Follow-up (local): Vault bootstrap fixtures missing required DB password keys (ExternalSecrets stay `SecretSyncedError`)

- **Symptom:** app Deployments remain `CreateContainerConfigError` because required Secrets (e.g., `transformation-db-credentials`) never materialize.
- **Observed root cause:** `ExternalSecret` resources reference Vault keys like `transformation-db-password`, `threads-db-password`, `workflows-db-password`, `temporal-db-password`, but the local Vault bootstrap fixtures do not seed those keys (and the backfill path only copies from Kubernetes Secrets that don’t exist on a fresh cluster).

Remediation approach (GitOps-idempotent, policy-shaped):
1. Ensure `foundation-vault-bootstrap` fixtures seed all required `*-db-password` keys (use `"__generate__"` for stable per-cluster values) so fresh local clusters converge without manual steps.
2. Keep the backfill behavior as a migration helper (copy existing K8s Secret → Vault only when the Vault key is missing/placeholder), but do not rely on it for greenfield clusters.

## Update (2025-12-16): ArgoCD repo credentials can block Git sync (invalid GitHub token forces auth)

- **Env:** `local` (k3d) and any bootstrap path that creates `repo-creds-ameide-gitops`.
- **Symptom:** ArgoCD stays pinned to an older revision (apps remain `Synced` but do not advance to new `main` commits), or repo-server shows git auth errors.
- **Observed evidence:** running `git ls-remote https://github.com/ameideio/ameide-gitops.git refs/heads/main` from the repo-server pod fails when a `repository` Secret provides an invalid/expired GitHub token (`Password authentication is not supported` / `Invalid username or token`).
- **Root cause:** For public Git repos, creating a repository Secret with an invalid token forces BasicAuth and breaks fetches (GitHub does not “fallback” to anonymous).

Remediation approach (vendor-aligned, reproducible):
1. For **public** repos, do not set `username`/`password` repo credentials at all (omit the Secret or create it with only `type` + `url`).
2. For **private** repos, use a valid credential and ensure the **username matches the credential type**:
   - **PAT**: `username=<your GitHub username>`, `password=<PAT>`
   - **GitHub App / installation token**: `username=x-access-token`, `password=<token>`
3. In bootstrap tooling (Terraform/Bicep/scripts), prefer “try anonymous first, then use token only if required”, and if `GITHUB_USERNAME` is not provided, derive it from the token via the GitHub API (best-effort) instead of hardcoding `x-access-token`.

## Update (2025-12-16): `local-platform-alloy-logs` rollout stuck `Progressing` (arm64 + mirror image CrashLoop)

- **App:** `local-platform-alloy-logs`
  - **Resource:** `Deployment/ameide-local/alloy-logs`
  - **Symptom:** app stuck `Progressing` with `Waiting for rollout to finish: 1 old replicas are pending termination...`
  - **Pod logs:** new ReplicaSet’s `alloy` container CrashLoops with `SIGSEGV` and stack traces referencing `runtime/asm_amd64.s`, consistent with an `amd64` artifact running on local `arm64` (or a broken emulation path).
  - **Root cause:** local uses `ghcr.io/ameideio/mirror/alloy:v1.11.3`; the mirror tag is not reliably multi-arch for local `arm64`.

Remediation approach (GitOps-aligned):
1. Add a **local-only** override to use the upstream multi-arch `grafana/alloy:v1.11.3` image until the GHCR mirror is republished as a proper multi-arch manifest list (track under 456).
2. Keep the shared defaults on the mirror for real clusters once multi-arch is verified.

## Update (2025-12-17): `local-platform-langfuse` CrashLoopBackOff (arm64 + mirror image not usable)

- **App:** `local-platform-langfuse`
  - **Resource:** `Deployment/ameide-local/platform-langfuse-web`
  - **Symptom:** Application stays `Progressing` with `Waiting for rollout to finish: 1 old replicas are pending termination...`; pods CrashLoop.
  - **Observed evidence (pod logs):** Go runtime fatal `lfstack.push invalid packing` with stack frames referencing `runtime/asm_amd64.s` while running on local `arm64`.
  - **Root cause:** the configured image `ghcr.io/ameideio/mirror/langfuse:3.120.0` is not usable on local `arm64` (single-arch mirror tag or broken artifact selection under emulation).

Remediation approach (vendor-aligned, reproducible):
1. Add a **local-only** override to use the upstream multi-arch Langfuse image (`ghcr.io/langfuse/langfuse:3.120.0`, or vendor-recommended equivalent) until the mirror tag is republished as a proper multi-arch manifest list (track under 456).
2. Keep the mirror defaults for clusters that are `amd64`-only once the mirror is validated as multi-arch.

## Update (2025-12-17): `local-workflows-runtime` CrashLoopBackOff (Temporal namespace missing)

- **App:** `local-workflows-runtime`
  - **Resource:** `Deployment/ameide-local/workflows-runtime`
  - **Symptom:** Application stays `Progressing` with `Waiting for rollout to finish: 0 of 1 updated replicas are available...`; container restarts and exits quickly.
  - **Observed evidence (pod logs):** connects to Temporal frontend successfully, then fails with `Temporal namespace is missing; create it or update configuration`.
  - **Root cause:** the runtime is configured for `TEMPORAL_NAMESPACE=ameide` but the GitOps-managed `TemporalNamespace` resources only create `default` in `ameide-local`, so the required namespace never exists.

Remediation approach (vendor-aligned, GitOps-idempotent):
1. Ensure `local-data-temporal` renders a `TemporalNamespace` CR for `ameide` (same `clusterRef` as the env’s `TemporalCluster`) so dependent apps never need imperative “create namespace” logic.
2. Keep ordering wave-safe: `TemporalCluster` + schema/preflight must converge before apps that require the namespace (use sync-waves and/or Argo health checks, not manual steps).

## Update (2025-12-17): `local-data-temporal` Degraded (TemporalCluster status flaps to `ServicesNotReady`)

- **App:** `local-data-temporal`
  - **Resource:** `TemporalCluster/ameide-local/temporal`
  - **Symptom:** Application stays `Degraded` even though Temporal Deployments/Pods are Running and clients can connect.
  - **Observed evidence:** `TemporalCluster.status.conditions[type=Ready]` reports `status=False` with reason `ServicesNotReady`, and `status.services[].ready=false` for all services.
  - **Likely root cause:** the operator renders `temporal-*-headless` Services with an `http-metrics` port that targets `targetPort: metrics`, but the Temporal Deployments do not expose a container port named `metrics` when `TemporalCluster.spec.metrics` is unset (config shows `global.metrics: null`). This produces Service endpoints that are missing the metrics port, and the operator reports `ServicesNotReady`.

Remediation approach (vendor-aligned, GitOps-idempotent):
1. Enable metrics exposition in the `TemporalCluster` CR (`spec.metrics.enabled=true` with Prometheus listen port), so Deployments expose a `metrics` port that matches the Service `targetPort`.
2. Keep Argo health customizations strict (do not mask operator readiness); fix the CR spec so the operator reports Ready deterministically.

## Update (2025-12-17): Local bootstrap exports CA cert (Terraform automation)

- **Goal:** eliminate manual steps during local cluster recreation.
- **Change:** local Terraform bootstrap waits for cert-manager `Certificate/ameide-local-ca` and `Certificate/ameide-wildcard-local` in `ameide-local`, then exports the CA cert to `artifacts/local/ameide-local-ca.crt` (used by `scripts/local/setup.sh` host trust flow).

## Update (2025-12-15): Local observability + Gateway Progressing (OTEL collector arch + local LoadBalancer)

Observed Argo apps stuck `Progressing`:
- `local-platform-otel-collector`
- `cluster-gateway`
- `local-platform-gateway`

### `local-platform-otel-collector`: CrashLoopBackOff (arm64)

- **Pod:** `ameide-local/otel-collector-*`
- **Symptom:** container exits `2` and CrashLoops; logs show a Go runtime dump with `runtime/asm_amd64.s` frames.
- **Root cause:** the configured mirror tag `ghcr.io/ameideio/mirror/otel-collector-contrib:0.91.0` is not usable on local `arm64` (either `amd64`-only or effectively broken under emulation).
- **Impact:** OTLP exporters fail across apps and add red-noise.

Remediation approach (GitOps-aligned):
1. Add a local override to use a known multi-arch collector image (or pin a digest) until the mirror is republished as a proper manifest list (track under 456).
2. Keep mirror validation so tags must include `linux/amd64` + `linux/arm64` before being accepted as “fleet standard”.

### `local-platform-gateway`: “No addresses assigned” (no local LB)

- **Resource:** `Gateway/ameide-local/ameide`
- **Symptom:** `ADDRESS` is empty and `Programmed=False`.
- **Root cause chain:**
  1. The Envoy Service for the local Gateway is `type: LoadBalancer` but has `EXTERNAL-IP: <pending>` in k3d/k3s, so the Gateway cannot be assigned an address.
  2. The Envoy Service also had **no Endpoints** because the Envoy proxy pods were not Ready (xDS instability while the controller was flapping), preventing steady-state convergence.

Remediation approach (reproducible local):
1. Make “LoadBalancer provider” an explicit local capability: deploy MetalLB (or re-enable k3s `servicelb`) via GitOps so `LoadBalancer` services are assigned addresses deterministically.
2. Keep local Gateway health tied to real address assignment; do not mask with Argo health overrides.
3. Stabilize Envoy Gateway (controller) so data-plane proxies become Ready and populate Service Endpoints; without endpoints, LoadBalancer provisioning and Gateway status can stall.

## Update (2025-12-16): Gateway policy schema drift (ComparisonError) + cluster-gateway OutOfSync

Observed Argo apps failing comparison or staying `OutOfSync` even while resources are otherwise functional:
- `local-platform-gateway`: `ComparisonError` during diff calculation
- `cluster-gateway`: `OutOfSync` on `Gateway/cluster`

Root cause:
- We rendered fields that **do not exist in the installed CRD/Gateway API schema**, which causes Argo to fail structured diff:
  - `EnvoyPatchPolicy.spec.targetRef.namespace` is not declared in the Envoy Gateway policy schema.
  - `Gateway.spec.infrastructure.parametersRef.namespace` is not declared in the Gateway API schema (namespace is implied; reference is within the Gateway’s namespace).

Remediation approach (vendor-aligned, no band-aids):
1. Remove the unsupported `namespace` fields from the manifests so Argo structured diff works and self-heal can converge.
2. Keep per-environment isolation by ensuring the referenced `EnvoyProxy` is deployed in the same namespace as the `Gateway` (namespaced ownership boundary stays intact without cross-namespace refs).

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

## Update (2025-12-16): `local-data-temporal` stuck `Missing` (PreSync DB preflight deadlocks schema ownership)

- **App:** `local-data-temporal`
  - **Symptom:** Application stays `OutOfSync/Missing`; `TemporalCluster`/`TemporalNamespace` never created.
  - **Observed failure:** PreSync hook Job `temporal-db-preflight` fails with `relation "namespace_metadata" does not exist`, because the Temporal schema has not been installed yet.
  - **Root cause:** the preflight hook runs **before** the Temporal operator ever sees a `TemporalCluster` to reconcile, but it assumes Temporal schema tables exist (schema ownership is Temporal, not Flyway).

Remediation approach (vendor-aligned, GitOps-idempotent):
1. Add a deterministic Temporal schema Job (use `temporal-sql-tool setup-schema` + `update-schema`) that runs **before** the metadata preflight, following the Temporal Helm chart pattern and backlog/425.
2. Keep the metadata preflight, but run it only after schema exists (either by ordering it after schema setup, or by guarding on table existence so it never blocks `TemporalCluster` creation).
3. Pin schema tooling images/tags (avoid `:latest`) so local arm64 and CI behave consistently.

## Update (2025-12-16): `local-platform-keycloak-realm` stuck `Missing` (realm import modeled as Argo PreSync hook)

- **App:** `local-platform-keycloak-realm`
  - **Symptom:** Application stays `OutOfSync/Missing` with `SyncError` even after `local-platform-keycloak` becomes Healthy.
  - **Observed failure:** `KeycloakRealmImport/*` resources are rendered with `argocd.argoproj.io/hook: PreSync` and repeatedly fail with `Deployment not yet ready`, exhausting Argo retries and preventing the CRs (and supporting ConfigMaps/ExternalSecrets) from ever being applied.
  - **Root cause:** realm import is a **declarative operator CR**; modeling it as an Argo hook turns “operator retries” into “Argo hard failure”, and breaks self-heal after transient Keycloak startup delays.

Remediation approach (vendor-aligned, Argo-standard):
1. Render `KeycloakRealmImport` CRs as normal tracked resources (use `sync-wave` ordering only), letting the Keycloak operator reconcile when the Keycloak instance is Ready.
2. Keep any imperative “client secret extraction” as a bounded Job (ideally PostSync or periodic), but do not gate CR creation on hook success.

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

Note:
- Running more than one cert-manager controller set in a shared cluster is a risk area (contention + cluster-scoped side effects). Fleet policy target is **one cert-manager install per cluster**, with multi-tenancy handled via RBAC/policy + namespaced Issuers (track under 519).
- Option A execution: converge on `cluster-cert-manager` only (namespace `cert-manager`) and delete `argocd-cert-manager` plus any per-environment `foundation-cert-manager*` installs.

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

### Azure Terraform deploy failed (2025-12-17)

**Status**: 🚧 OPEN (blocks `infra/scripts/deploy.sh azure` end-to-end)

**Symptom**: Terraform apply exits non-zero, so bootstrap never runs (no ArgoCD install, no root apps).

**Errors observed** (from `artifacts/terraform-outputs/azure-*.deploy.log`):
- `azuread_group.telepresence_developers`: `Authorization_RequestDenied` (403) while listing groups by `displayName`.
- `module.keyvault.azurerm_key_vault.main`: Key Vault already exists and must be imported to be managed by Terraform.
- `azurerm_key_vault_secret.env_secrets[*]`: `ForbiddenByRbac` (403) when Terraform checks/creates secrets (e.g. `github-token`, `ghcr-token`), because the Terraform runner identity has no Key Vault data-plane RBAC on the vault.
- ArgoCD bootstrap installed Redis from `ghcr.io/ameideio/mirror/redis:7.2.5` and failed with `ImagePullBackOff` because `ghcr-pull` is not present during bootstrap (chicken-and-egg).
- `foundation-vault-bootstrap` Jobs were stuck `Init:ImagePullBackOff` because the bootstrap chart used GHCR mirror images (`ghcr.io/ameideio/mirror/vault`, `ghcr.io/ameideio/mirror/bitnami-kubectl`) before `ghcr-pull` existed, preventing Vault initialization and leaving all `SecretStore/ameide-vault` resources NotReady.
- After switching Vault bootstrap to public images, `foundation-vault-bootstrap` Jobs failed `Init:StartError` because the initContainer used `rancher/kubectl` (no `/bin/sh`), but the chart’s initContainer command is a shell script that copies `kubectl` into a shared volume.
- `foundation-vault-bootstrap` Jobs also waited forever for Vault because the chart defaulted `vault.serviceName=foundation-vault-core`, but Vault core is deployed per-environment with `fullnameOverride: vault-core-<env>` (e.g., `vault-core-dev`), so `VAULT_ADDR` resolved to a non-existent Service.
- SecretStores could not authenticate to Vault even after unseal because the Vault Kubernetes auth roles were written with `audience=https://kubernetes.default.svc`, but AKS service account tokens use different `aud` values, resulting in `403 invalid audience (aud) claim`.
- Even after `ghcr-pull` was materialized, many pods remained `ImagePullBackOff` with `403 Forbidden` from `https://ghcr.io/token` because Vault bootstrap had seeded placeholder `ghcr-token`/`ghcr-username` instead of sourcing the real credentials from Azure Key Vault (AKV overrides were not applied as expected; see 451 for the `secretPrefix` contract).

**Root cause**
- **Entra ID directory operations are not least-privilege** for the cluster deployer identity: creating or even checking groups requires tenant-level Microsoft Graph permissions that our subscription-scoped deployer SP does not have.
- **Out-of-band Key Vault recovery breaks Terraform idempotency**: `deploy.sh` uses `az keyvault recover` before Terraform runs; that turns a soft-deleted vault into an active resource that is not in Terraform state, causing a create conflict.
- **Bootstrap did not apply the intended env overlay values**: bootstrap looked for `sources/values/<env>/foundation/foundation-argocd.yaml` (missing the `env/` segment), so the “public images during bootstrap” override at `sources/values/env/dev/foundation/foundation-argocd.yaml` was never applied.

**Remediation approach (vendor-aligned, reproducible)**
1. Keep Key Vault lifecycle logic inside Terraform by enabling Key Vault recovery in the `azurerm` provider (`features.key_vault.recover_soft_deleted_key_vaults=true`) and avoiding out-of-band `az keyvault recover` in the Terraform path.
2. Decouple Entra ID group provisioning from AKS infrastructure creation:
   - Default to using **pre-provisioned** Entra group object IDs (pass via `developer_role_assignments`).
   - Only create Entra groups in a dedicated “directory bootstrap” stack executed with a properly privileged identity.
3. Ensure the Terraform runner can seed Key Vault secrets deterministically:
   - Grant the Terraform runner identity `Key Vault Secrets Officer` on the vault (RBAC mode), and order secret creation after role assignment.
   - Account for Azure RBAC propagation delays (retry apply or gate secret writes) so the end-to-end `deploy.sh azure` path is stable.
4. Validate the full external chain after fix: `deploy.sh azure` → Terraform converges → `azure.json` outputs → bootstrap runs → Argo apps converge.
5. Fix bootstrap env overlay resolution and rerun bootstrap so ArgoCD uses public bootstrap images (especially Redis) until `ghcr-pull` is materialized by External Secrets.
6. For bootstrap jobs that need `kubectl`, prefer an initContainer image that includes `/bin/sh` (or change the initContainer to avoid shell) and pin the tooling image by digest to keep the bootstrap path deterministic.
7. Remove hardcoded cross-chart service name assumptions in bootstrap charts: default to deriving env-scoped service names from the release namespace (e.g., `ameide-dev` → `vault-core-dev`) unless explicitly overridden.
8. Do not hardcode Kubernetes service account token audiences in Vault roles: make `audience` optional (unset by default) or set it from a cluster-specific value when strict audience binding is desired.

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
