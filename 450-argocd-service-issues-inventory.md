# 450 – ArgoCD Service Issues Inventory

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

**Status**: Partially Resolved (CI/CD blocking staging/prod)
**Created**: 2025-12-04
**Updated**: 2026-01-03
**Commits**:
- `485a3961 fix(sso): make keycloak client-patcher deterministic`
- `e6dd0f32 fix(keycloak): ensure client specs exist before patcher hook`
- `a73eabe6 fix(sso): make keycloak secret extraction deterministic`
- `d7abdbea fix(registry): ensure default SA uses ghcr-pull`
- `1164cac7 fix(foundation-namespaces): restore manifests list`
- `3654788b fix(local): bootstrap argocd without dex blocking`
- `04915b5a fix(bootstrap): don't block on dex readiness`
- `3a311a7d fix(argocd): refresh dex secret from vault quickly`
- `65581180 ci(azure-destroy): block on nodepool delete (avoid --no-wait false positives)`
- `bed933c2 ci(azure-apply): manual workflow + explicit confirmation`
- `0af30169 ci(azure-apply): sync globals from terraform outputs`
- `1bb02048 refactor(azure): treat identity/ip values as generated`
- `e9400838 chore(azure): sync globals from terraform outputs`
- `364ca7e8 ci(terraform): queue apply/destroy + add force-unlock workflow`
**Related**: [442-environment-isolation.md](442-environment-isolation.md), [445-argocd-namespace-isolation.md](445-argocd-namespace-isolation.md), [446-namespace-isolation.md](446-namespace-isolation.md), [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), [451-secrets-management.md](451-secrets-management.md), [456-ghcr-mirror.md](456-ghcr-mirror.md), [589-rpc-transport-determinism.md](589-rpc-transport-determinism.md)

---

## Summary

Total applications: 200
- **Healthy/Synced**: ~83
- **Degraded**: 97 (across dev/staging/production)
- **Missing**: 6
- **OutOfSync**: 20+
- **Progressing**: 12

**Clean-state note (image policy):** This inventory contains historical incidents from the “floating tags + pull-policy” era. The current GitOps target state is `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`: deploy digest-pinned refs and drive rollouts via Git PR write-back/promotion PRs. When reading older “missing `:main` tag” or “restart pods to pick up `:dev`” entries, treat them as superseded.

## Update (2026-01-11): Local Gateway API GRPCRoute drift due to SSA field ownership (manual patch)

- **Date/Env**: 2026-01-11 / local
- **Argo app**: `local-platform-gateway`
- **Symptom**: App stayed `OutOfSync` while resources were Healthy; OutOfSync resources were `GRPCRoute/{platform,threads,workflows,inference}`.
- **Root cause**: A prior manual `kubectl patch` became a separate SSA field manager (`kubectl-patch`) owning `.spec.rules`, so Argo’s `ServerSideApply=true` could not reclaim ownership cleanly.
- **Recovery (one-time, operational)**:
  - Re-apply the rendered GRPCRoute manifests with SSA using `--field-manager=argocd-controller --force-conflicts` so GitOps re-owns the fields.
  - Once ownership is restored, Argo returns to `Synced/Healthy` without any ignore rules or “forever force” settings.
- **Prevent**: avoid `kubectl patch` against Argo-managed objects (especially Gateway API resources). Prefer Git changes; if an emergency patch is unavoidable, apply with the Argo field manager and capture the change as a follow-up Git PR immediately.

## Update (2026-01-03): Local bootstrap convergence incidents (Dex/Envoy, CNPG placement, hook jobs)

- **Date/Env**: 2026-01-03 / local
- **Argo app**: `local-platform-gateway`, `argocd-config`
- **Unhealthy resource**: `Deployment/argocd/argocd-dex-server`
- **Symptom**: Dex CrashLoopBackOff with `Get "https://auth.local.ameide.io/realms/ameide/.well-known/openid-configuration": ... dial tcp <envoy-svc-ip>:443: connect: connection refused`
- **Root cause**: `NetworkPolicy/envoy-ameide-local-ingress-lockdown` allowed Service ports `80/443` instead of Envoy data-plane Pod `targetPort`s `10080/10443`, blocking in-cluster callers (including Dex) from reaching Keycloak via the local Gateway.
- **Fix**: Set `networkPolicies.envoyProxyIngress.publicPorts` to `[10080, 10443, 8000]` so the policy matches pod ports.
- **Verification**: `envoy-ameide-local-ingress-lockdown` lists `10080 10443 8000`; Dex pods stay `Running`; local Applications converge `Synced/Healthy`.

- **Date/Env**: 2026-01-03 / local
- **Argo app**: `local-platform-postgres-clusters`
- **Unhealthy resource**: `Cluster/ameide-local/postgres-ameide` (CNPG) → `Pod/ameide-local/postgres-ameide-*` `Pending`
- **Symptom**: Postgres instance stuck Pending after local control-plane taints; cascades into Keycloak/Dex failures (`issuer unreachable`).
- **Root cause**: local-path `WaitForFirstConsumer` can pin a PV to the control-plane node if the first Postgres pod schedules there before taints/placement constraints exist.
- **Fix**: Add local CNPG `nodeAffinity` to avoid `node-role.kubernetes.io/control-plane` and `node-role.kubernetes.io/master` so first scheduling (and PV binding) lands on an agent node.
- **Verification**: With control-plane taints enabled, CNPG instances schedule on agent nodes and remain stable across restarts; downstream apps (Keycloak/Dex) converge.

- **Date/Env**: 2026-01-03 / local
- **Argo app**: `local-platform-langfuse-bootstrap`
- **Unhealthy resource**: `Job/ameide-local/platform-langfuse-bootstrap` (PostSync hook)
- **Symptom**: Hook Job fails early with `psql ... connection refused` when Postgres service has no endpoints yet; job remains as a failed resource and does not self-heal.
- **Root cause**: The bootstrap script attempted `psql` before Postgres was reachable; Argo hook name is stable, so a failed Job can block future bootstrap unless deleted.
- **Fix**: Add an explicit “wait for Postgres” loop before first `psql`, and set hook delete policy to include `BeforeHookCreation` so retries are deterministic on resync.
- **Verification**: On fresh bootstrap, Langfuse bootstrap waits for DB readiness and completes; no persistent failed hook Jobs remain.

- **Date/Env**: 2026-01-03 / local
- **Argo app**: `bootstrap-local` (Terraform ArgoCD bootstrap)
- **Unhealthy resource**: Helm install/upgrade step (bootstrap) failing with server-side apply conflicts
- **Symptom**: `kubectl apply --server-side` emits ownership/field manager conflicts during ArgoCD bootstrap.
- **Root cause**: Server-side apply conflicts with existing managers during rapid bootstrap/reconcile cycles.
- **Fix**: Force `--server-side=false` in bootstrap apply paths.
- **Verification**: `deploy.sh local` / bootstrap completes without SSA conflicts; Argo reconciles normally.

- **Date/Env**: 2026-01-03 / local
- **Argo app**: multiple (any pulling from `ghcr.io`)
- **Unhealthy resource**: `Pod/*` `ImagePullBackOff`
- **Symptom**: Local clusters fail to pull images from GHCR (missing pull secret / empty credentials).
- **Root cause**: Local secret seeding accepted empty `.env` values and silently skipped writing required credentials, preventing Vault/ExternalSecrets from producing `ghcr-pull` for namespaces.
- **Fix**: Make `seed-local-secrets.sh` fail fast when `GHCR_USERNAME` / `GHCR_TOKEN` are missing or empty.
- **Verification**: `Secret/ghcr-pull` exists and workloads start without manual secret patching.

- **Date/Env**: 2026-01-03 / local
- **Argo app**: N/A (developer tooling)
- **Unhealthy resource**: developer workflows (port-forwards killed)
- **Symptom**: `deploy.sh local` teardown killed unrelated port-forwards/processes via a broad `pkill`, causing flakey local dev sessions.
- **Root cause**: Process cleanup matched by name rather than port/PID ownership.
- **Fix**: Replace blanket `pkill` with targeted PID discovery (`lsof` → `kill`) for only the known port-forward ports.
- **Verification**: Rerunning `deploy.sh local` no longer terminates unrelated background processes.

- **Date/Env**: 2026-01-03 / local
- **Argo app**: `local-data-temporal`
- **Unhealthy resource**: `Job/ameide-local/temporal-db-schema` (PreSync hook)
- **Symptom**: Job stuck `ImagePullBackOff` with `401 Unauthorized` fetching `ghcr.io/ameideio/mirror/temporalio-admin-tools@...` while the Application sync stays `Running`.
- **Root cause**: Schema hook Job defaulted to a private GHCR mirror image and did not carry `imagePullSecrets`, so early bootstrap could deadlock before pull secrets converge.
- **Fix**: Override Temporal `schema.image.ref` (and `preflight.image.ref`) to public images in shared/local values.
- **Verification**: `temporal-db-schema` completes and `local-data-temporal` reaches `Synced/Healthy` on fresh bootstrap.

- **Date/Env**: 2026-01-03 / local
- **Argo app**: `local-data-db-migrations`, `local-agents`
- **Unhealthy resource**: `Deployment/ameide-local/agents`
- **Symptom**: Agents CrashLoopBackOff with `ERROR: relation "agents.nodes" does not exist` (seed upsert fails).
- **Root cause**: `local-data-db-migrations` rendered only hook resources, so Argo could report `Synced` without ever running the PreSync migration Job (hook-only desired state).
- **Fix**: Add a tracked non-hook resource (e.g. a tracker ConfigMap) so the Application performs an initial sync and executes the migration hook Job deterministically.
- **Verification**: `data-db-migrations` Job runs and completes; `agents` Deployment becomes `Healthy` without manual schema creation.

## Update (2025-12-14): Local GitOps health fixes (devcontainer + k3d)

### Update (2026-01): Devcontainer-service is optional/disabled in GitOps

- The `platform-devcontainer-service` component lives under `environments/_optional/**` and is not part of the default ApplicationSet component discovery.
- `sources/values/_shared/platform/platform-devcontainer-service.yaml` is currently a disabled placeholder (waiting on a digest-pinned runtime image + final deployment shape).
- Any “devcontainer service” scheduling/health incidents below should be read as historical context from earlier experiments, not as a currently enabled baseline service.

- Fixed `local-platform-keycloak-realm` being blocked by a failing PreSync hook: `platform-keycloak-realm-client-patcher` is now reproducible (no runtime GitHub tool downloads; Keycloak Admin REST + Kubernetes API patch for rotation ConfigMap).
- Fixed `local-platform-gateway` sync failure on cross-namespace `ReferenceGrant` rendering: ensure core-group `group: ""` is rendered as a string (quoted), not YAML null.
- Verified the local set converges after sync: `local-platform-gateway`, `local-inference`, `local-inference-gateway`, `local-data-pgadmin` reach `Synced/Healthy`.
- Fixed local “red pod noise” from provisioning/bootstrap jobs:
  - MinIO provisioning Job was stuck `ImagePullBackOff` on `ghcr.io/ameideio/mirror/bitnami-os-shell:latest` due to missing `imagePullSecrets` and a single-arch mirror tag.
  - Vault bootstrap Job had a transient failure due to runtime `kubectl` download TLS flakiness; bootstrap jobs should avoid runtime downloads (track under 519).
  - Vault bootstrap local overlay briefly regressed after adding digest support: overriding `bootstrap.kubectl.image.repository` without also overriding/unsetting `bootstrap.kubectl.image.digest` can render an invalid `repo@sha256:...` reference. Local now inherits the shared pinned image/digest to avoid value-merge footguns.

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

## Update (2025-12-17): Local `SecretStore/ameide-vault` Degraded (`permission denied`) from Vault Kubernetes auth audience binding

- **Env:** `local` (k3d)
- **App:** `local-foundation-vault-secret-store` (and any downstream app that depends on ESO secrets)
- **Symptom:** `SecretStore/ameide-vault` in `ameide-local`, `argocd`, `ameide-system` remains `Degraded` with:
  - `Code: 403` + `permission denied` from `PUT /v1/auth/kubernetes/login`
- **Root cause:** Vault Kubernetes auth roles were written with an `audience` that did not match the audience of tokens minted by External Secrets Operator’s TokenRequest flow (local k3s commonly requests an audience such as `vault`). With strict audience binding enabled, Vault rejects the login.

Remediation approach (vendor-aligned, reproducible):
1. Do not hardcode a cluster-specific service-account audience in local overlays unless all clients mint matching tokens.
2. Prefer leaving the Vault role `audience` unset (accept all audiences) until a single, explicitly configured token audience is enforced across ESO and any other Vault clients.

## Update (2025-12-31): Local `SecretStore/ameide-vault` 403 flapping from unstable `token_reviewer_jwt` (projected SA token)

- **Env:** `local` (k3d)
- **Symptom class:** `SecretStore/ameide-vault` intermittently flips `Ready=False` with:
  - `Code: 403` + `permission denied` from `PUT /v1/auth/kubernetes/login`
  - Downstream `ExternalSecret` resources fail with `SecretStore "ameide-vault" is not ready`, and Argo apps go `Degraded`.
- **Root cause (vendor-aligned):** Vault Kubernetes auth uses the Kubernetes **TokenReview** API. If Vault’s reviewer credentials become invalid/expired/revoked (common when `token_reviewer_jwt` is copied from a Pod’s projected service account token in Kubernetes 1.21+), Vault cannot complete TokenReview and login fails as `permission denied` even though the client role bindings are correct.
- **Diagnostic:** check Vault’s Kubernetes auth config:
  - `vault read -format=json auth/kubernetes/config` → `token_reviewer_jwt_set=true` is a red flag in in-cluster Vault if it was sourced from a Job/Pod token.
- **Fix (repo):** stop writing `token_reviewer_jwt` (and stop pinning `kubernetes_ca_cert`) in `foundation/vault-bootstrap`, and let Vault load/re-read the in-cluster token/CA (`disable_local_ca_jwt=false`). See `ameide-gitops` `sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml`.

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

## Update (2025-12-17): `argocd-config` Degraded from broken local component symlinks

- **Env:** `local` (k3d)
- **App:** `argocd-config` (specifically `ApplicationSet/ameide`)
- **Symptom:** ApplicationSet health `Degraded` with repo-server errors like:

## Update (2025-12-18): SSO bootstrap + secret pipeline reliability (local + azure)

### Symptom(s)

- Local ArgoCD SSO intermittently fails after a fresh cluster create with:
  - `unauthorized_client Invalid client credentials` (Dex ↔ Keycloak)
  - Or the Dex server crashloops early because `auth.local.ameide.io` isn’t routable yet
- Local `argocd/argocd-secret` can remain stuck with a placeholder Dex client secret for up to 1 hour.
- Azure CI was repeatedly failing post-apply verifications despite core components being healthy.

### Root causes

1. **Client reconciliation ordering bug (dev/staging redirect_uri rejected)**:
   - `platform-keycloak-realm-client-specs` is the input for the `client-patcher` hook Job, but it was not explicitly ordered before the hook.
   - In first-sync scenarios, the hook could run before the ConfigMap existed, so `platform-app` was created without redirect URIs → Keycloak returned `Invalid parameter: redirect_uri`.
2. **Secret extraction silently skipped due to insufficient Keycloak token permissions**:
   - The `client-patcher` authenticated via client credentials (from `keycloak-admin-sa`) first; that token can patch clients but may not have access to the `/client-secret` endpoint.
   - `get_client_secret()` returned empty → extraction treated the client as “public” and skipped writing the real secret to Vault, leaving bootstrap placeholders in place.
3. **Slow secret convergence window (1h) + env-var injection requires restart**:
   - With `refreshInterval: 1h`, ExternalSecrets could sync placeholder values for a long time after a fresh bootstrap or a Keycloak secret rotation.
   - Apps consuming secrets via env vars require a pod restart to pick up updated Secrets; without quick refresh + restart, users see `unauthorized_client` until convergence.

### Local follow-up (2025-12-18): enable platform SSO verifier end-to-end

- `infra/scripts/verify-platform-sso.sh --local` was previously skipping the full platform SSO flow because `www-ameide-platform` was disabled in the local overlay (no `Service`/`Deployment` rendered).
- Local enablement required:
  - Fixing the devcontainer kubecontext: `ameide-local` must target `https://host.docker.internal:6550` (not `https://127.0.0.1:6550`) or `kubectl`/Terraform operations intermittently fail.
  - Running Redis in Sentinel mode locally (and enabling `redis-operator`) because `www-ameide-platform` uses Sentinel discovery for session storage.
- Current status: `www-ameide-platform` is enabled locally and `infra/scripts/verify-platform-sso.sh --local` passes end-to-end.

### Update (2026-01-11): local Redis is standalone (no Spotahome operator / Sentinel)

We hit recurring local instability from the Spotahome `redis-operator` CrashLooping during leader-election renewal under k3d/k3s, leaving the local cluster permanently not-green.

Decision (GitOps-owned, no band-aids):

- Local runs `data-redis-failover` in `mode: standalone` (plain Redis StatefulSet; no `RedisFailover` CR).
- Local disables installing the cluster-scoped `redis-operator`.
- Local `www-ameide-platform` uses a direct Redis URL (`redis-master:6379`) instead of Sentinel discovery.

AKS dev/staging/prod can continue to use RedisFailover + Sentinel where HA is required.

## Update (2025-12-18): Azure CI destroy reliability (PDBs + state locks)

### Symptom(s)

- GitHub Actions “Terraform Azure Destroy (Manual)” would hang or fail in two recurring ways:
  - **AKS User nodepools failed to delete** due to eviction failures (“would violate PDB”), leaving the cluster Running and keeping Azure Load Balancer / Public IP attachments alive.
  - **Terraform remote state got stuck locked** (Azure Storage blob lease), causing later runs to sit at `Acquiring state lock` until timeout.

### Root causes

1. **PDBs blocking node drain**: AKS nodepool deletion triggers Kubernetes eviction; restrictive PDBs (e.g., Postgres primary MinAvailable=1) can prevent node drain, and the Azure API returns terminal provisioning state `Failed`.
2. **Workflow cancellation creating stale locks and drift**: cancelled/aborted CI runs can leave the remote state locked, and can also create partial Azure resources without committing state.

### Remediation (CI-first, no manual cluster operations)

- Azure lifecycle is driven through CI workflows:
  - `.github/workflows/terraform-azure-destroy.yaml` pre-deletes **User** nodepools with `--ignore-pdb` and blocks until they are actually gone before running Terraform destroy.
  - `.github/workflows/terraform-azure-force-unlock.yaml` provides an explicit “break glass” mechanism to clear a known lock ID (manual confirmation required).
  - `.github/workflows/terraform-azure-plan.yaml` runs `terraform plan` with `-lock=false` so PR plans do not block apply/destroy.
- Apply is **manual-only** and requires explicit confirmation (`apply-azure`) to avoid accidental creation and “cancel mid-apply” drift.

### Current status

- CI destroy is now reproducible end-to-end (cluster + nodepools removed, and attached Public IPs delete cleanly once AKS is gone).
- Verification: GitHub Actions run `20344623080` completed successfully; `az aks show -g Ameide -n ameide` returns NotFound and the `MC_Ameide_ameide_westeurope` node resource group is gone. The base resource group `Ameide` still contains shared resources (e.g., tfstate storage / Key Vault / identities / DNS zone) depending on what is managed by Terraform vs treated as shared prerequisites.

## Update (2025-12-18): Azure CI apply verification race (LB/PIP attach timing)

### Symptom(s)

- `terraform-azure-apply.yaml` completes `terraform apply`, but the first SSO verifier fails with:
  - `curl: (28) Failed to connect to auth.ameide.io:443 ... Couldn't connect to server`

### Root cause

- The Public IPs exist and DNS points at them, but **AKS has not yet associated them to a Load Balancer frontend** (no `ipConfiguration.id` on the Public IP), because the GitOps layer (Envoy / Keycloak ingress) has not converged yet.

## Update (2025-12-18): Azure HTTPS missing (no 443 LB rules) due to cert-manager DNS-01 auth failure

### Symptom(s)

- DNS `A` records resolve correctly to the Terraform-managed Public IPs (e.g. `auth.ameide.io -> <envoy-prod-pip>`), but:
  - `curl https://auth.ameide.io/...` returns `HTTP 000` / connect timeout
  - Azure `kubernetes` Load Balancer rules exist for `80/8000/9000`, but **no rule for `443`**
- Envoy Gateway `Gateway` resources show `Programmed=True`, but individual **HTTPS listeners are not programmed**.

### Root cause

- HTTPS listeners were invalid because required TLS Secrets were missing:
  - `argocd/argocd-ameide-io-tls` (ArgoCD)
  - `ameide-*/ameide-wildcard-tls` (platform)
- Those Secrets were missing because cert-manager ACME DNS-01 challenges failed with Entra ID auth errors:
  - `AADSTS700016: Application with identifier '<clientId>' was not found in the directory ...`
- The cluster was using **stale / wrong** Azure Workload Identity client IDs in Helm values, and (separately) Terraform had DNS federated identity credentials behind a flag (`enable_dns_federated_identity=false`), so even the correct identities were not federated.

### Remediation (Terraform + GitOps, no manual cluster poking)

- **Terraform**:
  - Default `enable_dns_federated_identity=true` (and backups) so the identities created by Terraform always have the required federated credentials for the cert-manager ServiceAccount.
- **GitOps values**:
  - Update `sources/values/cluster/azure/cert-manager.yaml` + `sources/values/*/globals.yaml` to match the Terraform-created identity client IDs and Public IPs.
  - **Stop hardcoding infra identifiers**: treat Managed Identity client IDs and Public IPs as *generated outputs* and sync them from Terraform into GitOps via `infra/scripts/sync-globals.sh` (CI apply runs this and commits/pushes back to git).
- **CI**:
  - After `terraform apply`, wait for:
    - Public IP association (`ipConfiguration.id`)
    - Certificates `Ready=True`
    - Envoy LoadBalancer Services to expose port `443`
  - Only then run the SSO verifiers.

### Remediation

- Add a deterministic CI gate to **wait for the Public IPs to be associated** before running the SSO verifiers (removes timing flakiness without changing cluster configuration).

## Update (2025-12-18): Azure bootstrap could not sync GitOps repo (repo-creds mis-detected public access)

### Symptom(s)

- `argocd-config` app stays `sync=Unknown` with:
  - `failed to list refs: authentication required: Repository not found.`
- Only bootstrap apps exist (`argocd-config`, `argocd-tls`); platform apps never get created, so Envoy/Keycloak never deploy and Public IPs remain unassociated.

### Root cause

- The bootstrap `repo_creds` job tested “is the repo public?” by running `git ls-remote` from inside the checked-out workspace.
- In GitHub Actions, `actions/checkout` injects Git credentials into the workspace repo (e.g. `http.extraheader` in `.git/config`), so the “anonymous” check could succeed even when the repo is private.
- Result: the bootstrap created `repo-creds-ameide-gitops` **without** username/password, so in-cluster ArgoCD could not fetch the repo.

### Remediation

- Run the anonymous `git ls-remote` probe from a temp directory (not inside the workspace repo) and explicitly ignore global/system git configs.
2. **Bootstrap deadlock on Dex**:
   - Dex depends on reaching the external Keycloak URL (`https://auth.<env>.ameide.io/...`) which is not guaranteed to exist during first bootstrap.
   - Helm `--wait` gates the release on Dex readiness, creating a chicken-and-egg loop (ArgoCD can’t reconcile Keycloak until ArgoCD is installed).

### Fix shipped (deterministic + reproducible)

- **Make `client-patcher` run *before* ExternalSecrets**:
  - `sources/charts/foundation/operators-config/keycloak_realm/templates/client-patcher-job.yaml`
  - Changed hook from `PostSync` → `Sync` and added a realm availability wait (operator reconciliation gate) before extracting and writing secrets to Vault.
- **Avoid bootstrap gating on Dex**:
  - `infra/terraform/modules/argocd-bootstrap/main.tf` now waits for ArgoCD CRDs + core components only (not Dex).
  - `bootstrap/lib/argocd.sh` does the same for the bootstrap-v2 workflow.
- **Speed up Dex secret propagation**:
  - `sources/values/_shared/cluster/vault-secrets-argocd.yaml` changes `ExternalSecret/argocd-secret-sync` refresh from `1h` → `1m`.

### Current status

- **Local**:
  - `infra/scripts/verify-argocd-sso.sh --local` ✅ passes end-to-end.
  - `infra/scripts/verify-platform-sso.sh --local` ✅ link check passes; platform flow may `SKIP` if `www-ameide-platform` is not deployed.
- **Azure**:
  - `infra/scripts/verify-argocd-sso.sh --azure` ✅ passes across dev/staging/production.
  - `infra/scripts/verify-platform-sso.sh --azure` ❌ currently fails the `www.*` login link gate (platform itself can complete SSO).

### Remaining blocker (not solvable in GitOps alone)

- `www.*` must render an environment-correct login link to `https://platform.<env>.ameide.io/login` **without baking configuration into the image**.
- Until the `www-ameide` image reads runtime configuration, CI “Apply + Verify” will keep failing the platform verifier even if cluster + SSO plumbing is correct.
- `unable to read files... pattern environments/local/components/platform/**/component.yaml: open .../platform/developer/devcontainer-service/component.yaml: no such file or directory`
- **Root cause:** local curated `component.yaml` entries were committed as **git symlinks** pointing into `environments/_shared/**`. After refactoring optional workloads (e.g., `gitlab`, `devcontainer-service`) out of `_shared` into `environments/_optional/**`, those symlink targets no longer existed. Argo CD repo-server follows symlinks when reading git-generator files, so the generator fails hard and stops producing Applications.
- **Remediation approach (vendor-aligned, GitOps-idempotent):**
  1. Keep optional workloads out of the baseline local component set (store them under `environments/_optional/**`).
  2. Avoid symlink-based component definitions for ApplicationSet git generators; prefer real `component.yaml` files (or stable, non-moving targets) to keep refactors safe.
  3. Add a CI/pre-commit guard to fail on **broken symlinks** under `environments/**` to prevent a recurrence.

## Update (2025-12-17): AKS `plausible-oauth2-proxy` CrashLoop (OIDC discovery) + RedisFailover auth drift (Langfuse)

### Plausible oauth2-proxy (OIDC discovery timeouts)

- **Apps:** `dev-plausible-oauth2-proxy`, `staging-plausible-oauth2-proxy`, `production-plausible-oauth2-proxy`
- **Symptom:** oauth2-proxy exits on startup with:
  - `ERROR: Failed to initialise OAuth2 Proxy ... failed to discover OIDC configuration ... dial tcp <env-public-ip>:443: i/o timeout`
- **Root causes (combined):**
  - **In-cluster calls to public env IPs are not reliable** on AKS (hairpin/NAT path). oauth2-proxy must not depend on reaching `auth.<env>.ameide.io` via the public IP from inside the cluster.
  - `hostAliases` pinned `auth.<env>.ameide.io` to the env public IP, bypassing DNS and forcing the failing network path.
  - With default `ndots:5`, some clients resolve names via search-domain expansion (e.g., `auth.dev.ameide.io.<ns>.svc.cluster.local`) if the query is not treated as absolute; this can bypass intended rewrite behavior.
- **Remediation (GitOps, vendor-aligned):**
  1. Manage CoreDNS `coredns-custom` at cluster scope (backlog/454) with rewrite rules so `*.dev.ameide.io`, `*.staging.ameide.io`, `*.ameide.io`, and `argocd.ameide.io` resolve to in-cluster Envoy Service aliases.
  2. Remove `hostAliases` from the Plausible oauth2-proxy chart values; rely on CoreDNS rewrites.
  3. Set `dnsConfig.options.ndots: "1"` and add a startupProbe so oauth2-proxy isn’t killed while doing initial OIDC discovery.

## Update (2025-12-17): SSO verification results (Azure + Local)

Using the repo verifiers:
- `infra/scripts/verify-argocd-sso.sh --all`
- `infra/scripts/verify-platform-sso.sh --azure`
- `infra/scripts/verify-platform-sso.sh --local`

Observed:
- **ArgoCD SSO**: ✅ green for `local` + all Azure envs (Dex ↔ Keycloak wiring, secrets, and session established).
- **Platform SSO (staging/prod)**: ✅ end-to-end SSO green (CSRF cookies + Keycloak redirect + session established).
- **`www` login link (staging/prod)**: ❌ `www.staging.ameide.io` and `www.ameide.io` render `href="//login"` instead of `https://platform.{env}.ameide.io/login`. This points to **application build/runtime-config behavior** in `www-ameide` (common root cause: relying on `NEXT_PUBLIC_*` at build time for `next start` images).
- **Platform dev**: ⚠️ observed intermittent failures while verifying (including `HTTP 500` from `platform.dev.ameide.io`); `verify-platform-sso.sh --azure` still intermittently fails at “session is not established” even when CSRF + Keycloak redirect succeed.

GitOps hardening applied (prepared for rollout):
- Removed baked environment defaults from shared Keycloak operator values (`_shared/platform/platform-keycloak.yaml`), so env overlays are the source of truth.
- Keycloak realm: ensured OIDC client reconciliation is defined per env (and fixed the Kubernetes Dashboard client ID mismatch by standardizing on `clientId=kubernetes-dashboard` and adding env-specific redirect URIs).
- Kubernetes Dashboard: deployed via GitOps (Dashboard Helm chart + oauth2-proxy + Gateway route). Note: upstream Dashboard still requires a Kubernetes bearer token; local dev may optionally use a token-injecting proxy behind SSO (shared `view` identity) for “hit URL → dashboard” UX.

### RedisFailover auth templating (breaks master election → Langfuse worker CrashLoop)

- **Apps:** `staging-platform-langfuse`, `production-platform-langfuse` (worker), plus `*-data-redis-failover`
- **Symptom (Langfuse):** worker loops with Redis connection failures (`ECONNREFUSED`/`ETIMEDOUT`) because `redis-master` has no endpoints.
- **Root cause:** RedisFailover master election never succeeds because the operator cannot authenticate to Redis pods:
  - Operator logs show `Make new master failed ... WRONGPASS invalid username-password pair`.
  - `Secret/redis-auth` was rendered with a literal string password (`{{ .REDIS_PASSWORD }}`) due to double-templating (Helm template inside ESO template).
- **Remediation (GitOps):**
  1. Fix the `redis-auth` ExternalSecret template so it writes the real password value and sets the `default` ACL to require that password.
  2. Restart Redis pods so the updated auth config is applied (operator-managed StatefulSets can use `OnDelete`, so a pod delete may be required for rollout).

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

## Update (2025-12-17): AKS non-green apps (taints block operator-managed Jobs, CoreDNS rewrite + Envoy alias drift)

- **Env**: `azure` (AKS)
- **Symptom**: a small set of apps remain non-green even though most of the fleet is `Synced/Healthy`:
  - `*-data-temporal` = `Synced/Degraded`
  - `*-data-nifi` = `Sync=Unknown` (`ComparisonError`)
  - `*-data-pgadmin` = `Synced/Degraded`
  - `*-plausible-oauth2-proxy` = `Synced/Degraded` (CrashLoop)
  - `*-platform-devcontainer-service` = `Synced/Degraded` (Pods Pending)
  - `*-platform-gitlab` = `OutOfSync/Missing`

### Update (2025-12-17): ArgoCD UI and Gateways unreachable (Azure LB provisioning 403)

- **Env:** `azure` (AKS)
- **Symptom:** `https://argocd.ameide.io/` is not reachable (TCP connect timeout), even though:
  - `Gateway/argocd/cluster` reports `Accepted=True` and `Programmed=True`.
  - `Certificate/argocd-server` is `Ready=True` and `HTTPRoute/argocd` is `Accepted=True`.
- **Observed evidence:**
  - `curl -vkI https://argocd.ameide.io/` hangs and times out.
  - The Envoy data-plane Service is **not provisioned** by Azure; it only has `spec.externalIPs` set (no `status.loadBalancer.ingress`).
  - `kubectl -n argocd describe svc envoy-argocd-cluster-*` shows repeated cloud-provider errors:
    - `ERROR CODE: AuthorizationFailed`
    - `does not have authorization to perform action 'Microsoft.Network/publicIPAddresses/write'`
    - client id `ae2a752e-153d-4f61-b7f0-a7d7e3852025` / object id `688bc921-26a0-4417-be3f-cdbc751cf9e9` (AKS cluster user-assigned identity `ameide-aks-mi`)
- **Root cause:** the Azure cloud provider tries to `CreateOrUpdate` a per-Service Public IP named `kubernetes-<service-uid>` in RG `Ameide` (instead of binding the existing Terraform-managed PIP). With least-privilege scoping, the AKS managed identity cannot write new PIPs in the RG, so the LoadBalancer never provisions and the endpoint times out.
- **Remediation (Terraform-first, reproducible):**
  1. Keep least-privilege: grant `ameide-aks-mi` `Network Contributor` on the **Terraform-managed** Public IP resources (ArgoCD PIP + per-env Envoy PIPs), not RG-wide.
  2. Bind Services to the Terraform-managed PIPs **by name** (cloud-provider supported): set `service.beta.kubernetes.io/azure-pip-name: <pip-name>` (and `service.beta.kubernetes.io/azure-load-balancer-resource-group`) so the cloud provider attaches the existing PIP and does not attempt to create `kubernetes-*` PIPs.
  3. Re-verify with: `kubectl -n argocd get svc envoy-argocd-cluster-* -o wide` (expect `status.loadBalancer.ingress.ip`), then `curl -I https://argocd.ameide.io/`.

### Update (2026-01-14): ArgoCD endpoint times out despite LoadBalancer provisioned (`externalTrafficPolicy: Local` + zero ready endpoints)

- **Env:** `azure` (AKS)
- **Symptom:** `https://argocd.ameide.io/` is not reachable (TCP connect timeout), even though the Envoy LoadBalancer Service has a valid `status.loadBalancer.ingress.ip`.
- **Observed evidence:**
  - `Service/envoy-argocd-cluster-*` has `type=LoadBalancer` and a stable Public IP, but `curl -I https://argocd.ameide.io/` hangs/times out.
  - The Service uses `externalTrafficPolicy: Local` and `Endpoints/envoy-argocd-cluster-*` shows **no ready addresses** (only `notReadyAddresses`).
  - `Gateway/argocd/cluster` reports `Programmed=False` with `NoResources` (Envoy replicas unavailable), and Envoy readiness returns `503` with `x-envoy-immediate-health-check-fail: true`.
- **Root cause:** `externalTrafficPolicy: Local` makes the Azure LB send traffic only to nodes with a **local ready endpoint**. With a single (or effectively single) ready proxy pod, any node/pod readiness issue can blackhole the public endpoint.
- **Fix (GitOps, target state):**
  - Set the cluster gateway Envoy Service to `externalTrafficPolicy: Cluster` and run **2 proxy replicas** with a zero-downtime rollout strategy (`maxUnavailable: 0`) and a proxy PDB (`minAvailable: 1`).
  - Code reference: `ameide-gitops` commit `3278ca33` (“fix(cluster-gateway): harden ArgoCD ingress”).
- **Related reliability fix:** ArgoCD reconciliation can be disrupted by `argocd-repo-server` restart loops when probes are too aggressive under load; we relaxed probe timeouts via `ameide-gitops` commit `82023553` (“fix(argocd): relax repo-server probe timeouts”).

### Update (2025-12-17): ArgoCD SSO broken (Dex upstream Keycloak issuer misconfigured)

- **Env:** `azure` (AKS)
- **Symptom:** ArgoCD SSO/Dex endpoints return 502 and clients fail to discover OIDC:
  - `https://argocd.ameide.io/api/dex/.well-known/openid-configuration` returns `502`
  - Error example:
    - `failed to query provider "https://argocd.ameide.io/api/dex": Get "https://argocd-dex-server:5556/api/dex/.well-known/openid-configuration": dial tcp 10.0.219.230:5556: connect: connection refused`
- **Root cause:** `dex.config` pointed at an in-namespace Keycloak hostname (`issuer: http://keycloak:8080/realms/ameide`), but Keycloak runs in the environment namespaces and is exposed externally as `https://auth.ameide.io`. Dex failed to initialize its Keycloak connector → it never bound port `5556` → ArgoCD server proxying `/api/dex` returned 502.
- **Remediation (GitOps + reproducible bootstrap):**
  1. Set Dex Keycloak issuer to the canonical external issuer (`https://auth.ameide.io/realms/ameide`) so browser redirects work.
  2. Keep local clusters on `https://auth.local.ameide.io/realms/ameide` via local ArgoCD bootstrap override.
  3. Upgrade the ArgoCD Helm release (Terraform-driven) so `argocd-cm` + Dex roll out and `/api/dex` becomes reachable.

### Update (2025-12-17): ArgoCD SSO fails in Keycloak (Invalid `redirect_uri`)

- **Env:** `azure` (AKS) + `local`
- **Symptom:** Keycloak login flow fails immediately with:
  - `We are sorry... Invalid parameter: redirect_uri`
  - Example (AKS): `https://auth.ameide.io/realms/ameide/protocol/openid-connect/auth?client_id=argocd&redirect_uri=https%3A%2F%2Fargocd.ameide.io%2Fapi%2Fdex%2Fcallback...`
  - Example (local): `https://auth.local.ameide.io/realms/ameide/protocol/openid-connect/auth?client_id=argocd&redirect_uri=https%3A%2F%2Fargocd.local.ameide.io%2Fapi%2Fdex%2Fcallback...`
- **Root cause:** the Keycloak `argocd` client did not include the Dex callback URL(s) in `redirectUris` (it only allowed `/auth/callback`, but Dex uses `/api/dex/callback`), and local hostnames were not registered.
- **Remediation (GitOps, idempotent):**
  1. Define the `argocd` client in `platform-keycloak-realm` `clientReconciliation` with explicit `redirectUris` and `webOrigins` for the ArgoCD URL(s).
  2. Enable “update existing” reconciliation so already-created clients are patched (Keycloak realm import is create-only).
- **Verification:**
  - Visiting ArgoCD SSO should present the Keycloak login page (no immediate `redirect_uri` error).
  - `curl -fsS 'https://auth.ameide.io/realms/ameide/protocol/openid-connect/auth?...redirect_uri=https%3A%2F%2Fargocd.ameide.io%2Fapi%2Fdex%2Fcallback...'` returns an HTML login page (not an “Invalid parameter” response).

### Update (2025-12-17): ArgoCD SSO fails (Dex/Keycloak `invalid_scope`, missing built-in scopes)

- **Env:** `azure` (AKS) + `local`
- **Symptom:** ArgoCD login fails with an internal Dex error page:
  - `Failed to authenticate: invalid_scope: Invalid scopes: openid openid profile email groups`
  - ArgoCD server logs show it initiating `/api/dex/auth?...scope=openid+profile+email+groups+...` followed by “received error from dex”.
- **Root cause:** Keycloak rejects the requested scopes for the `argocd` client.
  - Variant A (realm-level): the `ameide` realm can be missing built-in OIDC client scopes (`profile`, `email`) when realm JSON defines `clientScopes[]` but omits vendor defaults; Keycloak treats the list as authoritative (no implicit merge). See `backlog/460-keycloak-oidc-scopes.md`.
  - Variant B (client-level, observed 2025-12-17): the realm advertises the scopes in discovery, but the `argocd` client is not linked to them as default/optional client scopes, so Keycloak returns `invalid_scope` for `scope=openid profile email groups`.
    - Evidence: Keycloak event logs show `reason="Invalid scopes: openid profile email groups"` for `clientId="argocd"` while `.well-known/openid-configuration` `scopes_supported` includes `profile`, `email`, `groups`.
    - Contributing gap: updating the client JSON representation alone is insufficient; client-scope linkage must be applied via the Keycloak Admin API endpoints for default/optional client scopes.
- **Remediation (GitOps, reproducible):**
  1. Ensure realm template includes the built-in `profile`/`email` client scopes (see `backlog/460-keycloak-oidc-scopes.md`).
  2. In `platform-keycloak-realm` `client-patcher`, reconcile both:
     - realm scopes (create `profile`/`email` if missing), and
     - per-client scope linkage (attach `profile`/`email`/`groups` to `argocd` via `/clients/{id}/default-client-scopes/{scopeId}`).
  3. Re-run `platform-keycloak-realm-client-patcher` (via Argo sync).
  4. Verify SSO:
     - Keycloak auth endpoint returns login HTML (no `invalid_scope` redirect):
       - `https://auth.ameide.io/realms/ameide/protocol/openid-connect/auth?client_id=argocd&redirect_uri=https%3A%2F%2Fargocd.ameide.io%2Fapi%2Fdex%2Fcallback&response_type=code&scope=openid+profile+email+groups`
     - ArgoCD `/auth/login` reaches the Keycloak login page and completes without redirecting to `/login?has_sso_error=true`.

### Update (2025-12-17): ArgoCD SSO fails (Dex `unauthorized_client`, client secret mismatch)

- **Env:** `azure` (AKS) + `local`
- **Symptom (Dex logs):** `oidc: failed to get token: oauth2: "unauthorized_client" "Invalid client or Invalid client credentials"`
- **Root cause:** ArgoCD is not consuming the Keycloak-generated `argocd` client secret:
  - `argocd-secret` is still populated from ArgoCD Helm values with a placeholder (`configs.secret.extra.dex.keycloak.clientSecret`).
  - The intended GitOps flow (“Keycloak client-patcher extracts → Vault → ExternalSecret → `argocd-secret`”) was not applied because:
    - `ExternalSecret/argocd-secret-sync` is not deployed, and
    - `SecretStore/ameide-vault` is missing in the `argocd` namespace on AKS.
- **Remediation (GitOps, reproducible):**
  1. Ensure `SecretStore/ameide-vault` exists in `argocd` (typically from the production Vault via `foundation-vault-secret-store.extraSecretStores`).
  2. Deploy `ExternalSecret/argocd-secret-sync` in `argocd` to materialize `Secret/argocd-secret` keys used by Dex (`dex.keycloak.clientSecret`, `dex.oauth.clientSecret`) and ArgoCD (`server.secretkey`, `admin.password`, `admin.passwordMtime`).
  3. Verify Dex initializes and the error disappears.
- **Verification (no secret leakage):**
  - `kubectl -n argocd get externalsecret argocd-secret-sync`
  - `kubectl -n argocd describe externalsecret argocd-secret-sync | rg -n "Ready|Error|SecretSynced"`
  - `kubectl -n argocd logs deploy/argocd-dex-server --tail=200 | rg -n "unauthorized_client|invalid_client" || true`
  - End-to-end: `infra/scripts/verify-argocd-sso.sh --azure` and `infra/scripts/verify-argocd-sso.sh --local`

### Root cause A: all AKS node pools are tainted `NoSchedule`

- **Evidence**: scheduler events for Pods/Jobs show `0/N nodes are available: ... had untolerated taint {ameide.io/environment: <env>}` and `CriticalAddonsOnly=true:NoSchedule`.
- **Impact**:
  - **Temporal operator-managed schema Jobs** (e.g., `temporal-setup-default-schema`) are created without env tolerations/nodeSelectors (CRD does not expose these) → Jobs stay `Pending` → `TemporalCluster` never becomes Ready.
  - **Devcontainer service** pods (`devcontainer-coder`) cannot schedule (no tolerations) → Deployment fails progress deadline.
- **Remediation (vendor-aligned)**:
  - Do **not** taint all user pools in AKS by default. Keep `CriticalAddonsOnly` for the system pool if desired, but ensure at least one pool exists where standard workloads/Jobs can schedule without custom tolerations.
  - Track and implement this in Terraform (see `infra/terraform/modules/azure-aks/main.tf`).

**Update (2025-12-17): confirmed Temporal schema Job lacks tolerations**
- **Env/app:** `dev-data-temporal`, `staging-data-temporal`, `production-data-temporal`
- **Symptom:** `Job/<env>/temporal-setup-default-schema` Pods stuck `Pending` with scheduler message: `had untolerated taint {ameide.io/environment: <env>}`.
- **Evidence:** the Job is `ownerReferences: TemporalCluster/temporal` and the Pod template has **no** `tolerations`/`nodeSelector` (TemporalCluster `services.overrides` does not apply to this schema Job).
- **Fix strategy:** either (a) introduce a non-tainted pool for operator-managed Jobs, or (b) adopt a policy/mutating layer that injects env scheduling constraints into Jobs created in env namespaces.

**Update (2025-12-17): Staging Temporal remains Degraded (worker Pending due to node pool max)**
- **Env/app:** `staging-data-temporal`
- **Symptom:** `temporal-worker` Pod stuck `Pending` with scheduler `Insufficient cpu`, and cluster-autoscaler logs `max node group size reached` for the staging pool.
- **Root cause:** staging node pool `max_count` is too low for current workload density; Temporal worker requires an additional staging node but autoscaler cannot scale beyond the cap.
- **Fix (Terraform-first):** increase `staging_pool_max_count` in `infra/terraform/modules/azure-aks/variables.tf` (or override via tfvars) so autoscaler can add capacity.

**Update (2025-12-17): TemporalNamespace can stick `ReconcileError` if created before TemporalCluster**
- **Symptom:** `TemporalNamespace/ameide` shows `ReconcileError: TemporalCluster ... not found` even after the cluster exists (controller may not requeue on TemporalCluster creation).
- **Fix (GitOps-idempotent):** add explicit Argo sync-waves so `TemporalCluster` applies before `TemporalNamespace` (and forces a namespace update/reconcile when rolling out the change).

### Update (2025-12-17): Argo Healthy while workloads are Pending (operator-generated Deployments missing scheduling)

- **Env:** `azure` (AKS)
- **Symptom:** several env workloads show `Pod Pending` / `ProgressDeadlineExceeded`, but their Argo Applications remain `Synced/Healthy`.
- **Observed example:** `Deployment/ameide-dev/hello-v0-ui` has no tolerations/nodeSelector and is `ProgressDeadlineExceeded`, while the corresponding Application is `Healthy` because Argo health is derived from a higher-level primitive CR.
- **Impact:** fleet health dashboards can appear “green” while real workloads are not scheduled.
- **Remediation:** ensure operators propagate scheduling + readiness into their managed workloads (or add health customizations that treat non-ready generated workloads as unhealthy).

### Update (2025-12-17): Backstage Progressing/CrashLoop from Knex migration lock

- **Env/app:** `production-platform-backstage`
- **Symptom:** Argo shows `Progressing` (deployment has `0/1 available`); Pod `CrashLoopBackOff`.
- **Evidence:** logs contain `MigrationLocked: Plugin 'catalog' startup failed; caused by MigrationLocked: Migration table is already locked`.
- **Remediation (GitOps-idempotent):** see `backlog/467-backstage.md` (keep `replicas: 1` and ensure **no-overlap rollout** to avoid concurrent migrations; prefer `RollingUpdate` with `maxSurge: 0`/`maxUnavailable: 1` over `Recreate` if server-side diff/apply semantics make `Recreate` hard to transition to safely).
- **Follow-up issue (GitOps regression):** Argo can show `ComparisonError` and fail to render the unlock job:
  - **Symptom:** `YAML parse error on backstage/templates/migrations-unlock-job.yaml: yaml: line 55: could not find expected ':'`
  - **Root cause:** the unlock job template rendered an unindented heredoc terminator (`SQL`) at column 1, breaking YAML parsing.
  - **Fix:** indent the heredoc terminator within the YAML block scalar and keep the job script POSIX-`sh` compatible so it can run on `postgres:alpine`.
- **Follow-up issue (server-side diff):** Argo can show `ComparisonError` on the Backstage `Deployment`:
  - **Symptom:** `Deployment.apps "platform-backstage" is invalid: spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy type is 'Recreate'`
  - **Root cause:** the chart rendered a `strategy.rollingUpdate` stanza even when `strategy.type=Recreate`.
  - **Fix:** render `rollingUpdate` only when `strategy.type=RollingUpdate` (or omit it when type is `Recreate`) so Kubernetes validation passes.
- **Follow-up issue (unlock job runtime):** the unlock job can fail and block sync if the SQL wrapper is fragile:
  - **Symptom:** `Job ... has reached the specified backoff limit` with `ERROR: syntax error at or near "$"` from `psql`.
  - **Fix:** avoid `DO $$ ... $$` blocks in the job; use a `SELECT to_regclass(...)` probe and run a plain `UPDATE` only when the table exists.
- **Follow-up issue (job mutability):** changing a `Job` spec after it exists is not supported (pod template is immutable).
  - **Fix:** model the unlock as an Argo `PreSync` hook Job (delete before creation), so changing the script results in a fresh Job run without relying on in-place updates.
- **Follow-up issue (DB target):** Backstage/Janus can use per-plugin databases (e.g., `backstage_plugin_catalog`). The unlock job must target the DB(s) where `public.knex_migrations_lock` exists and is locked (not just `backstage`, and not a provider-default DB from a generic `POSTGRES_URL`).

### Root cause B: CoreDNS domain rewrite depends on an Envoy alias Service that is not stable

- **Evidence**: `kube-system/coredns-custom` rewrites `*.{dev,staging}.ameide.io` and `*.ameide.io` to `envoy.ameide-<env>.svc.cluster.local`.
- **Problem**: the `envoy` Service is an `ExternalName` alias that currently points to a hardcoded, non-existent/generated Envoy Gateway Service name (hash suffix), so in-cluster resolution for `auth.<env>.ameide.io` becomes unreliable.
- **Update (2025-12-22)**: the `envoy` alias is now rendered as a stable `ExternalName` to `envoy-<env-namespace>.<control-plane-namespace>.svc.cluster.local` (no hash suffix). Also, server-side gRPC callers should use the dedicated internal gateway endpoint (`http://envoy-grpc:9000`) per `backlog/589-rpc-transport-determinism.md`.
- **Impact**:
  - oauth2-proxy for Plausible fails OIDC discovery with `dial tcp ...:443: i/o timeout` against the rewritten hostname (`auth.<env>.ameide.io`), causing CrashLoop and `Degraded`.
- **Remediation (vendor-aligned)**:
  - Prefer **real DNS** (Terraform-managed A records + static IPs) on managed clusters; treat CoreDNS rewrites as **local-only** developer convenience.
  - If CoreDNS rewrite remains, the Envoy alias must point to a stable endpoint (not a controller-generated hashed Service name).

### Root cause C: NiFi chart requires a stable sensitive properties key, but values are not provided

- **Evidence**: Argo manifest generation error: `data/nifi: config.sensitivePropsKey is required`.
- **Remediation (GitOps-aligned)**:
  - Source the key via Vault KV → ESO → Secret, and configure the NiFi deployment/operator to consume it (avoid committing the plaintext key into Git).

### Root cause E: NetworkPolicy blocks `redis-system` from reaching Redis pods

- **Evidence**: `redis-operator` logs show repeated failures selecting a master with `dial tcp <podIP>:6379: i/o timeout`, while the environment namespace has a default-deny ingress policy (`NetworkPolicy/deny-cross-environment`) that does not allow ingress from `redis-system`.
- **Impact**:
  - `RedisFailover/redis` can have `0` masters and never populate `redis-master` endpoints.
  - Downstream apps that depend on Redis (e.g. Langfuse worker queues) CrashLoop with `ECONNREFUSED/ETIMEDOUT` to `redis-master:6379`.
- **Remediation**:
  - Add `redis-system` to the explicit allowlist in `deny-cross-environment` so the cluster-scoped Redis operator can probe/reconcile Redis in each env namespace.

### Root cause D: GitLab is present in the Azure component set but not intended/bootstrapped

- **Evidence**: `*-platform-gitlab` is `OutOfSync/Missing` across envs.
- **Remediation**:
  - Make cluster-type component sets explicit: exclude GitLab from Azure unless we are prepared to supply all required secrets, storage classes, and hook semantics (see policy in backlog/519).
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

## Update (2026-01-08): Local OTEL exporter noise (OTLP 4317 ClusterIP refused + collector RBAC spam)

- **Symptoms (apps):** workloads exporting OTLP metrics/traces spam errors like `PeriodicExportingMetricReader: metrics export failed ... connect ECONNREFUSED <cluster-ip>:4317`.
- **Symptoms (collector):** `otel-collector` logs spam reflectors with RBAC `forbidden` errors (pods/namespaces/replicasets list/watch at cluster scope).

Root causes:
1. **k3s/kube-proxy routing edge case:** `Service/otel-collector` port `4317` had `appProtocol: grpc`, which made the **ClusterIP port unroutable** (connection refused) even though the collector pod was listening.
2. **Missing RBAC for k8s enrichment:** collector config enables `k8sattributes` with `passthrough=false`, but the ServiceAccount lacked a ClusterRole/Binding, so enrichment could not work and logs spammed.

Remediation approach (GitOps-aligned):
1. In local, clear `ports.otlp.appProtocol` so `otel-collector:4317` is reachable for OTLP gRPC clients.
2. Enable the minimal ClusterRole (pods/namespaces/replicasets list/watch) and ensure ClusterRole/Binding names are unique per environment (avoid collisions across `ameide-dev/staging/prod` when using `fullnameOverride: otel-collector`).
3. Strengthen `platform-observability-smoke` to include OTEL collector OTLP sanity checks and a “no RBAC forbidden spam” check so these regressions fail fast during smoke syncs.

## Update (2026-01-08): Local transformation ingress stuck rollout (image missing ingress entrypoint)

- **App:** `local-process-transformation-v0-ingress`
  - **Resource:** `Deployment/ameide-local/transformation-v0-process-ingress`
  - **Symptom:** `Progressing=False (ProgressDeadlineExceeded)` with an old ReplicaSet still `Available=True` (Argo can show `Degraded`, but a naïve “deployment Available” smoke can still pass).
  - **Observed evidence:** new pod CrashLoopBackOff with `exec: "/app/process-transformation-ingress": stat /app/process-transformation-ingress: no such file or directory`.
  - **Root cause:** pinned image digest did not ship the ingress entrypoint binary expected by the Deployment command.

Remediation approach (GitOps-aligned):
1. Pin `process-transformation-v0-ingress` to an image digest that includes `/app/process-transformation-ingress` (or update the Deployment command to match the new image layout if the binary was intentionally renamed).
2. Keep rollout-aware smokes (e.g., `kubectl rollout status`) for Deployments to catch “Available old RS + broken new RS” states.

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
  - Related: `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md` (pin by digest/SHA so rollouts are Git-driven, not pullPolicy-driven).
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
3. Pin schema tooling images by **digest** (avoid `:latest`) so local arm64 and CI behave consistently.

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
| `transformation-*` | ameide-dev | `ghcr.io/ameideio/transformation@sha256:<digest>` (historically deployed via `:dev`; current target is digest-pinned per 602/603) |

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

### ImagePullBackOff in `ameide-system` (cluster-scoped operators) (2025-12-17)

**Status**: 🚧 OPEN (blocks `cluster-ameide-operators` health)

**Symptom**
- `cluster-ameide-operators` is `Degraded`.
- `ameide-system/ameide-operators-*` pods are `ImagePullBackOff` pulling from GHCR (missing pull secret and/or private image).

**Root cause**
- `ghcr-pull` is currently materialized only in **environment namespaces** (`ameide-dev`, `ameide-staging`, `ameide-prod`).
- Cluster-scoped operator Deployments run in `ameide-system` and already expect `imagePullSecrets: [ghcr-pull]`, but the Secret is missing in that namespace.
- Some controller images were also single-arch; on mixed clusters this surfaces as `no match for platform in manifest`.

**Remediation approach (GitOps-idempotent)**
1. Extend Vault bootstrap role binding to allow ESO auth from `ameide-system` (add it to `externalSecrets.namespaces`).
2. Ensure a `SecretStore/ameide-vault` exists in `ameide-system` that targets the stable cluster Vault backend (prefer `vault-core-prod`).
3. Materialize `ghcr-pull` in `ameide-system` via ExternalSecrets (reuse `foundation-ghcr-pull-secret` with `registry.additionalNamespaces: [ameide-system]`).
4. Ensure any cluster-scoped controller image is **multi-arch** and deployed by **digest** in GitOps (promotion PRs copy digests forward per 602/603).
5. Verify operator pods pull successfully and `cluster-ameide-operators` becomes `Healthy`.

### `ameide-operators-agent` CrashLoopBackOff (2025-12-17)

**Status**: 🚧 OPEN (blocks `cluster-ameide-operators` health)

**Symptom**
- `ameide-system/ameide-operators-agent` starts, then exits with `unable to configure transformation client` / gRPC dial timeout.

**Root cause**
- Cluster-scoped Agent operator defaulted to `TRANSFORMATION_SERVICE_ADDRESS=transformation.ameide.svc.cluster.local:3007`, but the shared AKS cluster has no `ameide` namespace and no cluster-scoped Transformation service.

**Remediation approach (GitOps-idempotent)**
- Treat the Agent operator as **environment-scoped** until it supports multi-env endpoint discovery:
  - Disable `operators.agent` in the Azure cluster values for `cluster-ameide-operators`, keeping it enabled for local where the wiring is known-good.
  - Follow-up: either deploy one Agent operator per environment namespace (each pointing at `transformation.ameide-<env>...`) or update the operator to derive the Transformation endpoint from the reconciled resource’s namespace.

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
| `www-ameide-platform` ImagePullBackOff | staging, prod | Digest points to missing/private or wrong-arch image, or missing pull secret | Ensure digest-pinned refs are valid and pull secrets exist; staging/prod move only by promotion PRs (602/603). |
| `plausible-seed` ImagePullBackOff | staging, prod | Digest points to missing/private or wrong-arch image, or missing pull secret | Ensure digest-pinned refs are valid and pull secrets exist; staging/prod move only by promotion PRs (602/603). |
| `workflows-runtime` CrashLoopBackOff | staging, prod | Runtime regression or missing dependencies (not a tag rollout mechanism issue) | Treat as app/runtime issue; promotion PRs should be gated by staging validation (602/603). |
| Bootstrap jobs Pending | staging, prod | Helm hooks need ArgoCD re-sync | Delete jobs and sync apps |

**Version bumping strategy (current target state)**:
- `local`/`dev`: automated PR write-back updates digest-pinned refs.
- `staging`/`production`: promotion PRs copy the exact same digests forward (human-gated).

The GitOps configuration should not rely on `:main`/`:dev` tags for rollouts; it should rely on digest-pinned refs per 602/603.

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
- AKV overrides were missed because `foundation-vault-bootstrap` was using a stale Azure Workload Identity client ID in `sources/values/env/*/foundation/foundation-vault-bootstrap.yaml` (did not match Terraform `vault_bootstrap_identity_client_id` output), so the bootstrap Job could not read Key Vault secrets and silently fell back to fixtures.

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

### Azure TLS stuck in `Issuing` (2025-12-17)

**Status**: 🚧 OPEN (blocks `argocd-config` + `platform-cert-manager-config` health)

**Symptom**
- `argocd-config` becomes `Degraded` because `argocd-tls` is `Degraded`.
- `argocd-tls` `Certificate/argocd-server` stays `Issuing` with `DoesNotExist` (TLS Secret not created).
- Per-environment wildcard certs (e.g. `Certificate/ameide-wildcard-dev`) stay `Issuing`, cascading into env `Gateway`/routes staying `Progressing`.

**Observed signals**
- `Challenge/argocd-server-*` shows `Presented=True` but repeatedly reports `Waiting for DNS-01 challenge propagation` in cert-manager logs.
  - External DNS resolution shows the TXT record is already visible publicly (e.g. `dig TXT _acme-challenge.argocd.ameide.io` returns the token), implying the *cluster’s recursive resolvers* used for propagation checks are stale/misleading.
- `Challenge/ameide-wildcard-*-*` fails with:
  - `error instantiating azuredns challenge solver: ClientID was omitted without providing one of --cluster-issuer-ambient-credentials or --issuer-ambient-credentials`
  - This affects **namespaced Issuers** (per-env `platform-cert-manager-config`) and prevents DNS-01 presentation entirely.

**Root cause**
- Cluster-shared cert-manager is not configured for Azure Workload Identity “ambient credentials” when solving DNS-01 for **namespaced Issuers**.
- cert-manager’s DNS-01 propagation checks rely on the cluster’s default recursive resolvers, which can lag or differ from public resolvers; this can stall Orders even when records are already publicly visible.

**Remediation approach (vendor-aligned, GitOps-idempotent)**
1. In the `cluster-cert-manager` Helm values (Azure only), enable:
   - `--issuer-ambient-credentials=true`
   - `--cluster-issuer-ambient-credentials=true`
2. In the same values, set cert-manager DNS-01 propagation recursion to known public resolvers:
   - `--dns01-recursive-nameservers-only=true`
   - `--dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53`
3. Verify `Certificate` resources become `Ready=True` and Argo apps converge (`argocd-tls`, `argocd-config`, `{env}-platform-cert-manager-config`).

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

---

## Implementation progress (602/603 alignment)

### 2025-12-26

- `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md` are now the source of truth for “Git-driven rollouts” (digest-pinned refs + `IfNotPresent`).
- `ameide` repo PR https://github.com/ameideio/ameide/pull/404 (in review) removes floating `:dev`/`:main` defaults from local tooling and updates operator chart docs to teach digest pinning (so the “restart pods to pick up :dev” class of issues can be retired).
