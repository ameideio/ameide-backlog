# 434 – Unified Environment Naming and Telepresence Wiring

> **Note:** This document remains relevant for environment naming conventions.
> The telepresence wiring and remote development workflow has been further refined in
> [435-remote-first-development.md](435-remote-first-development.md).

> **Related documents:**
> - [435-remote-first-development.md](435-remote-first-development.md) – **NEW**: Remote-first development architecture
> - [367-bootstrap-v2.md](367-bootstrap-v2.md) – Bootstrap design (moved to ameide-gitops)
> - [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md) – DevContainer bootstrap (superseded)
> - [432-devcontainer-modes-offline-online.md](432-devcontainer-modes-offline-online.md) – DevContainer modes (retired)

## Goals
- Standardize environment naming across all contexts: local devcontainer, remote AKS namespaces, and GitOps configurations.
- Enable seamless switching between local inner-loop development and remote cluster work via telepresence.
- Migrate domain naming from `tilt.ameide.io` to `local.ameide.io` for consistency.

## Environment Matrix

| Environment | Kube Context | Namespace | Domain | Purpose | Status |
|-------------|--------------|-----------|--------|---------|--------|
| `ameide-dev` | `ameide-dev` | `ameide-dev` | `dev.ameide.io` | Stable development | ✅ On shared AKS |
| `ameide-staging` | `ameide-staging` | `ameide-staging` | `staging.ameide.io` | Staging | ✅ On shared AKS |
| `ameide-prod` | `ameide-prod` | `ameide-prod` | `ameide.io` | Production | ⏸️ Namespace reserved |

> **Naming note:** As finalized in [435-remote-first-development.md](435-remote-first-development.md), local development tunnels into the shared AKS cluster. All contexts follow the `ameide-{environment}` convention and simply select different namespaces inside the single `ameide` cluster.

### Shared AKS Cluster

- **Resource group:** `Ameide`
- **Cluster name:** `ameide`
- **Kube contexts:** `ameide-dev`, `ameide-staging`, `ameide-prod` (all reference the same API server, differing only by namespace/user bindings)
- **Namespaces:** `ameide-dev`, `ameide-staging`, `ameide-prod`, plus `argocd` and platform shared namespaces

### Control plane & image ownership

- **Argo CD:** A single Argo instance in the `argocd` namespace reconciles every environment namespace using Projects and ApplicationSets. There are no per-namespace Argo installations; isolation is enforced by Git (Projects) and RBAC, not by multiple controllers.
- **GitHub Container Registry (GHCR):** `ameide-gitops` defines the repository/tag per environment. CI pushes images to GHCR, updates the GitOps values, and Argo syncs. No doc/tooling should reference Azure Container Registry directly.
- **Tilt-only releases:** Developers still run Tilt + Telepresence for the inner loop. Those resources carry the `-tilt` suffix and point at developer-scoped tags in GHCR but never alter the Argo-managed baselines.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Remote-First Developer Workflow                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                           ┌──────────────────────┐
                           │  DevContainer        │
                           │  (Tilt + Telepresence│
                           │   against AKS)       │
                           └─────────┬────────────┘
                                     │
                       Telepresence connect/intercept
                                     │
                                     ▼
                    ┌──────────────────────────────────┐
                    │    AKS Cluster: ameide           │
                    │  ┌────────────────────────────┐  │
                    │  │ Namespace: ameide-dev      │  │
                    │  │ Namespace: ameide-staging  │  │
                    │  │ Namespace: ameide-prod     │  │
                    │  └────────────────────────────┘  │
                    │   (Argo GitOps manages baseline)│
                    └──────────────────────────────────┘
```

## Remote AKS Cluster Details

### Verified Cluster Information (2025-12-02)

| Property | Value |
|----------|-------|
| Resource Group | `Ameide` |
| Cluster Name | `ameide` |
| Location | `westeurope` |
| Kubernetes Version | `1.32.7` |
| Nodes | 8 (d2as VMSS) |
| Kube Contexts | `ameide-dev`, `ameide-staging`, `ameide-prod` |

### Current GitOps Configuration (LEGACY)
⚠️ **Note**: The remote AKS cluster uses an **older GitOps structure**:
- **Repo**: `https://github.com/ameideio/ameide.git` (main monorepo)
- **Path**: `infra/kubernetes/gitops/argocd/applications`
- **NOT using**: `https://github.com/ameideio/ameide-gitops` (new structure)

### ArgoCD Status
- ArgoCD is installed and running in `argocd` namespace
- Applications are deployed via app-of-apps pattern
- Some applications showing `Degraded` or `ErrImagePull` status (cluster was stopped/starting)

### Current Namespaces
- `ameide-dev`, `ameide-staging`, `ameide-prod` – per-environment workloads (sharing the same AKS cluster)
- `argocd` – ArgoCD control plane

### Migration Needed

> **This migration is a blocker for:**
> - Bootstrap v2 targeting AKS clusters ([367 Phase 2](367-bootstrap-v2.md#phase-2-remote-cluster-support-in-progress))
> - Telepresence mode ([432 DC-4](432-devcontainer-modes-offline-online.md#epic-dc-4--online-telepresence-mode))

To align with the new structure, the remote cluster needs:

| Task | Description | Status |
|------|-------------|--------|
| **MIGRATE-1** | Delete legacy GitOps path (`ameide.git/infra/kubernetes/gitops/`) | ⏳ Pending |
| **MIGRATE-2** | Point Argo CD at `ameide-gitops` repo with `environments/staging/` path | ⏳ Pending |
| **MIGRATE-3** | Create `gitops/ameide-gitops/environments/staging/` structure | ⏳ Pending |
| **MIGRATE-4** | Create `infra/environments/staging/bootstrap.yaml` | ⏳ Pending |
| **MIGRATE-5** | Namespace split: `ameide` → `ameide-dev`, `ameide-staging`, `ameide-prod` within the single `ameide` cluster | ⏳ Pending |
| **MIGRATE-6** | Create kube contexts/users `ameide-{env}` that target the shared cluster but default to the matching namespace | ⏳ Pending |
| **MIGRATE-7** | Update ApplicationSets to use new environment pattern | ⏳ Pending |
| **MIGRATE-8** | Verify cluster is fully reproducible via bootstrap-v2 | ⏳ Pending |

**Approach:** The legacy GitOps content should be removed entirely (not migrated). The AKS cluster should be bootstrapped fresh using `bootstrap-v2.sh` once the staging environment config exists in `ameide-gitops`.

## Backlog

### Epic ENV-1 – Environment Naming Standardization

- **ENV-10 – Update local domain from tilt.ameide.io to local.ameide.io** ✅ Completed

  Goal: Consistent naming across all environments.

  **Updated components:**
  | File | Change |
  |------|--------|
| *(legacy)* `.devcontainer/manage-host-domains.sh` | removed in the remote-first workflow; Telepresence handles DNS routing |
  | `gitops/ameide-gitops/sources/values/local/apps/platform/gateway.yaml` | `https-local` listener + `auth.local.ameide.io` route |
  | `gitops/ameide-gitops/sources/values/dev/platform/platform-cert-manager-config.yaml` | Wildcard SANs for `*.local.ameide.io` |
  | `gitops/ameide-gitops/sources/values/dev/apps/apps-www-ameide(-platform)-tilt.yaml` | Hostnames + cookie domains moved to `local.ameide.io` |
  | `gitops/ameide-gitops/sources/values/_shared/platform/platform-keycloak-realm.yaml` | Redirect URIs include the local origin |

  Certificates inherit the new SANs automatically and documentation now references `local.ameide.io` (see [367-bootstrap-v2.md §3.8](367-bootstrap-v2.md#38-domain-naming)).

- **ENV-11 – Standardize kube context naming** ⏳ Not started

  Goal: Predictable context names across all environments.

| Environment | Context Name | Command |
|-------------|--------------|---------|
| Dev (AKS) | `ameide-dev` | `az aks get-credentials --name ameide --resource-group Ameide --overwrite-existing --context ameide-dev && kubectl config set-context ameide-dev --namespace=ameide-dev` |
| Staging (AKS) | `ameide-staging` | `az aks get-credentials --name ameide --resource-group Ameide --overwrite-existing --context ameide-staging && kubectl config set-context ameide-staging --namespace=ameide-staging` |
| Production (AKS) | `ameide-prod` | `az aks get-credentials --name ameide --resource-group Ameide --overwrite-existing --context ameide-prod && kubectl config set-context ameide-prod --namespace=ameide-prod` |

  Update bootstrap configs and `postCreate.sh` to reference standardized names.

- **ENV-12 – Document namespace conventions** ⏳ Not started

  Goal: Clear mapping between environments and namespaces.

  | Environment | Namespace | Notes |
  |-------------|-----------|-------|
  | Dev (AKS) | `ameide-dev` | Used by contexts/tools targeting `ameide-dev` |
  | Staging (AKS) | `ameide-staging` | Used by contexts/tools targeting `ameide-staging` |
  | Production (AKS) | `ameide-prod` | Reserved for prod workloads |

  **Decision:** A single AKS cluster hosts all environments; Kubernetes namespaces provide isolation while shared services (e.g., `argocd`, `observability`) remain cluster-scoped.

### Epic ENV-2 – Remote AKS Verification ✅ COMPLETED

- **ENV-20 – Verify AKS cluster is running** ✅
  Goal: Confirm remote cluster is healthy and accessible.
  Result:
- Cluster `ameide` in `Ameide` RG is running (8 nodes, K8s 1.32.7) and serves all environments via namespaces
- Shared `ameide` cluster hosts `ameide-dev`, `ameide-staging`, and `ameide-prod` namespaces; staging workloads now live in `ameide-staging`.
  - kubelogin installed for Azure AD auth
  Status: Completed.

- **ENV-21 – Verify ArgoCD is installed and syncing** ✅
  Goal: Confirm GitOps is operational on remote cluster.
  Result:
  - ArgoCD installed in `argocd` namespace
  - Using **legacy** app-of-apps from `ameide.git/infra/kubernetes/gitops/` (NOT ameide-gitops)
  - Applications: agents, agents-runtime, platform, repository, workflow, www-ameide, www-ameide-platform
  - Some apps degraded due to cluster restart (ErrImagePull, Progressing)
  Status: Completed (with findings).

- **ENV-22 – Verify ameide namespace is healthy** ✅
  Goal: Confirm dev environment workloads are running.
  Result:
- Workloads run inside the per-environment namespaces (`ameide-dev`, `ameide-staging`, `ameide-prod`)
  - Pods starting up after cluster restart
  - Some image pull issues (likely registry auth after cluster scale-up)
  Status: Completed (cluster recovering).

### Epic ENV-3 – Telepresence Implementation

> **Note:** This epic is aligned with [432-devcontainer-modes-offline-online.md](432-devcontainer-modes-offline-online.md#epic-dc-4--online-telepresence-mode) for devcontainer integration.
>
> **Blocker:** Requires [Remote GitOps Migration](#migration-needed) to be completed first.

- **ENV-30 – Install telepresence in devcontainer** ✅ Completed
  - Telepresence CLI installed via `.devcontainer/Dockerfile` (pinned version in `TELEPRESENCE_VERSION` ARG).
  - **Capabilities:** `devcontainer.json` includes `--cap-add=NET_ADMIN` and `--device=/dev/net/tun` in `runArgs` for VIF tunnel support.
  - Maps to [432 DC-31](432-devcontainer-modes-offline-online.md#epic-dc-4--online-telepresence-mode)

- **ENV-31 – Create telepresence connect helper script** ✅ Completed

  `tools/dev/telepresence.sh` exposes `connect|leave|status`, defaults to context `ameide-dev` (override with `TELEPRESENCE_CONTEXT`) / namespace `ameide-dev`, and is invoked automatically from `postCreate.sh`.

- **ENV-32 – Wire telepresence into online-telepresence mode** ✅ Completed
  - `.devcontainer/postCreate.sh` reads the telepresence config, writes `~/.devcontainer-mode.env`, switches the kube context to whichever `ameide-{env}` context is requested (default `ameide-dev`), and auto-connects via `tools/dev/telepresence.sh` when `autoConnect: true`.
  - Tilt picks up `DEV_REMOTE_CONTEXT`/`TILT_REMOTE=1` so remote registries/hosts are wired automatically.

- **ENV-33 – Configure Tilt for remote mode (TILT_REMOTE)** ✅ Completed
  - Add `TILT_REMOTE=1` environment variable support
  - Update Tiltfile to switch registry/context based on env
  - Update image push targets for remote registry (`ghcr.io/ameideio/*`)
  - Maps to [432 DC-32](432-devcontainer-modes-offline-online.md#epic-dc-4--online-telepresence-mode)

- **ENV-34 – Document telepresence workflow** ✅ Completed

  `docs/dev-workflows/telepresence.md` captures prerequisites (Azure login + AKS access), connect/disconnect commands, and Tilt usage in remote mode.

### Epic ENV-4 – Bootstrap Config Updates

> **Note:** Maps to [432 DC-33](432-devcontainer-modes-offline-online.md#epic-dc-4--online-telepresence-mode) and [367 Phase 3](367-bootstrap-v2.md#phase-3-telepresence-integration-pending).

- **ENV-40 – Add bootstrap-telepresence.yaml** ✅ Completed

  `infra/environments/dev/bootstrap-telepresence.yaml` defines the AKS context, sets `gitops.skip: true`, and enables `telepresence.autoConnect` for the `ameide-dev` namespace.

- **ENV-41 – Update postCreate.sh for telepresence bootstrap** ✅ Completed
  - Add `BOOTSTRAP_TELEPRESENCE_CONFIG` variable pointing to `bootstrap-telepresence.yaml`
  - Update `run_bootstrap_for_mode()` online-telepresence case to:
    1. Load telepresence config
    2. Set kube context
    3. Optionally call telepresence connect
    4. Skip GitOps apply (remote cluster is Argo-managed)

- **ENV-42 – Create staging bootstrap config** ⏳ Not started

  Create `infra/environments/staging/bootstrap.yaml`:
  ```yaml
  env: staging
  cluster:
    context: ameide-staging
    type: aks
  gitops:
    root: gitops/ameide-gitops
    env: staging
    rootApplication: gitops/ameide-gitops/environments/staging/argocd/applications/ameide.yaml
  bicepOutputsFile: artifacts/bicep-outputs/staging.json
  secrets:
    repo:
      # Use workload identity or Key Vault for AKS
      strategy: workload-identity
  ```

  See [367 Phase 2](367-bootstrap-v2.md#phase-2-remote-cluster-support-in-progress) for staging bootstrap requirements.

## Dependencies

| Dependency | Status | Notes |
|------------|--------|-------|
| [432](432-devcontainer-modes-offline-online.md) – Devcontainer modes | ✅ Complete | Provides mode framework |
| [367](367-bootstrap-v2.md) – Bootstrap v2 | ✅ Phase 1 complete | Local dev working |
| Azure subscription access for AKS | ✅ Available | Both clusters accessible |
| Remote GitOps migration | ⏳ Pending | Blocker for telepresence |

## Success Criteria

1. ✅ `kubectl config get-contexts` shows `ameide-dev` (remote dev context in kubeconfig)
2. ⏳ `kubectl config get-contexts` shows the `ameide-staging` context (same cluster, staging namespace) after migration
3. ⏳ `telepresence connect` successfully connects to staging cluster
4. ⏳ Tilt can deploy services to remote cluster via telepresence
5. ✅ Domain naming is consistent (`local.ameide.io` for local, `*.ameide.io` for remote)
6. ⏳ AKS cluster is fully reproducible via `bootstrap-v2.sh --config infra/environments/staging/bootstrap.yaml`

## Implementation Order

1. **First**: Complete [Remote GitOps Migration](#migration-needed) (MIGRATE-1 through MIGRATE-7)
2. **Second**: [ENV-10](#epic-env-1--environment-naming-standardization) domain rename (`tilt.ameide.io` → `local.ameide.io`)
3. **Third**: [ENV-3](#epic-env-3--telepresence-implementation) telepresence implementation
4. **Fourth**: [ENV-4](#epic-env-4--bootstrap-config-updates) bootstrap configs for staging/telepresence
