# backlog/364 – Argo configuration v3 (bootstrap plan)

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

This note consolidates the decisions discussed for promoting Argo CD to the first-class bootstrap artifact outside Tilt, aligning with our **v3 GitOps shape** while keeping the developer inner loop lightweight.

---

## 1. Objectives

- **Argo first, everything else later.** The Argo namespace, controller, and application tree must be ready before any infra or product workloads start. Argo is now “layer 00”.
- **Bootstrap outside Tilt.** Devcontainer startup (and any other bootstrap channel) is responsible for installing Argo; Tilt should assume Argo already exists and only wait for it.
- **Environment-appropriate git targets.**
  - Production Applications track `main`.
  - Staging Applications track `staging`.
  - Local Applications render directly from the workspace via `file://` sources so contributors see edits without pushing branches.
- **Keep the inner loop clean.** After Argo owns infra reconciliation we can migrate every pre-Tilt helmfile into GitOps, leaving Tilt to build/push images and apply first-party services only.

---

## 2. Layer renumbering (“00-argocd”)

| Item | Action |
| --- | --- |
| Helmfile | Rename `infra/kubernetes/helmfiles/16-argocd.yaml` → `00-argocd.yaml`. Update `infra/kubernetes/helmfile.yaml` ordering, comments, and any prose references. |
| Labels/selectors | Within the helmfile releases, keep `stage=argocd` but set `layer: 00-argocd` to match the new file name. |
| Dependencies | Remove `infra:16-argocd` from `tilt/infra_layers.tilt` entirely and drop it from every `deps` list that previously referenced it. |
| Guardrails | When other layers still need to ensure Argo is up (during the migration period), add a lightweight Tilt local_resource (e.g., `argo-bootstrap-ready`) that merely waits on the `argocd` deployments instead of reapplying Helm. |

Result: Argo stops being a Tilt-managed layer. Instead it is a prerequisite satisfied by bootstrap.

---

## 3. Devcontainer bootstrap flow

Bootstrap now does only what is required to bring Argo online, then waits for Argo to finish provisioning the foundations:

0. Merge the k3d kubeconfig into `/home/vscode/.kube/config`, select the `k3d-ameide` context, then run `scripts/dev/wait-for-kube-api.sh` so `/readyz` **and** `kubectl version --short` succeed before any manifests are applied. This script is invoked from both `postCreate` and `postStart`.
1. Execute `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/00-argocd.yaml` with `HELM_LAYER_SELECTOR=stage=argocd` and `HELMFILE_ENVIRONMENT=local`. This installs the Argo namespace, CRDs, and controllers before any Argo CRs are applied.
2. Wait for the CRDs to report `Established` so Argo CRs won’t race the API server:  
   `kubectl wait --for=condition=Established crd applications.argoproj.io appprojects.argoproj.io applicationsets.argoproj.io argocds.argoproj.io --timeout=60s`
3. No additional bootstrap steps are executed automatically; once the Helm release is applied you can manage AppProjects/parent manifests manually (or via Argo itself) as needed.
4. Let the ApplicationSets’ `strategy: RollingSync` gate cross-application rollouts. Run `argocd app wait --selector tier=foundation --health` until every foundation Application is Healthy, then mark the devcontainer “ready” and allow Tilt to start. RollingSync progresses automatically to `tier=platform` and `tier=app` once the foundation cohort finishes, so we no longer maintain a hardcoded application list here.

Integration points:

- `.devcontainer/postCreateCommand.sh`: runs the full bootstrap (steps 0‑6) on first container creation.
- `.devcontainer/postStartCommand.sh`: reapplies steps 0‑6 on every restart so drift is reconciled before Tilt resumes.
- Logs go to `/home/vscode/.devcontainer-setup.log` and `/home/vscode/poststart-command.log`; failures are summarized so engineers know what to inspect.

This ensures every namespace/CRD/operator/secret comes from Argo, while bootstrap merely waits for convergence.

---

## 4. GitOps source targeting

| Environment | repoURL | targetRevision | Notes |
| --- | --- | --- | --- |
| Production | `https://github.com/ameideio/ameide.git` | `main` | Defined in `gitops/argocd/parent/apps-root-production.yaml` and all `production.yaml` overlays. |
| Staging | `https://github.com/ameideio/ameide.git` | `staging` | Defined in `gitops/argocd/parent/apps-root-staging.yaml` + `applicationsets/staging.yaml`. |
| Local | `file:///workspace` | `HEAD` (git not required) | Defined in `gitops/argocd/parent/apps-root-local.yaml` + `applicationsets/local.yaml`. Local overlays mount `/workspace` into the repo-server. |

Local repo access:

- Register the repository via `configs.repositories` in the chart values (the bootstrap overlay) so Argo trusts `file:///workspace`. Because file repositories don’t need credentials, no secret is required.
- The devcontainer `HOST_WORKSPACE_DIR` is bind-mounted into every k3d node (`scripts/infra/ensure-k3d-cluster.sh` renders the `volumes` stanza in `infra/docker/local/k3d.yaml`) so the repo-server pod sees the same absolute `/workspace` path. `infra/kubernetes/environments/local/platform/argocd.yaml` adds the hostPath via `repoServer.volumes/volumeMounts`, while the CMP sidecar only mounts `/var/run/argocd`, `/home/argocd/cmp-server/{plugins,config}`, its own `/tmp`, and `/workspace` when the plugin requires it.
- Call out that this hostPath wiring is **dev-only**. Shared clusters (staging/prod) must continue to source manifests from Git so no hostPath is rendered outside the k3d profile.
- Document the behavior in `.devcontainer/README.md` (local edits sync instantly; shared envs still track their canonical branches).

---

## 5. Tilt behavior during the transition

1. Delete the `infra:16-argocd` spec from `tilt/infra_layers.tilt`.
2. Add a small `local_resource("argo-bootstrap-ready", "scripts/dev/wait-for-argo.sh", ...)` that:
   - Checks for the `argocd` namespace.
   - Waits on the core deployments (same as the bootstrap script but without running Helm).
   - Is listed as a dependency for every remaining infra layer resource so Tilt will not proceed until Argo is healthy.
3. When running outside the devcontainer, set an env var (e.g., `AMEIDE_BOOTSTRAP_ARGO_VIA_TILT=1`) to allow the wait script to invoke the helmfile layer directly. This keeps CI or bespoke environments functional until they adopt the new bootstrap flow.

Eventually, once all infra helmfiles are managed by Argo Applications, we can remove the infra resources from Tilt entirely, leaving Tilt to handle only developer services and live updates.

---

## 6. Migration checklist

1. ✅ Rename helmfile to `00-argocd` + update references.
2. ✅ Remove `infra:16-argocd` from Tilt; add the readiness local_resource.
3. ✅ Land `.devcontainer/bootstrap-argocd.sh` and wire it into `postCreate`/`postStart`.
4. ✅ Replace the `argocd-apps` chart with the v3 GitOps layout (`gitops/argocd/{projects,parent,foundation,platform,apps,applicationsets,ops}`) so every environment now syncs through env-specific parent + ApplicationSet manifests.
5. ✅ Ensure the Argo chart values register the local `file:///workspace` repository so Argo trusts the devcontainer mount.
6. ✅ Document the workflow in `.devcontainer/README.md` (including default admin credentials).
7. 🟡 Foundation components are split into individual Applications (`foundation/*/env.yaml`). Continue removing the legacy Helmfile releases as each Application proves stable.
8. 🔲 Once infra is GitOps-managed, simplify Tilt to only build/test/run inner-loop services.

This doc (backlog/364-argo-configuration-v3-bootstrap.md) is the source of truth for the bootstrap phase. Future backlog items will track the migration of each helmfile layer into Argo Applications and the final cleanup of Tilt infra resources.
