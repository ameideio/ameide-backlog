Below is a clean, vendor‑aligned **v3** design for Argo CD GitOps. It describes an ideal end‑state using current Argo CD semantics and official patterns—intentionally independent of any prior versions. After you review, we can do a separate mapping from v2 → v3.

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

---

# Argo CD GitOps Hierarchy – **v3 (vendor‑aligned)**

## 1) Goals & principles

* **Admin‑owned bootstrap; app teams self‑serve.** Use the **App‑of‑Apps** pattern to bootstrap a cluster, but treat the parent app as **admin‑only** and gate who can create child Applications via AppProjects. ([Argo CD][1])
* **Clear separation of concerns.** Split manifests by *foundation* (cluster prerequisites), *platform services* (third‑party), and *first‑party apps*. Manage each group with dedicated **AppProjects** and **ApplicationSets**. ([Argo CD][2])
* **Deterministic ordering & safe rollouts.** Use **Sync Waves** for *in‑application* ordering (e.g., CRDs before CRs), and **ApplicationSet Progressive Syncs (RollingSync)** to gate *cross‑application* rollouts on real health. ([Argo CD][3])
* **Prefer health‑based gating over ad‑hoc waits.** Encode readiness using **built‑in or custom health checks**, and only use hooks for discrete orchestration (e.g., one‑off migrations). ([Argo CD][4])
* **Secrets stay in the cluster.** Use External Secrets / Vault operators on the destination cluster; avoid injecting secrets during manifest generation. ([Argo CD][5])

### Bootstrap posture (layer 00)

Refer to `backlog/364-argo-configuration-v3-bootstrap.md` for the operational plan. Key expectations:

1. **Argo is layer 00.** The `00-argocd` helmfile (namespace + controller + GitOps bootstrap manifests + parent app) installs before any other infrastructure is touched, using vendored charts/resources.
2. **Devcontainer owns bootstrap.** `.devcontainer/postCreateCommand.sh` applies the AppProjects/argocd-cm, seeds `argocd-secret` (default login `admin / C1tr0n3lla!`), runs the `00-argocd` helmfile, and then waits for Argo-managed foundation Applications to reach `Synced/Healthy`. Tilt only starts after those checks pass.
3. **Environment targets:** production Applications pin to `main`, staging pins to `staging`, and local Apps render from `file:///workspace` via the repo-server hostPath mount plus the repo registration in `argocd-cm`.
4. **Bootstrap gates on a ready API.** Both `postCreate` and `postStart` call `scripts/dev/wait-for-kube-api.sh` so `/readyz` and `kubectl version --short` succeed before any manifest (including bootstrap) runs.
5. **Workspace hostPath is intentional.** `scripts/infra/ensure-k3d-cluster.sh` bind-mounts `${HOST_WORKSPACE_DIR}:/workspace` into every k3d node, and `infra/kubernetes/environments/local/platform/argocd.yaml` adds that path via `repoServer.volumes/volumeMounts` (with the CMP sidecar mounting only the directories it needs). This is the supported way to make `file:///workspace` sources work locally.

This bootstrap contract is part of the v3 design and must remain aligned as we migrate remaining helmfile layers into Argo.

---

## 2) Repository & object layout (reference shape)

```
gitops/
└── argocd/
    ├── projects/                  # AppProject guardrails
    │   ├── project-foundation.yaml
    │   ├── project-platform.yaml
    │   └── project-apps.yaml
    ├── parent/                    # admin-only app-of-apps
    │   └── apps-root.yaml
    ├── foundation/                # cluster-scoped / cross-namespace prerequisites
    │   ├── crds/                  # e.g., cert-manager, gateway API, ESO CRDs
    │   ├── operators/             # cert-manager, ESO, ingress/gateway controllers
    │   └── observability/         # log/metrics base, tracing infra
    ├── platform/                  # cataloged third-party services (one Application each)
    │   ├── keycloak/
    │   ├── temporal/
    │   └── prometheus/
    ├── apps/                      # first-party apps (one Application per service)
    │   ├── platform-api/
    │   └── agents/
    ├── applicationsets/           # generators over folders/environments/clusters
    │   ├── platform-appset.yaml
    │   └── apps-appset.yaml
    └── ops/                       # argocd-cm customizations, health checks, etc.
        ├── argocd-cm.yaml
        └── resource-customizations/
```

**Why this shape:**

* The **parent** app loads `foundation/`, `platform/`, `apps/`, and `applicationsets/` using **directory recursion**, keeping the parent thin while preserving discoverability. ([Argo CD][6])
* **ApplicationSets** use the **Git generator** to emit one Application per folder (and/or environment/cluster) and can progress updates safely via **RollingSync**. ([Argo CD][7])
* **AppProjects** enforce source/destination/kind policies per layer and enable **orphaned‑resource monitoring**. ([Argo CD][2])

---

## Current implementation snapshot

* `infra/kubernetes/gitops/argocd/` matches the reference layout: AppProjects under `projects/`, env-scoped parents under `parent/`, foundation/platform/app specs under their respective folders, ApplicationSets per environment, and `ops/argocd-cm.yaml` for CMP/custom health.
* Foundation components are being split into discrete Applications (e.g., `foundation/bootstrap-namespace`, `foundation/crds-gateway`, `foundation/secrets-argocd`). As each one proves stable, its legacy Helmfile release is removed and Tilt stops orchestrating it.
* `00-argocd.yaml` now only handles the Argo control plane plus the raw manifests needed for the GitOps tree (AppProjects, argocd-cm, parent). Everything else is managed via Applications.
* ApplicationSets use RollingSync keyed off the `tier` labels so foundation→platform→apps roll out deterministically.
* Bootstrap scripts already wait for the foundation Applications before signaling “ready”, but we still have to finish migrating the remaining operators/secrets/control-plane releases and remove their Helmfile/Tilt counterparts.

---

## 3) Control‑plane patterns

### 3.1 Parent app (App‑of‑Apps)

* Lives under `parent/apps-root.yaml`, owned by platform admins.
* Uses `spec.source.directory.recurse: true` to pull child Application and ApplicationSet manifests from subfolders.
* Ignores benign diffs in child Application specs to avoid churn (e.g., the child’s sync policy or operation fields changed by controllers). ([Argo CD][6])

**Parent skeleton**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-root
  namespace: argocd
spec:
  project: foundation
  source:
    repoURL: https://example.com/org/gitops.git
    targetRevision: main
    path: gitops/argocd
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions:
      - RespectIgnoreDifferences=true
  ignoreDifferences:
    - group: "*"
      kind: Application
      jsonPointers:
        - /spec/syncPolicy/automated
        - /metadata/annotations/argocd.argoproj.io~1refresh
        - /operation
```

(“Admin‑only” and “ignore differences” are recommended in the App‑of‑Apps doc.) ([Argo CD][1])

### 3.2 AppProjects (guardrails)

Create at least three projects: **foundation**, **platform**, **apps**. Lock down **sourceRepos**, allowed **destinations**, and (for cluster‑scoped kinds) **clusterResourceWhitelist**; enable **orphanedResources.warn**. ([Argo CD][2])

**Example**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  sourceRepos:
    - https://example.com/org/*
  destinations:
    - server: https://kubernetes.default.svc
      namespace: "*"
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'     # (tighten as you mature)
  orphanedResources:
    warn: true
```

### 3.3 ApplicationSets (scale safely)

Use **Git generators** to create one Application per service/env/cluster. When a set spans many apps, enable **Progressive Syncs → `type: RollingSync`** and define label‑based steps so the controller waits for Applications to become **Healthy** before proceeding. **RollingSync forces autosync off** on generated children; it still respects the child’s retry/prune policies when it triggers a sync. ([Argo CD][8])

**Example**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: platform-services
spec:
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: layer
              operator: In
              values: ["foundation"]
        - matchExpressions:
            - key: layer
              operator: In
              values: ["platform"]
  generators:
    - git:
        repoURL: https://example.com/org/gitops.git
        revision: main
        directories:
          - path: gitops/argocd/platform/*
  template:
    metadata:
      name: "svc-{{path.basename}}"
      labels: { layer: "platform" }
    spec:
      project: platform
      source:
        repoURL: https://example.com/org/gitops.git
        targetRevision: main
        path: "gitops/argocd/platform/{{path.basename}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"
      syncPolicy:
        retry: { limit: 3 }
        syncOptions:
          - CreateNamespace=true
```

(Enable Progressive Syncs via controller flag/ConfigMap as in the docs.) ([Argo CD][8])

---

## 4) Ordering, hooks & health

### 4.1 Sync Waves (in‑application ordering)

* Use **negative waves** for CRDs and foundation controllers, then move upward to services (≈0) and apps (>0). Waves control *apply order* inside an Application; they don’t guarantee health across Applications.
* Default **2‑second** wave delay gives other controllers time to react before health is assessed. ([Argo CD][3])

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-95" # e.g., CRDs
```

### 4.2 Hooks (for orchestration), but prefer health for steady‑state

Use **Resource Hooks** (PreSync/Sync/PostSync/SyncFail/PostDelete) when you truly need orchestration (e.g., DB migration). For steady‑state readiness, prefer **health checks** over “wait” Jobs—e.g., encode `ExternalSecret` or `Certificate` readiness in **Lua** custom health checks via `argocd-cm`. ([Argo CD][9])

```yaml
# Example: custom health for cert-manager Certificate
data:
  resource.customizations.health.cert-manager.io_Certificate: |
    hs = {}
    if obj.status and obj.status.conditions then
      for _, c in ipairs(obj.status.conditions) do
        if c.type == "Ready" and c.status == "True" then
          hs.status = "Healthy"; hs.message = c.message; return hs
        elseif c.type == "Ready" and c.status == "False" then
          hs.status = "Degraded"; hs.message = c.message; return hs
        end
      end
    end
    hs.status = "Progressing"; hs.message = "Waiting for Ready"
    return hs
```

(Argo CD documents custom health checks and provides examples.) ([Argo CD][4])

### 4.3 Gateway API & Ingress health contract

RollingSync waves now gate on **dataplane readiness** instead of “object exists.” Custom Lua in `argocd-cm` enforces:

* **Condition polarity awareness.** Negative Gateway API conditions (`Conflicted`, `Detached`, `NoResources`, etc.) only cause *Degraded* when they are explicitly `True`, while positive conditions (`Accepted`, `Programmed`, `Ready`, `ResolvedRefs`) must be `True`—and have a current `observedGeneration`—before declaring a resource *Healthy*.
* **Route parents must be programmed.** `HTTPRoute` and `GRPCRoute` stay *Progressing* until at least one parent reports `Accepted=True` **and** `Programmed/Ready=True` with `ResolvedRefs=True`. Fatal reasons (`Invalid`, `Unsupported`, `Conflict`, …) surface immediately as *Degraded* so RollingSync halts the wave.
* **Ingress reflects controller status.** The override now looks at real load-balancer assignments or `Ready`/`LoadBalancerReady` conditions; the previous “always Healthy” stub was removed so broken ingress wiring blocks downstream waves.

Keep these overrides in **one source of truth**—`gitops/ameide-gitops/sources/values/common/argocd.yaml` (mirrored into Helm bootstrap values)—and verify with `kubectl -n argocd get cm argocd-cm -o yaml` after changes. This makes Application health deterministic and aligned with Gateway API guidance. ([Argo CD][4], [Kubernetes Gateway API][17])

---

## 5) Sync policy & options (recommended defaults)

At the **Application** level:

```yaml
spec:
  syncPolicy:
    automated: { prune: true, selfHeal: true }   # enable on children you manage directly
    syncOptions:
      - CreateNamespace=true
      - RespectIgnoreDifferences=true
      # Use cautiously & only where needed:
      # - SkipDryRunOnMissingResource=true
      # - ServerSideApply=true
```

* **CreateNamespace** lets each app create its target namespace on first sync. ([Argo CD][10])
* **RespectIgnoreDifferences** ensures ignored fields are respected during sync as well as diff. ([Argo CD][10])
* **SkipDryRunOnMissingResource** helps when CRDs are applied in earlier waves; use sparingly. ([Argo CD][10])
* **ServerSideApply** is useful for big objects or partial patches; enable only where you need it. ([Argo CD][10])

Add **Diff Customization** rules (app‑level or global) for known, controller‑mutated fields (e.g., webhooks `caBundle`). ([Argo CD][11])

---

## 6) Secrets

**Strong recommendation:** Manage secrets on the **destination cluster** via operators such as External Secrets or Vault; do **not** inject secrets during manifest generation in repo‑server/plugins. This reduces leakage risk and decouples secret updates from application rollouts. ([Argo CD][5])

---

## 7) Config Management Plugins (CMP) & Helmfile (optional)

If you use **Helmfile** or other non‑native tools, install them as **Config Management Plugins** using the **repo‑server sidecar** method (CMP v2), not legacy `argocd-cm` plugin registration. Treat plugins as **trusted code** and set appropriate timeouts. ([Argo CD][12])

When authoring plugins, you can leverage Argo CD **build‑environment variables** (e.g., `ARGOCD_APP_NAME`, `ARGOCD_APP_REVISION`) inside the generate step. ([Argo CD][13])

---

## 8) Operational guardrails

* **Finalizers** on child Applications so deletion cascades cleanly (or deliberately preserve). ([Argo CD][1])
* **Orphaned Resources Monitoring** enabled in AppProjects to surface drift from manual edits. ([Argo CD][14])
* **Sync Windows** (per project) for change management in production. ([Argo CD][15])
* **Resource tracking** (labels) is the default; keep it unless you have a reason to switch. ([Argo CD][16])

---

## 9) Example: service Application (third‑party workload)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: svc-temporal
  namespace: argocd
  labels: { layer: "platform" }
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: platform
  source:
    repoURL: https://example.com/org/gitops.git
    targetRevision: main
    path: gitops/argocd/platform/temporal
  destination:
    server: https://kubernetes.default.svc
    namespace: temporal
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions:
      - CreateNamespace=true
      - RespectIgnoreDifferences=true
```

---

## 10) Example: foundation CRDs with early wave & dry‑run skip (only if needed)

```yaml
apiVersion: argoproj.io/v1
kind: List
items:
- apiVersion: apiextensions.k8s.io/v1
  kind: CustomResourceDefinition
  metadata:
    name: externalsecrets.external-secrets.io
    annotations:
      argocd.argoproj.io/sync-wave: "-95"
# In consumers that may apply before CRD register completes:
# metadata.annotations.argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
```

(Sync waves & SkipDryRunOnMissingResource behavior are documented.) ([Argo CD][3])

---

## 11) Health of Applications in App‑of‑Apps

If you orchestrate using the health of **Application** resources (e.g., to drive waves), note that built‑in Application health was removed in Argo CD 1.8; you can restore it with a small Lua snippet in `argocd-cm` (shown in the docs). ([Argo CD][4])

---

## 12) What “good” looks like in the UI

* **Parent**: one app named `apps-root` (Synced/Healthy).
* **Rows**: three distinct sets—`foundation-*`, `svc-*` (platform), and `app-*` (first‑party)—each in the correct **AppProject**.
* **AppSet rollouts**: logs show **RollingSync** steps waiting for groups to reach **Healthy** before proceeding. ([Argo CD][8])

---

## 13) Minimal “Day‑2” checklist

* Projects created with tight **source/destination/kind** rules; `orphanedResources.warn=true`. ([Argo CD][2])
* Parent app synced (recurse on), **ignoreDifferences** in place. ([Argo CD][6])
* Foundation CRDs/controllers in **negative waves**; services/apps later. ([Argo CD][3])
* ApplicationSets enabled with **Progressive Syncs**; autosync not relied on for those children. ([Argo CD][8])
* Custom **health checks** added for key CRDs (e.g., certificates, secrets). ([Argo CD][4])
* Secret management is **destination‑cluster operator‑based**, not render‑time injection. ([Argo CD][5])
* Optional: **Sync Windows** configured for prod projects. ([Argo CD][15])

---

### Appendix: Notes on naming (optional convention)

* **foundation‑*** (cluster primitives), **svc‑*** (third‑party services), **app‑*** (first‑party).
* Common labels: `layer`, `env`, `team`, `app.kubernetes.io/part-of` to power selectors and RollingSync steps. (Label‑based selection is how RollingSync groups apps.) ([Argo CD][8])

---

This v3 provides a neutral, vendor‑aligned baseline you can adopt as‑is. When you’re ready, I’ll map your current v2 artifacts into this structure (paths, labels, projects, and sync semantics) so you end up with an unbiased, durable setup.

[1]: https://argo-cd.readthedocs.io/en/latest/operator-manual/cluster-bootstrapping/ "Cluster Bootstrapping - Argo CD - Declarative GitOps CD for Kubernetes"
[2]: https://argo-cd.readthedocs.io/en/stable/user-guide/projects/ "Projects - Argo CD - Declarative GitOps CD for Kubernetes"
[3]: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/ "Sync Phases and Waves - Argo CD - Declarative GitOps CD for Kubernetes"
[4]: https://argo-cd.readthedocs.io/en/latest/operator-manual/health/ "Resource Health - Argo CD - Declarative GitOps CD for Kubernetes"
[5]: https://argo-cd.readthedocs.io/en/latest/operator-manual/secret-management/ "Secret Management - Argo CD - Declarative GitOps CD for Kubernetes"
[6]: https://argo-cd.readthedocs.io/en/latest/user-guide/directory/?utm_source=chatgpt.com "Directory - Argo CD - Declarative GitOps CD for Kubernetes"
[7]: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/?utm_source=chatgpt.com "Generators - Argo CD - Declarative GitOps CD for Kubernetes"
[8]: https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Progressive-Syncs/ "Progressive Syncs - Argo CD - Declarative GitOps CD for Kubernetes"
[9]: https://argo-cd.readthedocs.io/en/release-2.14/user-guide/resource_hooks/?utm_source=chatgpt.com "Resource Hooks - Declarative GitOps CD for Kubernetes"
[10]: https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/ "Sync Options - Argo CD - Declarative GitOps CD for Kubernetes"
[11]: https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/?utm_source=chatgpt.com "Diff Customization - Declarative GitOps CD for Kubernetes"
[12]: https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/?utm_source=chatgpt.com "Config Management Plugins - Argo CD - Read the Docs"
[13]: https://argo-cd.readthedocs.io/en/latest/user-guide/build-environment/?utm_source=chatgpt.com "Build Environment - Argo CD - Declarative GitOps CD for ..."
[14]: https://argo-cd.readthedocs.io/en/latest/user-guide/orphaned-resources/?utm_source=chatgpt.com "Orphaned Resources Monitoring - Argo CD - Read the Docs"
[15]: https://argo-cd.readthedocs.io/en/release-2.14/user-guide/sync_windows/?utm_source=chatgpt.com "Sync Windows - Declarative GitOps CD for Kubernetes"
[16]: https://argo-cd.readthedocs.io/en/latest/user-guide/resource_tracking/?utm_source=chatgpt.com "Resource Tracking - Declarative GitOps CD for Kubernetes"
[17]: https://gateway-api.sigs.k8s.io/?utm_source=chatgpt.com "Kubernetes Gateway API"
