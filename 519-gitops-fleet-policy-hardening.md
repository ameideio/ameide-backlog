# 519 – GitOps Fleet Policy + Values Schema Hardening

**Status:** In progress (partially executed)  
**Owner:** Platform SRE / GitOps  
**Depends on:** 364 (Argo bootstrap + ApplicationSets), 375 (RollingSync waves), 446 (namespace isolation), 452 (Vault RBAC isolation), 456 (GHCR mirror policy), 462 (secrets origin classification), 505 (devcontainer service target architecture)

---

## 0) Executive summary

This backlog hardens the GitOps repository so Argo CD/ApplicationSet waves stay deterministic across cluster types, Helm value layering is collision-safe, and secrets/images/bootstraps follow operator-first (destination cluster) patterns.

Primary outcomes:
- Eliminate Helm values merge footguns (e.g. top-level `cluster:` collisions) by introducing a namespaced global contract (`global.ameide.*`) and schemas.
- Make “component presence” and “component enablement” explicit and wave-safe per cluster type (local vs azure).
- Remove non-deterministic Helm-generated runtime secrets; converge on Vault KV + External Secrets Operator for stable secrets.
- Stop patching vendor charts directly; introduce wrapper charts as the policy translation layer.

---

## 0.1) Backlog alignment (how this relates)

- **364 / 375:** keep the existing AppSet + RollingSync wave model; only refine how apps are selected per cluster type (without changing `rollout-phase` semantics).
- **446 / 452 / 462:** finish the transition to “destination cluster secrets” by removing Helm-generated stable secrets and formalizing Vault KV → ESO → Secret as the default.
- **456:** turn “mirror vs upstream” into a first-class fleet policy, then translate it in wrappers per chart image schema.
- **505:** normalize “optional workloads” (e.g., devcontainer) behind explicit capability/enablement rather than ad-hoc local-only overrides.

---

## 1) Current state snapshot (as of now)

**What is already aligned**
- **AppSet waves still function**: component definitions retain `rolloutPhase` and RollingSync steps continue to gate rollouts.
- **AppSet strict templating is enabled**: `goTemplateOptions: ["missingkey=error"]` is enforced so mis-templated apps fail fast.
- **A `global.ameide.*` contract exists**: cluster/env globals now publish a Helm-native contract and at least one owned chart enforces it via `values.schema.json`.
- **The fleet contract is available to cluster-scoped apps too**: both local and Azure overlays ensure cluster-scoped operators/config can consume the cluster-type globals file (e.g. `sources/values/cluster/{local,azure}/globals.yaml`) when needed.
- **Cluster-type component set scaffolding exists (local)**: local can now be driven from `environments/local/components/**` (curation/allowlist work still pending). Azure keeps using `_shared` as the canonical full set until it needs to diverge.
- **Destination-cluster secret management is in place** for Vault + SecretStore + GHCR pull secrets; cross-namespace Vault role now explicitly includes `ameide-system`.
- **Argo drift mitigations exist** where needed:
  - CNPG drift fix for `cluster.*` key collisions applied in templates.
  - Argo CD system `resource.customizations` covers Temporal CRs and CRDs to keep health deterministic (no “Unknown” during controller/CRD ordering).
- **Accepted exception: Argo CD kustomize load restrictions are relaxed**: `configs.cm.kustomize.buildOptions: "--load-restrictor LoadRestrictionsNone"` (in `sources/values/common/argocd.yaml`) is enabled to allow Argo CD self-management via `argocd/overlays/*`; track a follow-up to restructure kustomize bases so this can be removed.
- **Values layering footgun removed (namespace leak)**: `sources/values/cluster/globals.yaml` previously set `namespace: argocd`, which leaked into third-party charts that honor `.Values.namespace` and caused cluster-scoped operators to deploy into the wrong namespace. This is now removed; treat `namespace` as an application concern (Release namespace / Application destination), not a fleet-global default.
- **Envoy Gateway topology is standardized (cluster-shared control plane, vendor-aligned)**: controller runs in `argocd`; per-environment `EnvoyProxy` + `Gateway` resources live in `ameide-{env}`.
  - Per-env infra knobs are expressed via **namespaced** `Gateway.spec.infrastructure.parametersRef` → `EnvoyProxy` (no env-owned `GatewayClass` objects).
  - Static IP is requested via `Gateway.spec.addresses` (rendered from `azure.envoyPublicIpAddress` in env globals when `infrastructure.useStaticIP=true`), not via `EnvoyProxy.spec.provider.kubernetes.envoyService.loadBalancerIP` or Azure `azure-load-balancer-ipv4` annotations.
  - Internal/xDS TLS remains owned by Envoy Gateway `certgen` in `argocd`, while external hostname TLS remains cert-manager-managed in the environment namespace.
- **Policy (updated): canonical `*.{env}.ameide.io` must be correct via DNS, not CoreDNS rewrite**:
  - Managed clusters must treat `*.{env}.ameide.io` as a **canonical hostname contract** for both humans and pods.
  - Primary mechanism: **split-horizon DNS at the DNS layer** (public DNS + private DNS), with Terraform as the DNS authority (Single Writer).
  - CoreDNS rewrite remains permitted only as a narrowly-scoped, temporary exception (debug/local), not as the correctness mechanism for production plumbing.
  - Avoid `hostAliases` as an OIDC/DNS “fix”; it bypasses DNS and tends to force the public-IP path.
  - The canonical internal gateway target must be a normal Service with endpoints (avoid `ExternalName` alias chains in the critical path).
  - Reference: `backlog/716-platform-dns-split-horizon-envoy.md`.
- **Policy: AKS node pool taints must not block operator-managed Jobs**:
  - Do not taint every user pool `NoSchedule` by default; some vendor operators create setup Jobs without configurable tolerations/nodeSelectors (e.g., Temporal operator schema Jobs).
  - If taints are required for isolation, ensure at least one schedulable pool remains available for operator-managed setup Jobs, and document any exceptions explicitly.
- **Backstage stable session secret is operator-managed**: cookie signing secret is sourced from Vault KV and synced via External Secrets Operator (no Helm randomness, no Argo diff ignores).
- **Postgres credential Secrets are operator-managed**: CNPG user Secrets are now sourced from Vault KV and synced via External Secrets Operator (no Helm `rand*/lookup` loops, no secret payload diff ignores).
- **Schema guardrails are expanding**: additional internal charts now validate the `global.ameide.*` contract (start with Backstage + CNPG config charts).
- **Postgres password drift can be reconciled (local-only)**: a gated PostSync hook Job can reconcile existing DB role passwords from the synced Secrets using the CNPG superuser secret. This is intended as a migration/self-heal tool when switching password sources (e.g. Helm-generated → Vault KV → ESO).
- **A proper chart toggle exists for one vendor chart fork**: Langfuse worker can be disabled (currently implemented inside the vendored chart tree).
- **Temporal bootstrap is operator-native**: Temporal namespaces are managed via `TemporalNamespace` CRs and Temporal DB readiness is gated by an idempotent Argo hook Job (no “run this once” manual bootstrap).

### Incident addendum (2025-12-14): local k3d apiserver pressure → ComparisonError + leader-election loss

Symptoms:
- Argo apps intermittently show `Unknown` with `ComparisonError: ... context deadline exceeded` during health evaluation (not actual resource failure).
- Controllers (e.g. CNPG operator, redis-operator, Envoy Gateway) intermittently lose leader election because lease renewals time out (notably against the in-cluster `kubernetes.default.svc` ClusterIP path in k3d/k3s).

Policy-shaped remediation direction:
1. **Local Argo controller tuning is part of bootstrap policy**: apply `argocd-cmd-params-cm` settings via the Argo CD Helm install/upgrade inputs (bootstrap), not as ad-hoc Kustomize patches or manual `refresh=hard` loops.
2. **Operator k8s client defaults must be explicit** (where configurable): set QPS/burst and reduce concurrency for local clusters so leader election stays stable even under degraded apiserver latency.
3. **Reduce GitOps hook reliance for stable state**: eliminate Helm hook-based stable secret generation (e.g. GitLab shared-secrets) in favor of Vault KV → ESO → Secret so sync is deterministic and does not block on hook batches.
4. **Single-replica local controllers should not require leader election**: when we run `replicas: 1` for local, disable leader election (or expose a chart knob in a wrapper) so apiserver lease-write latency cannot crash the controller and cascade into `Progressing` Apps.
5. **Verify “policy knobs” are real in rendered manifests**: some vendor charts merge “base + user” config in a way that can prevent overrides (e.g. `provider.kubernetes.*`) from taking effect; treat this as a wrapper responsibility (or an explicit, tracked vendored patch) so local-hardening values are not silently ignored.

### Deferred TODOs (values layering contract)

These are intentionally deferred to later in this backlog (but now tracked explicitly).

- **Define “base chart vs env overlay” rules** (write down as repo policy):
  - Charts must remain portable: no environment identity (domains/hostnames), no cluster-type branching unless explicitly keyed off `global.ameide.*`.
  - `_shared` values represent Ameide baseline defaults; env values represent environment identity and sizing.
  - If a vendor image/layout differs (paths, plugins, CLI args), add a chart-level knob (not env-specific templating).
- **Expand schema guardrails** beyond one chart:
  - Enforce `global.ameide.*` in every Ameide-owned chart.
  - Add a lint/check that rejects environment hostnames in `_shared` and charts.
- **Centralize image policy**:
  - Add `global.ameide.images.policy.*` and standard translation patterns in wrappers.
  - Add a validation step for “local requires multi-arch (arm64 + amd64)” so we stop discovering arch issues at runtime.
  - Define a standard for hook/ops Jobs that need CLI tooling (curl/jq/kubectl): prefer a pinned multi-arch image by digest (no `apk/apt` downloads at runtime).
    - Avoid distroless-style CLI images when the Job/initContainer uses a shell script (e.g., `rancher/kubectl`, `registry.k8s.io/kubectl` do not ship `/bin/sh`).
    - Prefer a toolbox image that explicitly includes `/bin/sh` + required core utils (e.g., `alpine/k8s` for `kubectl`) and can be pinned by digest.
    - Values layering guardrail: if an overlay overrides `image.repository`, it must also override/unset `image.digest` (or it can accidentally produce an invalid `repository@digest` reference).
- **Reduce environment diffs**:
  - Collapse repeated hostname/domain wiring into `global.ameide.network.*` and service-level `httproute.hostname` only when truly needed.
  - Use consistent env/cluster value file ordering and document it once.

### Recent incident-driven fixes (captured for follow-up)

- **Local `arm64` image/arch gaps surfaced**:
  - Backstage Janus image was `amd64`-only → local moved to a multi-arch Backstage image + chart knobs for differing filesystem layouts.
  - Grafana mirror image crashed on `arm64` → local temporarily uses upstream multi-arch image; mirror pipeline needs multi-arch validation.
- **GHCR mirror pipeline produced single-arch tags**:
  - `docker pull`+`docker push` mirroring flattens upstream manifest lists into the runner’s architecture.
  - This can surface as `ImagePullBackOff` on local `arm64` (e.g., MinIO provisioning init image `bitnami-os-shell`).
  - Fix direction: copy with manifest preservation (`skopeo copy --all`) and validate `linux/amd64` + `linux/arm64` per mirrored tag.
- **GitOps “self-heal” gaps surfaced**:
  - Postgres role password drift now reconciles via a CronJob (migration/self-heal) rather than a one-shot hook.
  - Postgres URI generation now URL-encodes passwords to avoid runtime auth failures for certain passwords.
  - ClickHouse + Argo CD CRD diff/SSA panics required local-specific sync strategy (avoid SSA + allow Replace for the CHI app).
  - Keycloak realm GitOps can be blocked by non-deterministic hook Jobs: `client-patcher` must not rely on runtime-downloaded tooling (and must be multi-arch safe for local `arm64`).
  - Gateway API cross-namespace access is fragile when CRDs require empty-string fields: `ReferenceGrant.spec.to[].group` for core resources must render as `""` (quoted) to avoid YAML null + validation failures.
  - Helm `helm.sh/hook: test*` hooks are not a reliable execution mechanism under Argo CD; smoke Jobs should use Argo CD Resource Hooks (`argocd.argoproj.io/hook`, typically `PostSync`) plus hook delete policies.
  - Helm hook artifacts can persist and block convergence:
    - GitLab’s `upgrade-check` hook ConfigMap can remain in-cluster with `argocd.argoproj.io/hook-finalizer` even after the hook is disabled, preventing pruning and leaving applications `OutOfSync` while workloads are healthy.
    - Fix direction: disable Helm hooks (or wrap charts to remove hooks), and avoid depending on Argo hook finalizers for long-lived resources.
- Disabling Argo controller server-side diff for local stability increases client-side diff sensitivity to Kubernetes-defaulted fields (e.g. gRPC probe `service: ""`, `imagePullPolicy: IfNotPresent`), requiring targeted `ignoreDifferences` to keep GitOps noise-free without reintroducing apiserver pressure.
  - See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md` for the complementary image reference policy (digest pinning) that avoids relying on `imagePullPolicy` to force convergence.
  - Hook/bootstrap race windows can leave “Last Sync Failed” residue even when the system self-heals:
    - `foundation-vault-bootstrap` seeds keys on a schedule, while consuming apps (e.g., `platform-langfuse-bootstrap`) can sync immediately and temporarily observe missing Vault keys → ESO errors → Argo sync failures.
    - Smoke hooks (e.g., `helm-test-jobs`) can fail fast if backoff/timeouts are too aggressive for first-rollout readiness.
    - Fix direction: hook Jobs must wait on prerequisites; and charts must have correct `enabled` semantics so local can disable/enable behaviors intentionally.
  - Hook-only Applications (all resources annotated as hooks) can’t rely on Argo’s `OutOfSync` detection to trigger auto-sync reruns, which can leave stale `operationState.phase=Failed` even after fixes land in Git.
    - Fix direction: ensure smoke/bootstraps include at least one non-hook tracked resource (e.g. a ConfigMap with a deterministic checksum of test definitions) so Git changes cause OutOfSync → auto-sync → new successful operation state.
  - Local networking surfaced a bind-address portability issue: multiple services were observed listening on IPv6 wildcard (`:::PORT`) while the cluster is IPv4 service-addressed, leading to `connect: connection refused` from ClusterIP traffic.
    - Fix direction: default application listen addresses to explicit IPv4 (`0.0.0.0:PORT`) or ensure runtimes bind dual-stack correctly; avoid shipping local smoke hooks that assume per-service IPv4 reachability until this is enforced.
  - Smoke/ops Jobs need a standard, pinned toolbox image:
    - The `data-data-plane-ext-smoke` Temporal schema check failed due to `psql` runner image behavior; local mitigation may require a different image, but the fleet policy should mandate a pinned multi-arch toolbox (psql + curl + jq + grpcurl as needed) to avoid runtime package installs and surprise image behavior.

**What remains misaligned (gaps)**
- **Values schema collision risk remains repo-wide**: the worst offender (local top-level `cluster:`) is removed, but we still need to migrate remaining ambiguous root keys into the `global.ameide.*` contract and expand schema guardrails beyond a single chart.
- **cert-manager topology is not policy-shaped**: the repo currently encodes multiple cert-manager installs in some cluster types; vendor-supported topology is a single controller set per cluster, with multi-tenancy handled via RBAC/policy and Issuer/Certificate scoping (see 4.5).
- **Local disable semantics are inconsistent**:
  - Some components are disabled by “empty manifests” / pruning behavior instead of a first-class `enabled` contract.
  - Some components should not exist at all on local and should be excluded via the ApplicationSet generator rather than installed then pruned.
- **Non-deterministic runtime secrets still exist** in some charts and are currently masked via Argo diff ignores (migrate them to Vault KV → ESO → Secret, and remove ignores).
- **Vendor chart modifications exist in-tree** (e.g., Langfuse changes landed under `sources/charts/third_party/...`), which will complicate upstream upgrades.
- **Operator runtime stability is not policy-shaped**: some cluster-scoped controllers can CrashLoop under local k3d load due to leader-election lease renewal failures (e.g., Spotahome `redis-operator` exiting on `client rate limiter Wait ... context deadline exceeded`), leaving CR reconciliation incomplete and causing downstream smoke hooks to fail.
- **Local scheduling policy for controllers is not standardized**: critical controllers may need local-only node placement/priority (e.g., prefer the control-plane node + `system-cluster-critical`) to keep leader-election lease renewals reliable under constrained k3d resources.
- **Local apiserver access path is not standardized**: some Go controllers appear sensitive to the kube-proxy ClusterIP path for leader-election updates under k3d load; allow local-only overrides to target the `default/kubernetes` endpoint host/port directly (without changing production behavior).
- **Local fallback for operator-backed datastores is not standardized**: if a controller’s leader-election behavior is too fragile under local apiserver write latency, local should be able to switch the component to a “standalone” chart mode (no CRD/controller dependency) and update local consumers explicitly (e.g., Redis without Sentinel, using `redis-master` directly).
- **Third-party chart defaults are not validated**: published image tags referenced in vendored charts/values may not exist (e.g., Spotahome `redis-operator:v1.3.0` is not available on `quay.io`, while `v1.3.0-rc1` is), so image tags must be pinned to known-good/pulled-by-CI values.
- **Image policy is not centralized**: multi-arch requirements and “mirror vs upstream” decisions are handled ad-hoc per chart.
- **Bootstrap Jobs are not standardized**: job immutability, wait/retry patterns, and cleanup semantics vary per bootstrap chart.
- **Cluster-wide RBAC for per-environment smoke Jobs is collision-prone**: multiple env Applications using `helm-test-jobs` with `rbac.clusterWide: true` can collide on `ClusterRole/ClusterRoleBinding` names (derived from a shared Helm `releaseName`), breaking determinism in multi-env clusters.
  - Fix direction: make cluster-scoped RBAC names namespace-unique (or centralize cluster-wide roles as cluster-scoped components); also remove unnecessary `clusterWide` where namespace-scoped RBAC is sufficient.

---

## 2) Goals

1. **Collision-safe values layering**
   - Platform/fleet globals live under `global.ameide.*` (Helm-native propagation).
   - Avoid ambiguous root keys like `cluster`, `name`, `type`, `spec` in global values.
2. **Environment overlays are the source of truth**
   - `_shared/**` values must not contain environment-specific hostnames, issuers, or per-env defaults (e.g., `*.dev.ameide.io`).
   - Environment identity comes from env globals (`sources/values/env/*/globals.yaml`), not from shared app values.
   - Image tags choose code versions only; runtime configuration is injected via Kubernetes (`ConfigMap`/`Secret`), not baked into images.
3. **Wave-safe fleet targeting**
   - AppSets install the right component set per cluster type while preserving `rolloutPhase` gating:
     - Azure: uses `environments/_shared/components/**` as the canonical full set.
     - Local: uses `environments/local/components/**` as a curated allowlist/subset.
   - **No symlinked component definitions:** ApplicationSet git file generators must not rely on git symlinks (or any path that can be refactored out from under them). Broken symlinks cause repo-server file reads to fail and can Degrade `argocd-config` / stop application generation.
4. **First-class enablement**
   - Every workload we own or wrap supports `enabled: true|false`.
   - For truly unsupported components, prefer “do not generate the Application” over “install disabled placeholder resources”.
5. **Operator-first secrets**
   - Stable runtime secrets come from Vault KV → ESO → Kubernetes Secret (per 446/452/462).
   - Eliminate Helm-generated stable secrets (no `rand*` + `lookup` loops in runtime charts).
6. **Wrapper charts for third-party**
   - Changes required for fleet policy are implemented in wrappers, not in `sources/charts/third_party/**`.
7. **Central image policy**
   - A single contract for mirror preference + multi-arch requirement, with wrapper translation to each chart’s image schema.
8. **Standard bootstrap runner pattern**
   - Consistent wait/retry/idempotency + cleanup semantics for bootstrap Jobs that gate waves.
   - Avoid hardcoded cross-chart naming assumptions in bootstrap scripts (e.g., Service names): derive env-scoped names from the release namespace or express them via the `global.ameide.*` contract so per-env isolation (`fullnameOverride`) doesn’t silently break bootstrap.
   - Avoid hardcoding Kubernetes token audience defaults in Vault Kubernetes auth roles: `aud` varies by cluster type (e.g., AKS includes API server/OIDC issuer audiences), so audience binding must be optional or cluster-derived to keep Vault → ESO auth reproducible.
   - Avoid writing Vault Kubernetes auth `token_reviewer_jwt` from a Pod/Job-mounted serviceaccount token: projected SA tokens are short-lived/pod-bound in Kubernetes 1.21+, and pinning them can later break TokenReview and cascade into `SecretStore` 403s. For in-cluster Vault, prefer configuring kubernetes auth without `token_reviewer_jwt` and keep `disable_local_ca_jwt=false` so Vault re-reads the local token/CA.
   - Ensure secret-source overrides are actually applied: `secretPrefix: ""` must remain empty (no prefix) when fetching from AKV; avoid shell defaults that convert an explicit empty prefix back to `env-`.
   - Ensure Azure Workload Identity client IDs consumed by GitOps stay in sync with Terraform outputs:
     - Vault bootstrap (`vault_bootstrap_identity_client_id`) must be synced into `sources/values/env/*/foundation/foundation-vault-bootstrap.yaml` (and trigger a CronJob update) or AKV reads silently fail and the cluster deadlocks on private image pulls.
   - Prefer fail-fast over silent fixture fallback for bootstrap critical-path secrets (e.g., GHCR pull creds): if Key Vault lookup fails for a required secret, the bootstrap should error so the misconfiguration is obvious and doesn’t degrade into opaque `ImagePullBackOff` loops.
   - Standardize Vault Kubernetes auth token audiences: avoid setting `audience` on Vault roles unless all clients (ESO + Jobs) request the same TokenRequest audience; otherwise local can regress into `SecretStore` 403s (`permission denied` / `invalid audience`) and cascade into widespread Degraded Apps.

---

## 3) Non-goals

- Rewriting the entire repo layout in one shot (must remain incremental and reversible).
- Converting every third-party chart immediately (prioritize the ones that currently cause drift or local incompatibility).
- Introducing new secret providers beyond Vault + ESO.

---

## 4) Target architecture sketch

### 4.1 `global.ameide.*` fleet contract (Helm-native)

New canonical globals contract (examples; exact fields to be finalized in schema):

```yaml
global:
  ameide:
    cluster:
      type: local|azure
      name: k3d-ameide|aks-...
      arch: arm64|amd64
    capabilities:
      operators:
        agentOperator: true|false
      workloads:
        devcontainerService: true|false
        langfuseWorker: true|false
    images:
      policy:
        preferMirror: true|false
        requireMultiArch: true|false
        mirrorRegistry: ghcr.io/ameideio/mirror
```

### 4.2 Cluster-type component sets

- Azure hosted clusters use `environments/_shared/components/**/component.yaml` as the canonical full set.
- `environments/local/components/**/component.yaml` is the curated subset for local k3d; either omits unsupported components or points to wrappers with `enabled=false`.
- If/when Azure needs to diverge from `_shared`, introduce a real `environments/azure/components/**` tree (not a symlink mirror) and document the divergence.
- Cluster-scoped controllers must not assume environment-scoped service DNS (e.g., `transformation.ameide.svc`). If a controller needs env-specific endpoints, deploy it per environment namespace or make endpoint discovery namespace-aware.

### 4.5 cert-manager topology (single install per cluster)

Policy goal: a **single cert-manager controller set** per cluster (vendor-supported), with isolation achieved through **RBAC + admission policy + namespaced Issuers** rather than by installing multiple cert-managers.

Recommended default (shared cluster):
- **One** cert-manager install (controller + webhook + cainjector) in a dedicated namespace (prefer `cert-manager`).
- CRDs remain managed by the CRD Application (`cluster-crds-cert-manager`), not by Helm `crds.enabled=true`.
- Use **namespaced `Issuer`** resources per environment namespace (`ameide-dev`, `ameide-staging`, `ameide-prod`, `ameide-local`) for env-scoped certificate authority choices.
- Use `ClusterIssuer` only when explicitly permitted by policy (e.g., cluster-wide ingress/gateway TLS managed by platform).
- Enforce multi-tenancy with:
  - RBAC: only platform admins can create/modify `ClusterIssuer` and cluster-scoped CertificateRequests; app namespaces can only create `Issuer` + `Certificate` in their namespace.
  - Admission policy: disallow privileged issuer references (e.g., prevent app namespaces from referencing a platform `ClusterIssuer` unless allowed).

Operator webhook TLS + CA injection (e.g., Temporal operator in `ameide-system`):
- Keep operator workloads cluster-scoped in `ameide-system`, but use the single cert-manager to issue webhook server certs and inject CA bundles.
- Prefer explicit `Certificate` resources in the operator namespace and annotate webhook configurations with `cert-manager.io/inject-ca-from: ameide-system/<certificateName>` so CA bundles are reconciled deterministically by cainjector.

Azure DNS-01 (Workload Identity) requirements:
- If we solve ACME DNS-01 via Azure DNS using Azure Workload Identity, cert-manager must explicitly allow “ambient credentials” for both cluster and namespaced issuers:
  - `--cluster-issuer-ambient-credentials=true`
  - `--issuer-ambient-credentials=true`
- Pin DNS-01 propagation checks to known public recursive resolvers to avoid “record exists publicly but propagation check still fails”:
  - `--dns01-recursive-nameservers-only=true`
  - `--dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53`

Local-only deviation (developer cluster):
- If local clusters are extremely resource constrained, we may choose to restrict cert-manager watch scope or capabilities, but we should not model this as “multiple cert-managers”; instead capability-gate non-essential certificate consumers (or use per-component certgen jobs where vendor-supported).

Implementation note (Option A execution):
- Replace all per-environment and per-namespace cert-manager installs (`foundation-cert-manager`, `argocd-cert-manager`, `operators-cert-manager`) with a single cluster-scoped `cluster-cert-manager` Application targeting the `cert-manager` namespace.
- Keep cert-manager CRDs in `cluster-crds-cert-manager`.
- Retain `platform-cert-manager-config` as the place where Issuers/Certificates are declared (namespaced) for applications/operators.

### 4.6 Local stability guardrails (controllers + runtime determinism)

Local k3d clusters are capacity-constrained and can exhibit apiserver write latency that breaks leader-election renewals and long-running health checks. Local should stay **reproducible and self-healing**, but it can legitimately diverge in *operational toggles* that make single-replica controllers stable.

**Policy (local-only, capability-shaped):**
- **Treat local clusters as disposable infrastructure (Terraform-first):**
  - If the local apiserver becomes unreliable (timeouts / stale Argo status / lease renewals failing), the supported remediation is to **recreate the local cluster via Terraform** (`infra/scripts/tf-local.sh destroy/apply`), not to manually delete namespaces/resources.
  - This is consistent with k3s vendor guidance that embedded SQLite is the default and is best suited to “simple/short-lived” clusters; heavy churn can surface datastore latency that cascades into controller health flapping.
- **Local secret seeding is part of reproducible bootstrap (no manual prerequisites):**
  - Local bootstrap must ensure `vault-bootstrap-local-secrets` exists (seeded from `.env`) before `foundation-vault-bootstrap` runs; otherwise Vault stays sealed and every `ExternalSecret` remains `SecretSyncedError`.
- **Disable leader election when `replicas=1`** for controllers/operators that exit on `leader election lost` (e.g., Strimzi, NiFiKop, Envoy Gateway). Leader election becomes a failure mode under local latency and provides no benefit at single replica.
- **Argo CD must be locally stable (no self-inflicted CrashLoops):**
  - For local clusters, increase Argo CD probe budgets (especially `argocd-repo-server` liveness/readiness timeouts) so transient stalls don’t cause kubelet restarts that cascade into stale Application status and rendering failures.
- **No runtime downloads / no dev servers in Argo baseline**:
  - Argo-managed “baseline” workloads must start deterministically from the published image (no Corepack/pnpm downloads, no runtime codegen, no writes to the image filesystem).
  - Baseline images must be **runtime-capable** (e.g., Next.js images must include a production `.next` build if we run `next start`).
  - Baseline images must be **multi-arch** when the cluster type requires it (local k3d on arm64). Treat `amd64`-only `main` tags as a fleet policy violation (track under 456).
  - The inner loop (dev servers, live reload) belongs in Tilt/Telepresence “*-tilt” releases, not the Argo baseline.
- **Charts must honor auth toggles**: if a chart exposes `auth.enabled`, templates must not unconditionally enable auth (especially in local fallbacks like standalone Redis).
- **Gateway addressability must be a declared capability**:
  - If a local environment expects `Service.type=LoadBalancer` (e.g., Envoy Gateway data-plane Services), the cluster must include a load-balancer implementation (k3s `servicelb` or MetalLB) so Gateways get addresses deterministically.
  - Do not “green” Gateways by relaxing health checks; treat missing Gateway addresses as a real capability gap that must be solved in GitOps.

**Tracking note:** if any local-only knob is required (leader election disable, reduced concurrency, reduced replicas), record it explicitly as a local capability decision and keep it out of shared defaults unless it is safe for real clusters.

**Terraform/DNS/externals impact (local):**
- Terraform owns the k3d cluster lifecycle (`infra/terraform/local` + `infra/scripts/tf-local.sh`). Avoid ad-hoc `k3d cluster create/delete` outside Terraform unless state is already lost (the wrapper detects and repairs this case).
- Any Terraform `local-exec` that runs `kubectl`/`helm` must pin the intended kube context (`kubectl --context ...` / `helm --kube-context ...`); never rely on ambient `kubectl config current-context` (prevents accidental writes to AKS while operating “local”).
- Any Terraform heredoc shell snippet must use Terraform-safe `$` escaping correctly: only escape Terraform interpolation (`${...}`) as `$${...}`. Do not use `$$` for shell variables/arithmetic (`$$((...))`, `$$*`, `$$var`), because `$$` expands to the shell PID and can break scripts at runtime.
- Recreating the cluster changes ephemeral container IDs/IPs; anything that pins those (e.g., `*.local.ameide.io` host mappings, DNS scripts, port-forwards) must be derived from the current k3d/kube context (or rerun) after recreate.

**Terraform/DNS/externals impact (azure):**
- Keep AKS infrastructure provisioning **subscription-scoped and least-privilege**. Tenant-wide Entra ID directory objects (e.g., creating groups) should live in a separate “directory bootstrap” stack run with appropriately privileged credentials.
- Treat Azure Key Vault as a special-case lifecycle: purge protection + soft delete means “destroy then recreate” is effectively “delete then recover”. Keep recovery behavior inside Terraform (`azurerm` Key Vault features) and avoid out-of-band `az keyvault recover` that creates unmanaged existing resources and breaks idempotency.
- When Terraform seeds Key Vault secrets, the Terraform runner identity must have **data-plane RBAC** (`Key Vault Secrets Officer` or stricter equivalent) on the vault; otherwise `azurerm_key_vault_secret` will fail during existence checks with `ForbiddenByRbac`.
- GitOps consumers (Envoy Gateway addresses, DNS hostnames, etc.) must source Azure runtime facts from Terraform outputs (`artifacts/terraform-outputs/azure.json` → `infra/scripts/sync-globals.sh`) rather than hardcoding IPs/hostnames in Helm values.
- ArgoCD bootstrap must not depend on private registry pull secrets (chicken-and-egg): ensure bootstrap correctly applies `sources/values/env/<env>/foundation/foundation-argocd.yaml` so Redis and other bootstrap images are public until `ghcr-pull` is created by External Secrets.
- Any cluster-shared namespace that runs cluster-scoped controllers (e.g., `ameide-system`) must also receive `ghcr-pull` deterministically; do not assume “env namespaces only” or operators will `ImagePullBackOff` while env workloads appear healthy.
- Cluster-scoped controllers on hosted clusters must only use **multi-arch images** (at least `linux/amd64`) and must be deployed by **digest** in GitOps. If producer pipelines rely on `:dev` as the “discovery tag” for the local/dev bump workflow, that tag must be a multi-arch manifest list so digest resolution is valid for every cluster type.
- Any “bootstrap critical path” Job (e.g., Vault initialization/bootstrap) must also use public images and avoid `ghcr-pull` dependencies; otherwise Vault can’t initialize, SecretStores remain NotReady, and the entire cluster deadlocks on image pulls.

### 4.3 Wrappers for vendor charts

Example wrapper conventions:
- `sources/charts/platform/langfuse` depends on `sources/charts/third_party/langfuse/...`
- Wrapper implements:
  - `enabled` toggles
  - policy translation from `global.ameide.images.policy.*`
  - local capability gating
  - any extra manifests needed for Ameide conventions

### 4.4 Diff customization split (system vs app)

- **System-level (global, kind-based):** Argo CD `resource.customizations` for known controller/webhook mutations we never want to fight repo-wide.
- **App-level (component-specific):** `ignoreDifferences` only where a single component needs it.
- When using ignore rules that should apply during sync (not only diff), prefer enabling `RespectIgnoreDifferences=true` on those Applications (documented and intentional).

---

## 5) Delivery plan (incremental phases)

### Phase A — Fleet contract + compatibility bridge

1. Add `global.ameide.*` to `sources/values/base/globals.yaml` and cluster/env overlays.
2. Add a compatibility bridge for existing charts that read root keys:
   - keep current keys temporarily (`cluster.*`, etc.) but treat them as deprecated.
3. Add `values.schema.json` to at least one internal chart to validate `global.ameide.*` presence and structure.

**Exit criteria**
- `global.ameide.cluster.*` exists across environments.
- One chart fails fast if contract is missing/malformed.

### Phase B — ApplicationSet generator split by clusterType

1. Create `environments/local/components/**` (start minimal).
2. Patch AppSet overlays so the local overlay reads `environments/local/components/**`.
3. Azure remains on `environments/_shared/components/**` by default; only introduce `environments/azure/components/**` if Azure needs to diverge.
3. Enforce strict AppSet templating (fail fast):
   - `goTemplateOptions: ["missingkey=error"]`
4. Preserve wave labels (`rolloutPhase`) and validate RollingSync still progresses.

**Exit criteria**
- Local installs only the intended set of apps (no “unsupported-but-present” apps).
- `argocd app wait --selector rollout-phase=... --health` progresses without stalling on excluded apps.

### Phase C — First-class enablement pattern

1. For all Ameide-owned charts: add `enabled` gates.
2. For vendor charts: add enablement in wrappers (not vendored chart trees).
3. Establish a repo convention:
   - omit from component set when unsupported
   - use `enabled=false` only when we want the app “visible but off” (chart should render a deterministic marker resource when disabled so Argo never sees an empty desired state)

**Exit criteria**
- Local disable decisions are expressed either by omission or `enabled=false`, never by “empty manifests + prune”.

### Phase C.1 — Probe budgets and resource baselines (local stability)

1. For local clusters, ensure critical controllers and observability components do not use overly strict HTTP probe defaults (e.g., `timeoutSeconds: 1`) that cause restarts/flapping under transient apiserver stalls.
2. Prefer vendor-supported values knobs for `timeoutSeconds`/`failureThreshold`; when a vendored chart does not expose them, apply a minimal patch that adds the knobs with defaults matching upstream behavior.
3. Set explicit resource requests for local observability components so scheduling/cpu starvation doesn’t translate into false probe failures.
4. Reserve the local k3d control-plane node for control-plane work (taint `NoSchedule` when agents exist) so data-plane pods don’t starve the apiserver and trigger probe/leader-election churn.
5. When an operator CRD only exposes partial probe settings (e.g., Keycloak’s top-level probe fields), use the operator-supported pod template override mechanism (e.g., `spec.unsupported.podTemplate`) to set full probe objects deterministically.
6. Avoid hardcoded liveness semantics that depend on external systems (e.g., apiserver) for local: if vendor charts hardcode probe paths, patch vendored charts minimally to expose the path/enablement knobs with upstream-matching defaults.

**Exit criteria**
- Local bootstrap completes with no recurring probe-driven restarts for core operators (e.g., Strimzi) and observability (Loki/Alloy).

### Phase C.2 — Repo access is deterministic (no broken credentials)

1. Treat ArgoCD Git repo access as a first-class bootstrap invariant: repo-server must be able to fetch new commits deterministically.
2. Do not force credentials for public repos (omit `username/password`); for private repos, source tokens from a secure store and avoid persisting raw tokens in Terraform state.
3. Prefer “anonymous-first, token-if-required” logic in bootstrap tooling so stale tokens don’t break sync.

### Phase D — Secrets: remove Helm-generated stable secrets

1. Move Backstage session secret generation to Vault KV and sync via ESO.
2. Remove Helm `rand*/lookup` stable-secret generation from app charts (or disable those templates by default).
3. Keep `ignoreDifferences` only for truly controller-managed fields, not for secrets we can make deterministic.
4. Document when ESO generators are allowed (rotation-only), and prohibit their use for “must remain stable” secrets.
5. Prohibit committed placeholder secret values (`CHANGEME`, `REPLACE_ME`) in shared values; charts must either consume secrets from ESO/Vault or fail fast when enabled.

**Exit criteria**
- No runtime chart uses Helm randomness to produce a stable secret value.
- Backstage app is `Synced/Healthy` without secret payload ignores.

### Phase E — Image policy consolidation

1. Define `global.ameide.images.policy.*` as the only place to decide mirror + multi-arch requirements.
2. Implement translation in wrappers to each vendor chart’s image schema.
3. Add a lightweight lint check (script) that flags images that violate local multi-arch requirement (optional).

**Exit criteria**
- Local no longer needs ad-hoc “use upstream multi-arch image” overrides per chart.

### Phase F — Bootstrap Jobs standardization

1. Create a reusable “bootstrap runner” chart (or a shared template library) that supports:
   - dependency readiness wait
   - bounded retry/backoff
   - idempotent operations
   - TTL cleanup for completed Jobs
2. Migrate Temporal namespace bootstrap to the standard pattern.
3. Prohibit runtime dependency installs in Jobs/CronJobs (`apk add`, `curl | bash`, downloading CLIs): images must be self-contained so GitOps is reproducible offline and under transient network stalls.

**Exit criteria**
- Bootstrap jobs do not fail due to race-to-dependency readiness.
- Bootstrap jobs do not create immutable-template drift loops during Argo sync.

### Phase G — Diff customization and SSA policy

1. Move generic ignore rules into Argo CD system config (`argocd-cm` `resource.customizations`).
2. Keep component-level `ignoreDifferences` only for workload-specific, justified cases.
3. For remaining SSA noise (e.g., CNPG-managed fields), prefer:
   - removing the root cause (values collisions / non-determinism), otherwise
   - a scoped ignore rule + `RespectIgnoreDifferences=true` when appropriate.

**Exit criteria**
- No secret payload ignores remain for stable secrets.
- Any remaining ignore rules are scoped, documented, and consistent across environments.

---

## 6) Acceptance criteria (definition of done)

- Values collision class is eliminated: no chart receives unexpected `spec` keys due to global `cluster.*` merges.
- Local and Azure are cleanly separated by component descriptors while preserving waves.
- No vendored third-party chart trees contain Ameide-specific logic (all policy is in wrappers).
- Stable runtime secrets are sourced from Vault/ESO, not Helm randomness.
- Generic diff ignore rules live in Argo system config; component-level ignores are rare and documented.
- `argocd app list` shows no `Degraded|OutOfSync|Progressing` after a full bootstrap on local and one hosted environment.

---

## 7) Risks & mitigations

- **Large migration surface area** → execute in phases; keep compatibility bridge until wrappers are adopted.
- **Wave stalling if components are miscategorized** → start with small local component set; validate each wave before expanding.
- **Vendor upgrades become painful if we patch vendor charts** → enforce wrapper-only rule; add CI lint to block edits under `sources/charts/third_party/**` except version bumps.

---

## 8) Tracking checklist

- [ ] Phase A: `global.ameide.*` contract + schema
  - [x] AppSet injects `global.ameide.*` for env-scoped apps (missingkey=error)
  - [x] Schema guardrails added to initial owned charts (Backstage + CNPG config + Temporal)
  - [ ] Expand schema guardrails repo-wide; migrate remaining ambiguous root keys
- [ ] Phase B: clusterType component directories + AppSet wiring
  - [x] Local overlay reads `environments/local/components/**`
  - [x] Azure remains canonical on `environments/_shared/components/**`
  - [ ] Curate `environments/local/components/**` into a true local allowlist (omit unsupported apps)
- [ ] Phase C: enablement contract across owned/wrapped charts
- [ ] Phase D: stable secrets moved to Vault/ESO; remove Helm rand secrets
- [ ] Phase E: central image policy + wrapper translation
- [ ] Phase F: standardized bootstrap runner pattern
- [ ] Phase G: system-level diff customizations + SSA policy
  - [x] Scoped app-level drift mitigations added where required (e.g., GitLab defaulted fields)
  - [ ] Move generic drift rules into Argo system config where appropriate; minimize `ignoreDifferences`

---

## Implementation progress (image policy)

### 2026-01-11

- `ameide-gitops`: ARC runner image publishing + pinning updated to multi-arch (`linux/amd64,linux/arm64`) and pinned by manifest digest; BuildKit binfmt installs `amd64,arm64` to support cross-arch builds.

### 2025-12-26

- `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`: clarified “local is automated, not an exception” and expanded the cross-repo checklist.
- `ameide` repo PR https://github.com/ameideio/ameide/pull/404 (in review) removes floating `:dev`/`:main` defaults from producer-side tooling (scripts/CLI/scaffold) and adds digest/ref support to the operators Helm chart (pre-req for fleet-wide enforcement).
