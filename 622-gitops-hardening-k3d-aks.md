---
title: "622 – GitOps Hardening: k3d (local) + AKS (dev) parity"
status: active
owners:
  - platform-sre
  - gitops
created: 2026-01-10
updated: 2026-01-11
---

# 622 – GitOps Hardening: k3d (local) + AKS (dev) parity

## Problem statement

We must be able to iterate on **AKS dev** without breaking **local k3d**, and vice-versa.
Every “fix” must be evaluated across **both targets** and must result in a configuration that is:

- **Vendor-aligned** (Kubernetes + Azure AKS + OCI image supply chain).
- **Deterministic** (no “works if you click it twice”; no silent fallbacks).
- **Scoped** (cluster-scoped resources are owned once; env-scoped resources don’t fight each other).

This backlog captures the current failure modes we observed and defines a **single, convergent approach** for both local and dev.

## Scope

### In scope

- GitOps configuration parity for **local k3d** and **AKS dev**.
- Cluster bootstrap and ArgoCD reconciliation hardening (render validity, image policy, scheduling).
- Operator + data-plane correctness for both targets (Temporal operator/webhooks, Strimzi/Kafka scheduling, Gateway API routes).
- CI guardrails that prevent “fix dev, break local”.

### Out of scope

- Re-architecting the entire app portfolio; we focus on minimum changes that restore full health and prevent regressions.
- Terraform/provider changes unrelated to GitOps parity (tracked under `backlog/444-terraform-v2.md`).

## Target state (definition of done)

1. **Both targets converge** from a clean bootstrap:
   - local: `k3d` + local terraform + GitOps bootstrap results in “baseline apps” green.
   - dev: GitHub CI apply/verify results in “baseline apps” green.
2. **No Argo ComparisonError** for applications that are declared “installed”.
3. **No SharedResourceWarning** (cluster-scoped resource ownership is unambiguous).
4. **No hostPort collisions** in multi-env AKS (dev/staging/prod share the same worker nodes).
5. **Images are multi-arch** (linux/amd64 + linux/arm64) wherever local needs arm64 and AKS is amd64.
6. CI includes a **render + conformance gate** for both local and dev overlays.

## Status (2026-01-11)

- The main parity blockers described here have been implemented in `ameide-gitops` (ARC runner parity + GitHub-variable routing, multi-arch runner/build support, and Docker Hub pull-rate limiting mitigations via mirroring).
- Remaining work is mostly “guardrail + hygiene”: keep expanding CI render/conformance coverage, and keep separating “local-only stability knobs” from shared defaults.

### Status note: platform-coder (AKS dev)

While wiring `platform-coder` (Coder CE for AmeideDevContainerService), we hit three “parity-class” issues that map directly to this backlog’s goals (deterministic convergence, no manual clicks):

1) **Vault/Keycloak client secret pipeline**
   - Symptom: Coder pod failed because `keycloak-realm-oidc-clients` lacked `coder-client-secret`.
   - Root cause: Vault policy for the Keycloak client patcher did not allow `secret/data/coder-*`.
   - Fix: allow `secret/data/coder-*` writes so the patcher can persist/extract the Coder client secret.

2) **Coder first-run `/setup`**
   - Symptom: `coder.dev.ameide.io` served `/setup` despite Keycloak OIDC being configured.
   - Root cause: Coder requires a “first user” record; OIDC alone does not create it.
   - Fix: dev-only bootstrap Job creates an initial admin user so humans go straight to Keycloak SSO.

3) **Postgres role password drift**
   - Symptom: Coder crashed with `pq: password authentication failed for user "coder"`.
   - Root cause: Vault/ESO password existed, but CNPG role password did not reliably converge without a reconcile mechanism (no superuser secret available).
   - Fix: dev-only password reconcile CronJob (`kubectl exec` into CNPG primary) keeps DB roles aligned with Vault/ESO secrets.

### Status note: platform-camunda8 (AKS dev)

While enabling the Camunda 8 baseline (`orchestration` + `connectors`) we hit “parity-class” issues that map to this backlog’s goals (deterministic convergence; no manual clicks):

1) **Helm schema type mismatch in env overlays**
   - Symptom: values like `clusterSize: 1` did not apply as intended.
   - Root cause: upstream chart schema expects some topology parameters as **strings**, so number values can be rejected at render/apply time.
   - Fix direction: keep topology values as quoted strings and add a render gate that fails on schema errors.

2) **StatefulSet immutability on persistence changes**
   - Symptom: Argo sync failed with “Forbidden: updates to statefulset spec … are forbidden”.
   - Root cause: changing persistence/storage shape often changes immutable StatefulSet fields.
   - Fix direction: treat storage shape changes as replace/migration tasks; only change replicas / pod template bits via upgrades.

3) **OIDC templating scope / empty-host URLs**
   - Symptom: runtime errors like `https://auth./realms/ameide` leading to crashloops and `500`s.
   - Root cause: `tpl` rendering scope must reference values that exist in the subchart scope.
   - Fix direction: derive issuer/redirects from a canonical, globally-present public base and add a CI “no empty-host URLs” guardrail.

4) **Keycloak → Vault → ESO secret materialization**
   - Symptom: placeholder client secrets lead to token `401` and application failures.
   - Root cause: missing Vault policy paths / hook rerun semantics.
   - Fix direction: treat as a platform secret pipeline issue; ensure Keycloak client-patcher can write required Vault paths and ExternalSecrets refresh behavior is deterministic.

## Current findings (AKS dev) and how they relate to local

### A) Docker Hub pull rate limiting blocked reconciliation (AKS) and was masked locally

**Observed in AKS dev:** many pods failed with `429 Too Many Requests` from `docker.io`, leaving multiple Argo apps `Progressing/Degraded`.

**Why local didn’t show it consistently:** local k3d defaults to `imagePullPolicy: IfNotPresent` (see `sources/values/cluster/local/globals.yaml`) and tends to have warmer caches; AKS nodes were repeatedly pulling from Docker Hub.

**Remediation (parity-safe, vendor-aligned):**
- Make `docker.io` a “do not depend on at runtime” registry for shared workloads.
- Use:
  - public mirrors for “docker library” images (see `backlog/456-ghcr-mirror.md` + shared values),
  - a GHCR mirror for third-party images that don’t have a stable non-DockerHub source,
  - CI-driven mirroring so the cluster never needs Docker Hub (see `.github/workflows/mirror-images.yaml`).

**Hard rule (avoid band-aids):** fix image sources in `sources/values/_shared/**` and `sources/values/env/**` (the ApplicationSet layering inputs). Do not “fix” in `sources/charts/shared-values/**` unless a chart explicitly consumes it.

### B) Multi-arch image integrity (amd64 + arm64) is a hard requirement

**Observed in AKS dev:** images pinned to a single-arch digest caused `no match for platform in manifest` on amd64 nodes (or `exec format error` when the wrong arch image was used).

**Why this is a parity issue:** k3d (developer laptops) is often `linux/arm64`, AKS is `linux/amd64`. Any shared image must be a **manifest list** digest, not a single-arch digest.

**Remediation:**
- For smoke jobs: replace the broken `grpcurl-runner` digest with a multi-arch upstream (`fullstorydev/grpcurl`) mirrored to GHCR and pinned by digest.
- For app images: only pin digests that are known multi-arch (see `backlog/603-image-pull-policy.md`).
- For ARC: ensure the runner image is published as a multi-arch manifest list and GitOps pins the manifest digest (not an arch-specific digest).

### C) Cluster-scoped collisions across envs (SharedResourceWarning)

**Observed:** Kubernetes Dashboard installs per env can collide on cluster-scoped objects (`metrics-scraper` ClusterRole/Binding), producing `SharedResourceWarning` and preventing clean convergence.

**Remediation:**
- Keep the Dashboard env-scoped, but scope any cluster-scoped names per env.
- Concrete implementation: set `metricsScraper.role` uniquely per env in:
  - `sources/values/env/dev/platform/kubernetes-dashboard.yaml`
  - `sources/values/env/staging/platform/kubernetes-dashboard.yaml`
  - `sources/values/env/production/platform/kubernetes-dashboard.yaml`
  - `sources/values/env/local/platform/kubernetes-dashboard.yaml`
  while keeping shared defaults in `sources/values/_shared/platform/kubernetes-dashboard.yaml`.

### D) Secrets and pull credentials must be namespace-complete (Vault + ExternalSecrets)

**Observed:** `clickhouse-system` had `ExternalSecret` resources that referenced `SecretStore/ameide-vault`, but the SecretStore didn’t exist in that namespace. Result: `ghcr-pull` never materialized there and private image pulls failed (401).

**Related backlogs:**
- `backlog/446-namespace-isolation.md` (SecretStore is namespace-scoped)
- `backlog/452-vault-rbac-isolation.md` (cross-namespace consumers must be explicitly declared)
- `backlog/419-clickhouse-argocd-resiliency.md` (operator namespace + resiliency)

**Remediation:**
- Extend `extraSecretStores` to include `clickhouse-system` in:
  - `sources/values/env/production/foundation/foundation-vault-secret-store.yaml`
  - `sources/values/env/local/foundation/foundation-vault-secret-store.yaml`

### E) Local must not require the cloud “runtime-facts repo” to render

**Observed (local k3d):** Argo CD repo-server ComparisonErrors when it could not fetch `https://github.com/ameideio/ameide-runtime-facts.git`.

**Why this breaks parity:** local bootstrap should be self-contained (devs should not need access to a separate CI-owned repo to get a working local cluster).

**Remediation (no drift):**
- Patch the local overlay to source “runtime facts” from this repo for local only:
  - `argocd/overlays/local/kustomization.yaml`
- Provide an empty placeholder contract so Helm valueFiles always resolve:
  - `runtime-facts/cluster/local/globals.yaml`
  - `runtime-facts/env/local/globals.yaml`

### F) Remaining “full health” work (must be solved without breaking local)

These are still present in AKS and must be addressed with changes that also make sense for local:

- Kafka `Progressing`: remove env-specific pool/affinity assumptions and converge on a provider-neutral node profile (see `backlog/421-argocd-strimzi-kafkanodepool-health.md` + `backlog/444-terraform-v2.md`).
- Prometheus `Progressing`: avoid per-env hostPort DaemonSets in multi-env clusters; deploy node-exporter once at cluster scope (see `backlog/447-third-party-chart-tolerations.md`).
- `dev-process-transformation-v0-ingress` CrashLoopBackOff: treat as an application-level failure (needs logs + config verification) but must be reproducible locally.
- Gateways `Degraded` in staging/prod due to missing backend Service for GRPCRoutes: decide whether the routes are env-conditional or the backend must exist in those envs.

### G) Guardrails (to prevent recurrence)

- CI must verify:
  - render validity for both local and azure overlays (values layering),
  - no runtime-critical pulls from Docker Hub for shared workloads,
  - multi-arch digest validity for shared images,
  - namespace completeness for SecretStore + ExternalSecrets where cross-namespace secrets are required.

## Cross-target “boundaries” (to prevent spaghetti)

## Allowed target-specific deltas (explicit and intentional)

These deltas are expected and must remain **explicitly scoped to env overlays** (not scattered conditionals):

- **Local** keeps `k3d` + local Terraform for cluster lifecycle and fast iteration.
- **Local** does not use Azure DNS / Let’s Encrypt DNS01; Azure Workload Identity helpers are disabled where appropriate.
- **Local** must remain compatible with **linux/arm64** (common on Apple Silicon), while **AKS dev** is **linux/amd64**.
- **Local** may need “relaxed scheduling” tolerations to allow stateful pods on control-plane nodes.

Everything else should converge (same charts, same health semantics, same required values, same schema validation).

### 1) Cluster-scoped vs env-scoped ownership

**Cluster-scoped (one owner per cluster):**
- CRDs and operators (cert-manager, external-secrets, strimzi, temporal-operator, cnpg, etc.)
- Gateway / shared ingress dataplane and its “global” routes
- Any hostPort DaemonSets (node-exporter)

**Env-scoped (one owner per env namespace):**
- App workloads and env-specific routes
- Env-specific secrets materialization and seed jobs

Rule: env-scoped apps must not create cluster-scoped objects with stable names unless they include env scoping in the name.

### 2) Image supply chain contract (local + dev)

- Any image used by both targets must be published as a **multi-arch manifest list** (amd64 + arm64).
- GitOps pins **manifest list digests** (not single-arch digests) so both runtimes pull the correct platform.
- Local may route images via a **local registry** (k3d) or a **GHCR mirror**, but the pinned reference must remain deterministic (digest pinned; no floating tags).
- CI must detect:
  - missing `:main` channel tags for local/dev automation
  - missing multi-arch manifests
  - missing required `image.ref` values for installed apps

## Implementation notes (2026-01)

### ARC runner parity + routing

- ARC runner sets exist in both targets:
  - Local: `arc-local`
  - AKS: `arc-aks-v2`
- Workflow routing is GitHub-driven only via `vars.AMEIDE_RUNS_ON` (no workflow defaults).
- Runner registration is org-scoped (`githubConfigUrl: https://github.com/ameideio`), so multiple repos can use the same runner substrate.
  - Failure mode: if runner groups restrict repo access, jobs may remain queued until the repo is allowed (or a dedicated runner set is deployed).

### BuildKit cross-arch support

- `buildkit/binfmt` installs emulation for `amd64,arm64` so a runner on either arch can build both platforms when needed.

### Telepresence traffic-manager scoping (local + AKS)

We hit a convergence blocker where the per-environment Telepresence traffic-manager could:
- CrashLoop under namespaced RBAC because it tried to `list namespaces` at cluster scope, and/or
- Inject sidecars pointing at the wrong environment’s traffic-manager (cross-env `managerHost` drift).

Fix direction (vendor-aligned and GitOps-owned):
- Run traffic-manager with **namespaced RBAC** and scope it to exactly one environment namespace.
- Ensure the rendered `ConfigMap/traffic-manager` sets `namespace-selector.yaml` to `operator: In` with `values: [ameide-<env>]` so the manager does not need cluster-wide namespace listing.

Related tracking: see `backlog/492-telepresence-reliability.md`.

### Local Redis operator instability (k3d) vs Redis posture (AKS)

We observed `cluster-redis-operator` (Spotahome `redis-operator`) stuck `Synced/Progressing` in local due to the operator **CrashLooping** around leader-election renewal under k3d/k3s (tight election deadlines and a single-node-ish control-plane can make this flaky).

This was a *parity blocker* because local cluster health could not be “green by default” while AKS dev remained healthy.

Fix direction (no band-aids, GitOps-owned):

- **Local**: run the `data-redis-failover` chart in `mode: standalone` (a plain Redis StatefulSet) and disable the cluster-scoped `redis-operator` installation in the local ArgoCD overlay.
- **Local**: apps connect directly to `redis-master:6379` (no Sentinel dependency).
- **AKS dev**: can continue to use `mode: failover` + Sentinel where HA is desired.

This keeps both targets convergent and green while making the local posture intentionally deterministic (single-master, minimal control-plane dependencies).

### Local ClickHouse node placement (local-path PV binding)

Local k3d’s `local-path` storage binds PVs to the node where the first consumer pod schedules. If ClickHouse initially schedules to the control-plane, later “avoid control-plane” hardening can strand the PVC on the wrong node.

Fix direction:
- Make local ClickHouse schedule to worker nodes deterministically (local-only affinity), and treat any required PVC deletion as an **operational local-only reset** (data loss acceptable locally).

Related tracking: see `backlog/419-clickhouse-argocd-resiliency.md`.

### Local Plausible ClickHouse DNS resolution

Some clients fail to resolve short service names reliably in local, causing Plausible init/migrations to fail with NXDOMAIN-like errors. The local overlay must use an explicit FQDN for ClickHouse.

Related tracking: see `backlog/422-plausible-argocd-alignment.md`.

### Gateway API SSA ownership (avoid manual `kubectl patch`)

Server-side apply field ownership matters. A one-off manual `kubectl patch` against Gateway API resources can steal `.spec.*` ownership and leave the Argo app permanently `OutOfSync` even though everything is Healthy.

Fix direction:
- Treat Git as the only steady-state writer for Argo-managed objects.
- If you must recover ownership, do a one-time SSA apply with the Argo field manager and `--force-conflicts`, then avoid reintroducing out-of-band patches.

Related tracking: see `backlog/450-argocd-service-issues-inventory.md`.

### 3) Scheduling contract (pools and profiles)

We cannot rely on “env pools” if dev is a shared-node cluster.

Target direction:
- **Stateful services** → standard, non-preemptible capacity (future “stateful/standard” profile).
- **Stateless services** → can prefer spot (future “spot” profile) but must safely fallback.
- **Local** has no pools; its profile must be either “none” or “relaxed required + preferred”.

Short-term compatibility:
- Keep `config/node-profiles/none.yaml` for local.
- Use `config/node-profiles/general-pool.yaml` for AKS until the stateful/spot split is introduced.
- Remove remaining env-specific affinity/toleration assumptions from env overlays.

## CI guardrails (must cover both local and dev)

Add a GitHub Actions job (or extend existing gates) that, on every PR:

1. Renders a representative set of Helm charts for:
   - `env/local` with `config/node-profiles/none.yaml`
   - `env/dev` with `config/node-profiles/general-pool.yaml`
2. Validates:
   - no Helm render errors (schema-required values must exist)
   - Kubernetes schema conformance (Gateway API, CRD presence constraints)
   - no duplicate cluster-scoped resource names across envs (SharedResourceWarning patterns)
   - no runtime-critical pulls from `docker.io` for the AKS overlay (allowlist only if needed)
   - multi-arch digest validity for shared images (manifest list, not single-arch)

This is the enforcement mechanism that prevents “fix dev, break local”.

## Seeding status (AKS dev)

Current evidence that baseline seeding/migrations ran:
- `dev-platform-dev-data` is `Synced/Healthy`.
- `job.batch/data-db-migrations` in `ameide-dev` completed successfully (`succeeded=1`).

This aligns with the contract in `backlog/582-local-dev-seeding.md` (same seed/verify approach).

## Work plan (prioritized)

1. **Run CI image mirroring + resync** (Dashboard/Kong + smoke tools + any remaining DockerHub blockers).
2. **Confirm secret plumbing is namespace-complete** (SecretStore + `ghcr-pull` in `clickhouse-system`).
3. **Resolve remaining app-level Degraded states**:
   - `dev-process-transformation-v0-ingress` CrashLoopBackOff (logs + config parity with local)
   - staging/prod Gateway `Degraded` due to missing GRPC backend Service (decide env-conditional route vs deploy backend)
4. **Normalize shared-cluster scheduling**:
   - Kafka scheduling (remove env-specific pool assumptions)
   - Prometheus hostPort DaemonSets (node-exporter strategy)
5. **Add CI guardrails** (render + schema + image policy checks for local+dev).

## Related backlogs / references

- `backlog/444-terraform-v2.md` (CI-owned AKS; remove env-specific pools; long-term standard/spot split)
- `backlog/465-applicationset-architecture-preview-envs-v2.md` (values layering; gateway route requirement)
- `backlog/582-local-dev-seeding.md` (seed/verify contract)
- `backlog/603-image-pull-policy.md` / `backlog/602-image-pull-policy.md` (digest pinning + :main automation)
- `backlog/420-temporal-cnpg-dev-registry-runbook.md` (Temporal operator/webhook and recovery)
- `backlog/447-third-party-chart-tolerations.md` (DaemonSet hostPort conflicts)
- `backlog/456-ghcr-mirror.md` (multi-arch + mirror pitfalls)
- `backlog/612-kubernetes-dashboard.md` (dashboard deployment model)
- `backlog/454-coredns-cluster-scoped.md` (SharedResourceWarning is misconfiguration)
