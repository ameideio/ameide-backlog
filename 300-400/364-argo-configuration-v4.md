# backlog/364 – Argo configuration v4 (bootstrap + GitOps contract)

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

Version 4 collapses the vendor-aligned GitOps design and the bootstrap execution plan into a single source of truth. It keeps Argo CD as layer 00, spells out how devcontainer bootstrap owns installation, and reiterates the hierarchy, guardrails, and patterns we expect everywhere else.

---

## 1. Objectives & guardrails

- **Argo first, everything else later.** The `00-argocd` Helm chart installs only Argo CD (namespace + controllers); the parent Application and GitOps tree are applied separately once Argo is healthy. ([Argo CD][9])
- **Bootstrap outside Tilt.** Devcontainer `postCreate`/`postStart` scripts are responsible for installing Argo, registering repositories, and waiting for convergence; Tilt simply waits for Argo to be healthy before managing developer services.
- **Environment-appropriate Git targets.** Production Applications pin to `main`, staging pins to `staging`, and local Applications render directly from `file:///workspace` so contributors see edits instantly.
- **Admin-owned App-of-Apps, tenant guardrails.** The parent Application is operated by platform admins, while AppProjects + ApplicationSets define how foundation/platform/apps workloads are orchestrated and who can self-serve. ([Argo CD][1], [Argo CD][4])
- **Deterministic ordering with health-based gating.** Use sync waves inside an Application, ApplicationSet RollingSync across Applications, and custom health checks so “Healthy” means truly ready. ([Argo CD][5], [Argo CD][7], [Argo CD][13])
- **Secrets stay in-cluster.** Manage sensitive data with External Secrets/Vault operators on the destination cluster; avoid injecting secrets during manifest generation. ([Argo CD][5])

---

## 2. Layer 00 + bootstrap pipeline

### 2.1 Layer 00 (“00-argocd”) – Argo CD chart

| Item | Action |
| --- | --- |
| Chart release | Treat the pinned Argo CD Helm chart (`00-argocd`) as layer 00 and install it via a single `helm upgrade --install` call. |
| Labels/selectors | Keep `stage=argocd`, and set `layer: 00-argocd` across manifest metadata so tooling can gate on it. |
| Tilt dependencies | Remove `infra:16-argocd` from `tilt/infra_layers.tilt`; remaining infra layers depend on a lightweight Argo readiness wait. |
| Guardrails | Use the wait described in §2.5 so no other layer runs until the Argo Deployments/StatefulSet are healthy. |

Result: bootstrap satisfies Argo once, then leaves GitOps objects to Argo itself.

### 2.2 Devcontainer bootstrap flow

0. **API gate.** Merge the k3d kubeconfig into `/home/vscode/.kube/config`, select `k3d-ameide`, and run `scripts/dev/wait-for-kube-api.sh` (configurable timeout/interval) until `/readyz` **and** `kubectl version` succeed. (`postCreate` and `postStart` both call this.)
1. **Install/upgrade Argo CD (single Helm command).** `/.devcontainer/bootstrap-argocd.sh` wraps one `helm upgrade --install argo-cd` invocation against the vendored OCI chart, pinned to the approved version, with `--namespace argocd --create-namespace`, `--wait --timeout 5m --atomic`, and the environment-specific values file `infra/kubernetes/environments/<env>/platform/argocd.yaml`.
2. **Controller readiness (no extra work).** The Helm call already uses `--wait`, but we still surface per-deployment rollout logs (`argocd-repo-server`, `argocd-application-controller` statefulset, `argocd-server`, `argocd-applicationset-controller`) for clarity.
3. **Seed admin access.** `/.devcontainer/ensure-argocd-secret.sh` keeps the local admin credential deterministic (`admin / C1tr0n3lla!`) so engineers can reach the UI without scraping pod secrets.
4. **Apply the admin GitOps tree.** `scripts/infra/argocd-apply-admin.sh <env>` applies AppProjects, the parent Application, and the high-priority platform Applications (Envoy controller/config) after Argo is healthy. This keeps layer 00 clean while still letting bootstrap reconcile the authoritative manifests for local dev.
5. **Wait for the Envoy Applications.** `scripts/dev/wait-for-argo-apps.sh platform-envoy-controller-local platform-envoy-config-local` blocks until Envoy reaches `Synced/Healthy`, guaranteeing ingress/gateway readiness before Tilt moves on. Other applications can be added to the wait list as they migrate to Argo.

**Integration points**

- `.devcontainer/postCreateCommand.sh` runs the full sequence once.
- `.devcontainer/postStartCommand.sh` reapplies it on every restart to reconcile drift before Tilt resumes.
- Logs stream to `/home/vscode/.devcontainer-setup.log` and `/home/vscode/poststart-command.log`; failures are summarized so engineers know where to look.

### 2.3 GitOps source targeting

| Environment | repoURL | targetRevision | Notes |
| --- | --- | --- | --- |
| Production | `https://github.com/ameideio/ameide.git` | `main` | `gitops/argocd/parent/apps-root-production.yaml` and production overlays |
| Staging | `https://github.com/ameideio/ameide.git` | `staging` | `gitops/argocd/parent/apps-root-staging.yaml` + staging ApplicationSets |
| Local | `file:///workspace` | `HEAD` | `gitops/argocd/parent/apps-root-local.yaml` + local ApplicationSets. Local overlays mount `/workspace` into the repo-server, and the file repo is registered via chart `configs.repositories` (dev-only). |

### 2.4 Local repo access (dev-only)

- Register `file:///workspace` via the Argo chart’s `configs.repositories` entry so Argo trusts the local source; file repositories need no credentials.
- `scripts/infra/ensure-k3d-cluster.sh` bind-mounts `${HOST_WORKSPACE_DIR}:/workspace` into every k3d node so the repo-server sees the same absolute path. `infra/kubernetes/environments/local/platform/argocd.yaml` adds this hostPath via `repoServer.volumes/volumeMounts`, while CMP sidecars only mount `/var/run/argocd`, `/home/argocd/cmp-server/{plugins,config}`, `/tmp`, and `/workspace` when the plugin needs it.
- Call out that the hostPath is dev-only; shared clusters (staging/prod) must continue to pull from Git, and those value files must not include the hostPath or `file://` repo registration.
- Document the behavior in `.devcontainer/README.md` (local edits sync instantly but shared envs track canonical branches).

### 2.5 Tilt behavior during the transition

1. Delete the `infra:16-argocd` spec from `tilt/infra_layers.tilt`.
2. Add `local_resource("argo-bootstrap-ready", "scripts/dev/wait-for-argo.sh", …)` that only waits on the Argo namespace and controller Deployments/StatefulSet via `kubectl rollout status`, then list it as a dependency for every remaining infra resource.
3. Outside the devcontainer (CI or bespoke hosts), install Argo by running the same `helm upgrade --install` command before Tilt starts; the wait resource simply validates health and never re-applies Helm.

Eventually, once infra helmfiles become Argo Applications, Tilt only handles developer services and live updates.

---

## 3. GitOps hierarchy & repository layout

```
gitops/
└── argocd/
    ├── projects/                  # AppProject guardrails
    ├── parent/                    # admin-only app-of-apps
    ├── foundation/                # cluster-scoped + prerequisites
    ├── platform/                  # curated third-party services
    ├── apps/                      # first-party workloads
    ├── applicationsets/           # generators per env/tier
    └── ops/                       # argocd-cm, health, CMP config
```

* The parent app keeps `spec.source.directory.recurse: true` so it auto-discovers child Application and ApplicationSet manifests while remaining thin. ([Argo CD][6], [Argo CD][9])
* ApplicationSets (per env/tier) use the Git generator plus RollingSync to fan out one Application per folder/env and gate rollouts on actual health. ([Argo CD][1], [Argo CD][13])
* AppProjects enforce source/destination/kind policies, orphaned-resource monitoring, and sync windows for each responsibility boundary. ([Argo CD][4], [Argo CD][10], [Argo CD][11])

Foundation components continue to be carved into discrete Applications (e.g., `foundation/crds-*`, `foundation/operators/*`). Each time an Application proves stable, its legacy Helmfile release is removed and Tilt stops touching it.

---

## 4. Control-plane patterns

### 4.1 Parent Application (App-of-Apps)

Owned by platform admins, lives under `gitops/argocd/parent/`, and ignores benign diffs in the child specs to avoid churn. The parent Application targets the `foundation` project, points its source at `gitops/argocd` with `directory.recurse: true`, deploys to the in-cluster server/`argocd` namespace, enables automated sync with self-heal/prune, adds `RespectIgnoreDifferences`, and sets `ignoreDifferences` rules for child Application sync policy, refresh annotations, and operation fields. The App-of-Apps behavior and child deletion semantics are documented in the Argo CD bootstrapping guide. ([Argo CD][9])

### 4.2 AppProjects (tenancy guardrails)

Create at least three projects (`foundation`, `platform`, `apps`) with strict source/destination policies, optional cluster resource allowlists, and orphaned-resource warnings. For example, the `platform` project whitelists the `https://github.com/ameideio/*` repos, allows all namespaces on the in-cluster destination, turns on orphaned-resource warnings, and can define deny sync windows (e.g., weekdays 20:00 UTC for ten hours) for change control. AppProjects are the authoritative place for RBAC, sync windows, and orphan detection. ([Argo CD][4], [Argo CD][10], [Argo CD][11])

### 4.3 ApplicationSets & RollingSync

Use ApplicationSets with the Git generator to emit one Application per service/env. RollingSync groups enforce cross-Application ordering without ad-hoc waits; the controller waits for the matched cohort to become Healthy before syncing the next cohort, and it automatically disables auto-sync on generated children while RollingSync is in effect. ([Argo CD][1], [Argo CD][13])

Reference shape: the `platform-services` ApplicationSet uses RollingSync with two steps (`tier=foundation` then `tier=platform`), a Git generator that walks `gitops/argocd/platform/*`, and a template that names Applications `svc-<folder>`, labels them `tier=platform`, assigns the `platform` AppProject, targets the folder-specific path/namespace, enables `CreateNamespace`, and limits retries to three attempts.

### 4.4 Parent Application apply

Apply the parent once, outside bootstrap, using any lightweight mechanism:

- **Manual one-liner.** `kubectl apply -f gitops/argocd/parent/apps-root-<env>.yaml`
- **Make target.** e.g., `make argo:parent-apply` that shells out to the same `kubectl apply`.
- **Tiny Helm release.** A dedicated `raw-manifests` style chart containing only the parent Application.

The parent stays separate from the Argo CD chart so controller rollouts remain atomic and ownership stays clear.

---

## 5. Ordering, health, and sync policy

### 5.1 Sync waves (inside an Application)

- Use negative waves for CRDs/controllers, zero for services, positive for dependent workloads. Waves only control apply order inside a single Application. ([Argo CD][5], [Argo CD][12])
- Default 2-second wave delay gives other controllers time to react before health is evaluated.
- Annotate early resources (e.g., CRDs) with `argocd.argoproj.io/sync-wave: "-95"` so they run before operators and workloads.

### 5.2 Hooks vs health checks

- Prefer Lua-based health checks (global via `argocd-cm` or per Application) so RollingSync gates on true readiness rather than on external wait scripts. A typical snippet inspects resource conditions (e.g., `cert-manager` Certificates) and returns `Healthy`, `Degraded`, or `Progressing` based on the `Ready` status. ([Argo CD][7])
- Use resource hooks (PreSync/Sync/PostSync/etc.) only for orchestrating discrete workflows (e.g., migrations). ([Argo CD][3])

### 5.3 Sync policy defaults

- Recommended defaults: enable automated sync with prune + self-heal, set `CreateNamespace=true` and `RespectIgnoreDifferences=true`, and only add `SkipDryRunOnMissingResource`, `ServerSideApply`, or `PruneLast` when a specific workload demands it. These options are documented in the Application spec, and `RespectIgnoreDifferences` ensures ignored fields are honored during sync as well as diff. ([Argo CD][3], [Argo CD][6])

### 5.4 Diff customization

Silence controller-mutated fields (e.g., webhook `caBundle` values or TLS secret data) via `ignoreDifferences` so they do not appear as drift; set the rule either per Application or globally. ([Argo CD][8])

### 5.5 Secrets

Leverage External Secrets/Vault operators on the destination clusters; avoid injecting sensitive values via repo-server plugins so secrets never leave the cluster boundary. ([Argo CD][5])

---

## 6. Config management, guardrails, and naming

- **Config Management Plugins + Helmfile.** Default path is native Helm/Kustomize/plain manifests; only reach for Helmfile or other non-native tools when necessary, and then install them via the repo-server sidecar CMP model (v2) so plugins run in a constrained container with explicit mounts. Treat plugins as trusted code and honor controller timeouts. ([Argo CD][12])
- **Finalizers & orphaned resources.** Keep finalizers on child Applications when you want deletion cascades; enable `orphanedResources.warn` inside AppProjects to flag manual edits. ([Argo CD][4])
- **Sync windows & automation.** Default to auto-sync + self-heal; add deny windows for production change-control. ([Argo CD][11], [Argo CD][15])
- **Resource tracking.** Use Argo CD’s built-in label-based tracking unless you have a documented reason to switch. ([Argo CD][16])
- **Naming, labels, annotations.** Prefer `tier=<foundation|platform|app>`, `env=<env>`, and `owner=<team>` labels; annotate sync waves via `argocd.argoproj.io/sync-wave`. These labels also drive RollingSync group selectors. ([Argo CD][12], [Argo CD][14])

---

## 7. Example manifests (golden templates)

### Application (third-party workload)

Use `svc-<component>` naming, label the workload `tier=platform`, annotate it with sync-wave `0`, and attach it to the `platform` AppProject. Source manifests from the component folder under `gitops/argocd/platform`, deploy into the matching namespace on the in-cluster destination, and enable automated sync with `CreateNamespace` plus `RespectIgnoreDifferences`.

### ApplicationSet (per-env fan-out)

Generate one Application per service/environment with a list generator (`env=dev|staging|prod`). Name each Application `<service>-<env>`, label it `tier=app` with the environment value, and point the source at `gitops/argocd/apps/<service>`. RollingSync gates dev → staging → prod, while the template enables automated sync and targets a namespace matching the service.

### AppProject (apps tier)

Allow sources from `https://github.com/ameideio/ameide.git`, permit all namespaces on the in-cluster destination, enable orphaned-resource warnings, and define a deny sync window (e.g., weekdays 20:00 UTC for ten hours) for production change control.

---

## 8. Anti-patterns to avoid

- One giant Application mixing unrelated concerns—prefer many small Applications orchestrated via Projects and ApplicationSets. ([Argo CD][1])
- Relying on sync waves for cross-Application readiness—use RollingSync for that; waves only order resources inside one Application. ([Argo CD][5], [Argo CD][13])
- Treating controller-mutated fields as drift—use `ignoreDifferences` and `RespectIgnoreDifferences`. ([Argo CD][8])

---

## 9. Migration checklist

1. ✅ Pin the `00-argocd` Helm chart release and update references.
2. ✅ Remove `infra:16-argocd` from Tilt and add the readiness `local_resource`.
3. ✅ Land `.devcontainer/bootstrap-argocd.sh` and wire it into `postCreate`/`postStart`.
4. ✅ Replace the `argocd-apps` chart with the v3 GitOps layout (`gitops/argocd/{projects,parent,foundation,platform,apps,applicationsets,ops}`).
5. ✅ Ensure the Argo chart values register the local `file:///workspace` repository.
6. ✅ Document the workflow in `.devcontainer/README.md` (including default admin credentials).
7. 🟡 Keep splitting foundation components into individual Applications and remove the corresponding Helmfile releases once proven.
8. 🔲 After every infra helmfile migrates into GitOps, simplify Tilt to only build/test/run inner-loop services.

This document supersedes the separate v3 bootstrap plan and serves as the authoritative reference for continuing the migration.

---

## 10. Value & next steps

Adopting this structure delivers deterministic ordering, safe rollouts, strong multi-tenancy, accurate health gating, clean diffs, and predictable bootstrapping. ([Argo CD][5], [Argo CD][7], [Argo CD][8], [Argo CD][9])  
Next: produce a v2 → v4 mapping (structure, projects, sync options, dependency handling, naming) to mechanically plan remaining migrations.

---

[1]: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/?utm_source=chatgpt.com "Introduction to ApplicationSet controller - Argo CD"
[2]: https://argo-cd.readthedocs.io/?utm_source=chatgpt.com "Argo CD - Declarative GitOps CD for Kubernetes"
[3]: https://argo-cd.readthedocs.io/en/latest/user-guide/application-specification/?utm_source=chatgpt.com "Application Specification Reference - Argo CD - Read the Docs"
[4]: https://argo-cd.readthedocs.io/en/stable/user-guide/projects/?utm_source=chatgpt.com "Projects - Argo CD - Declarative GitOps CD for Kubernetes"
[5]: https://argo-cd.readthedocs.io/en/release-2.9/user-guide/sync-waves/?utm_source=chatgpt.com "Sync Phases and Waves - Argo CD - Read the Docs"
[6]: https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/?utm_source=chatgpt.com "Sync Options - Argo CD - Declarative GitOps CD for Kubernetes"
[7]: https://argo-cd.readthedocs.io/en/latest/operator-manual/health/?utm_source=chatgpt.com "Resource Health - Declarative GitOps CD for Kubernetes"
[8]: https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/?utm_source=chatgpt.com "Diff Customization - Declarative GitOps CD for Kubernetes"
[9]: https://argo-cd.readthedocs.io/en/latest/operator-manual/cluster-bootstrapping/?utm_source=chatgpt.com "Cluster Bootstrapping - Declarative GitOps CD for Kubernetes"
[10]: https://argo-cd.readthedocs.io/en/latest/operator-manual/project-specification/?utm_source=chatgpt.com "Project Specification Reference - Argo CD - Read the Docs"
[11]: https://argo-cd.readthedocs.io/en/release-2.14/user-guide/sync_windows/?utm_source=chatgpt.com "Sync Windows - Declarative GitOps CD for Kubernetes"
[12]: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/?utm_source=chatgpt.com "Sync Phases and Waves - Argo CD - Read the Docs"
[13]: https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Progressive-Syncs/?utm_source=chatgpt.com "Progressive Syncs - Declarative GitOps CD for Kubernetes"
[14]: https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/applicationset-specification/?utm_source=chatgpt.com "ApplicationSet Specification Reference - Argo CD"
[15]: https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/?utm_source=chatgpt.com "Automated Sync Policy - Declarative GitOps CD for Kubernetes"
[16]: https://argo-cd.readthedocs.io/en/latest/user-guide/resource_tracking/?utm_source=chatgpt.com "Resource Tracking - Declarative GitOps CD for Kubernetes"
