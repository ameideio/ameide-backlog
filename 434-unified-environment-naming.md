# 434 – Unified Environment Naming and Telepresence Wiring

> **Note:** This document remains relevant for environment naming conventions.
> The telepresence wiring and remote development workflow has been further refined in
> [435-remote-first-development.md](435-remote-first-development.md).

> **Related documents:**
> - [443-tenancy-models.md](443-tenancy-models.md) – **Tenancy model overview (single source of truth)**
> - [435-remote-first-development.md](435-remote-first-development.md) – Remote-first development architecture
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

## Tenancy & Environment Matrix

In addition to dev/staging/production environments, AMEIDE supports three tenancy models. See [443-tenancy-models.md](443-tenancy-models.md) for comprehensive documentation.

| Tenancy Model | Cluster Pattern | Namespace Pattern | Example Domains |
|---------------|-----------------|-------------------|-----------------|
| Shared SaaS | Single shared cluster `ameide` | `ameide-prod` | `*.ameide.io` |
| Namespace-per-tenant | Single shared cluster per region | `tenant-<slug>-prod` (per tenant) | `<slug>.ameide.io`, `api.<slug>.ameide.io` |
| Private cloud (per-tenant) | Dedicated cluster `ameide-<slug>` per tenant | `ameide-dev/staging/prod` inside tenant cluster | `*.customer-domain.tld` or `*.ameide-<slug>.io` |

Tenancy is **orthogonal** to environment:
- We always keep `ameide-dev`, `ameide-staging`, `ameide-prod` as internal environments.
- For namespace-per-tenant and private cloud, each tenant gets its own "production" environment running the same components.

### Shared AKS Cluster

- **Resource group:** `Ameide`
- **Cluster name:** `ameide`
- **Kube contexts:** `ameide-dev`, `ameide-staging`, `ameide-prod` (all reference the same API server, differing only by namespace/user bindings)
- **Namespaces:** `ameide-dev`, `ameide-staging`, `ameide-prod`, plus `argocd` and platform shared namespaces

### Namespace & Pod Labels

All workloads MUST be labeled consistently to support environment and tenant isolation (see [441-networking.md](441-networking.md), [442-environment-isolation.md](442-environment-isolation.md)):

| Label | Values | Description |
|-------|--------|-------------|
| `ameide.io/environment` | `dev`, `staging`, `production`, `cluster` | SDLC stage |
| `ameide.io/tenant-kind` | `shared`, `namespace`, `private` | Tenancy model |
| `ameide.io/tenant-id` | `<tenant-slug>` (e.g., `acme`, `contoso`) | Tenant identifier |
| `ameide.io/tier` | `frontend`, `backend`, `data`, `platform`, `foundation` | Service tier |

**Examples:**

Shared SaaS `ameide-prod` namespace:
```yaml
metadata:
  name: ameide-prod
  labels:
    ameide.io/environment: production
    ameide.io/tenant-kind: shared
```

Namespace-per-tenant `tenant-acme-prod` namespace:
```yaml
metadata:
  name: tenant-acme-prod
  labels:
    ameide.io/environment: production
    ameide.io/tenant-kind: namespace
    ameide.io/tenant-id: acme
```

Backend pod labels:
```yaml
metadata:
  labels:
    ameide.io/environment: production
    ameide.io/tenant-kind: namespace
    ameide.io/tenant-id: acme
    ameide.io/tier: backend
```

> **Reference:** See [443-tenancy-models.md](443-tenancy-models.md) for complete label documentation.

### Control plane & image ownership

- **Argo CD:** A single Argo instance in the `argocd` namespace reconciles every environment namespace using a **single parametrized ApplicationSet**. There are no per-namespace Argo installations or per-environment ApplicationSet files; isolation is enforced by Git (Projects) and RBAC, not by multiple controllers.
- **GitHub Container Registry (GHCR):** `ameide-gitops` defines the repository/tag per environment. CI pushes images to GHCR, updates the GitOps values, and Argo syncs. No doc/tooling should reference Azure Container Registry directly.
- **Tilt-only releases:** Developers still run Tilt + Telepresence for the inner loop. Those resources carry the `-tilt` suffix and point at developer-scoped tags in GHCR but never alter the Argo-managed baselines.

### GitOps Repository Structure

> **Updated 2025-12-11**: Refactored to use Git file generators for cluster config and 6-layer values.
> See [config/README.md](../config/README.md) and [sources/values/README.md](../sources/values/README.md).

```
ameide-gitops/
├── argocd/                              # Cluster-level ArgoCD config
│   ├── applicationsets/
│   │   ├── ameide.yaml                  # Environment-scoped (env × component)
│   │   └── cluster.yaml                 # Cluster-scoped (operators, CRDs)
│   ├── base/                            # Kustomize base
│   │   └── kustomization.yaml
│   ├── overlays/                        # Cluster-specific overlays
│   │   ├── azure/                       # AKS: config/clusters/azure.yaml
│   │   └── local/                       # k3d: config/clusters/local.yaml
│   └── projects/
│
├── config/                              # Data-driven cluster configuration
│   ├── clusters/                        # One file per cluster
│   │   ├── azure.yaml                   # dev, staging, production envs
│   │   └── local.yaml                   # local env only
│   └── node-profiles/                   # Scheduling constraints
│       ├── dev-pool.yaml                # nodeSelector + tolerations
│       ├── staging-pool.yaml
│       ├── prod-pool.yaml
│       └── none.yaml                    # Empty (for k3d)
│
├── environments/
│   └── _shared/
│       └── components/                  # Shared component definitions
│           ├── apps/
│           ├── data/
│           ├── foundation/
│           ├── observability/
│           ├── platform/
│           └── cluster/                 # Cluster-scoped components
│
└── sources/
    ├── charts/                          # Helm charts
    └── values/                          # 6-layer values architecture
        ├── base/                        # Layer 1: Cluster-agnostic defaults
        │   └── globals.yaml
        ├── cluster/                     # Layer 2: Cluster-specific
        │   ├── azure/globals.yaml       # Azure tenant, storage class
        │   └── local/globals.yaml       # k3d config
        ├── env/                         # Layer 3: Environment-specific
        │   ├── dev/globals.yaml
        │   ├── staging/globals.yaml
        │   ├── production/globals.yaml
        │   └── local/globals.yaml
        ├── _shared/                     # Layer 5: Component defaults
        └── {env}/                       # Layer 6: Env-specific overrides
```

> **Note**: Layer 4 (node profiles) is in `config/node-profiles/` to keep scheduling constraints out of environment values.

### Single Parametrized ApplicationSet

> **Updated 2025-12-11**: Now uses Git file generator to read cluster config from YAML files.

The ApplicationSet uses a **matrix generator** with a Git file generator (not a list):

```yaml
# argocd/applicationsets/ameide.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ameide
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          # Git generator reads cluster config (with environments array)
          - git:
              repoURL: https://github.com/ameideio/ameide-gitops.git
              revision: main
              files:
                - path: config/clusters/azure.yaml  # Patched by overlay for local
          # Component definitions
          - git:
              repoURL: https://github.com/ameideio/ameide-gitops.git
              revision: main
              files:
                - path: environments/_shared/components/apps/**/component.yaml
                - path: environments/_shared/components/data/**/component.yaml
                # ... etc
  template:
    metadata:
      name: '{{ .env }}-{{ .name }}'
    spec:
      destination:
        namespace: '{{ .namespace }}'  # From cluster config
      sources:
        - helm:
            valueFiles:
              # 6-layer values architecture
              - '$values/sources/values/base/globals.yaml'
              - '$values/sources/values/cluster/{{ .clusterType }}/globals.yaml'
              - '$values/sources/values/env/{{ .env }}/globals.yaml'
              - '$values/config/node-profiles/{{ .nodeProfile }}.yaml'
              - '$values/sources/values/_shared/{{ .domain }}/{{ .name }}.yaml'
              - '$values/sources/values/env/{{ .env }}/{{ .domain }}/{{ .name }}.yaml'
```

**Cluster config file** (`config/clusters/azure.yaml`):
```yaml
- cluster: aks-main
  clusterType: azure
  env: dev
  namespace: ameide-dev
  nodeProfile: dev-pool
  domain: dev.ameide.io
# ... staging, production
```

**Benefits:**
- **Clusters are data** – Add new cluster by adding a YAML file
- **Environments per cluster** – Each cluster defines its own environments
- **Node profiles separate** – Scheduling constraints in dedicated files
- **6-layer values** – Clean separation: base → cluster → env → nodeProfile → shared → env-specific
- **Kustomize overlays** – Switch cluster by patching the file path

#### ApplicationSet – Tenant-aware Generator (Future)

For namespace-per-tenant deployments, extend the `list` generator with tenants:

```yaml
generators:
  - matrix:
      generators:
        - list:
            elements:
              # Shared SaaS
              - env: production
                tenantKind: shared
                tenantId: shared
                namespace: ameide-prod
              # Namespace-per-tenant
              - env: production
                tenantKind: namespace
                tenantId: acme
                namespace: tenant-acme-prod
              - env: production
                tenantKind: namespace
                tenantId: contoso
                namespace: tenant-contoso-prod
        - git:
            files:
              - path: environments/_shared/components/**/component.yaml
```

Tenants only add new `list.elements` and per-tenant values under `sources/values/tenants/<tenant-id>/...`.

> **Scaling note**: For >20 tenants, consider replacing the `list` generator with a
> `git` generator that reads tenant definitions from `sources/values/tenants/*/tenant.yaml` files.

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
| **MIGRATE-1** | Delete legacy GitOps path (`ameide.git/infra/kubernetes/gitops/`) | ✅ Complete |
| **MIGRATE-2** | Point Argo CD at `ameide-gitops` repo with `argocd/` path | ✅ Complete |
| **MIGRATE-3** | Create `argocd/` structure with single parametrized ApplicationSet | ✅ Complete |
| **MIGRATE-4** | Create `bootstrap/configs/` for each environment | ✅ Complete |
| **MIGRATE-5** | Namespace split: `ameide` → `ameide-dev`, `ameide-staging`, `ameide-prod` within the single `ameide` cluster | ✅ Complete |
| **MIGRATE-6** | Create kube contexts/users `ameide-{env}` that target the shared cluster but default to the matching namespace | ✅ Complete *(automation documented in [491-auto-contexts.md](491-auto-contexts.md))* |
| **MIGRATE-7** | Consolidate ApplicationSets into single parametrized file | ✅ Complete |
| **MIGRATE-8** | Verify cluster is fully reproducible via bootstrap-v2 | ⏳ Pending |
| **MIGRATE-9** | Update service URLs in values files to use `{{ .Values.namespace }}` | ✅ Complete |
| **MIGRATE-10** | Remove hardcoded `namespace: ameide` from shared values; use `{{ .Release.Namespace }}` | ✅ Complete |

**Approach:** The legacy GitOps content should be removed entirely (not migrated). The AKS cluster should be bootstrapped fresh using `bootstrap-v2.sh` once the `argocd/` structure exists in `ameide-gitops`.

### Migration Status Update (2025-12-04)

> **Note:** This section tracks current implementation status.

| Task | Current Status | Notes |
|------|----------------|-------|
| **MIGRATE-1** | ⏳ Blocked | Legacy path still in use on cluster |
| **MIGRATE-2** | ⏳ Blocked | Requires MIGRATE-1 |
| **MIGRATE-3** | ✅ Complete | `argocd/applicationsets/ameide.yaml` created |
| **MIGRATE-4** | ✅ Complete | `bootstrap/configs/` exists |
| **MIGRATE-5** | ✅ Complete | Namespaces `ameide-dev/staging/prod` exist on cluster |
| **MIGRATE-6** | ✅ Complete | Contexts configured automatically via `.devcontainer/postCreate.sh` (calls `tools/dev/bootstrap-contexts.sh`) |
| **MIGRATE-7** | ✅ Complete | Single parametrized ApplicationSet in `argocd/applicationsets/ameide.yaml` |
| **MIGRATE-8** | ⏳ Blocked | Requires MIGRATE-1 & MIGRATE-2 |
| **MIGRATE-9** | ✅ Complete | Service URLs updated to use short names (Kubernetes DNS resolves within namespace) |
| **MIGRATE-10** | ✅ Complete | Hardcoded `namespace: ameide` removed from ~50 files; templates use `{{ .Release.Namespace }}` or `{{ .Values.namespace }}` |

**Current Blocker:** AKS cluster still points at legacy GitOps path (`ameide.git/infra/kubernetes/gitops/`).

**Repository Refactoring (Complete):**
- [x] Create `argocd/applicationsets/ameide.yaml` (single parametrized ApplicationSet)
- [x] Create `argocd/projects/` (shared project definitions)
- [x] Create `argocd/kustomization.yaml`
- [x] Add `namespace` field to `globals.yaml` for each environment
- [x] Update service URLs to use short names (same-namespace DNS resolution)
- [x] Remove `environments/*/argocd/` directories (replaced by `argocd/`)
- [x] Remove hardcoded `namespace: ameide` from all shared values files
- [x] Update ExternalSecret manifests to use `{{ .Release.Namespace }}`
- [x] Update CNPG postgres cluster to use dynamic namespace (removed `namespaceOverride`)
- [x] Add tier labels (`tier`, `domain`, `exposure`) to all app values files for NetworkPolicy support
- [x] Update chart defaults to use `.Release.Namespace` (gateway, redis, kafka, clickhouse, etc.)
- [x] Update environment gateway configs to use `{{ .Values.namespace }}`
- [x] Update observability smoke tests to use `{{ .Release.Namespace }}`
- [x] Update cert-manager issuer templates to use dynamic gateway namespace
- [x] Update Grafana datasources to use dynamic service URLs

**Remaining Operational Steps:**
- [ ] Point ArgoCD on AKS cluster at `ameide-gitops` repo with `argocd/` path
- [ ] Delete legacy GitOps path from `ameide.git`
- [ ] Verify full cluster sync with new structure

## Backlog

### Epic ENV-1 – Environment Naming Standardization

- **ENV-10 – Update local domain from tilt.ameide.io to local.ameide.io** ✅ Completed

  Goal: Consistent naming across all environments.

  **Updated components:**
  | File | Change |
  |------|--------|
| *(legacy)* `.devcontainer/manage-host-domains.sh` | removed in the remote-first workflow; Telepresence handles DNS routing |
  | `gitops/ameide-gitops/sources/values/env/local/apps/platform/gateway.yaml` | `https-local` listener + `auth.local.ameide.io` route |
  | `gitops/ameide-gitops/sources/values/env/dev/platform/platform-cert-manager-config.yaml` | Wildcard SANs for `*.local.ameide.io` |
  | `gitops/ameide-gitops/sources/values/env/dev/apps/apps-www-ameide(-platform)-tilt.yaml` | Hostnames + cookie domains moved to `local.ameide.io` |
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
> The ongoing operational backlog (connectivity, intercept verification, troubleshooting) lives in [492-telepresence.md](492-telepresence.md).
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

---

## Appendix: Certificate & Gateway Architecture

### Domain and Certificate Matrix

| Environment | Domain | DNS Zone | Issuer | Certificate |
|-------------|--------|----------|--------|-------------|
| dev | `*.dev.ameide.io` | `dev.ameide.io` | `letsencrypt-dev-dns01` | `ameide-wildcard-tls` |
| staging | `*.staging.ameide.io` | `staging.ameide.io` | `letsencrypt-staging-dns01` | `ameide-wildcard-tls` |
| production | `*.ameide.io` | `ameide.io` | `letsencrypt-prod-dns01` | `ameide-wildcard-tls` |
| ArgoCD (cluster) | `argocd.ameide.io` | `ameide.io` | `letsencrypt-apex-dns01` | `argocd-ameide-io-tls` |

**Key decisions:**
- Production uses apex domain (`ameide.io`) directly — there is NO `prod.ameide.io`
- ArgoCD is cluster-level at `argocd.ameide.io`, not tied to any environment
- Each environment has its own DNS zone and ClusterIssuer

### Bicep → Helm Data Flow

Infrastructure values flow from Bicep to Helm via git:

```
┌─────────────────────────────────────────────────────────────────┐
│ Bicep deployment                                                 │
│   bicep/main.bicep → az deployment group create                 │
│                                                                  │
│ Outputs:                                                         │
│   - envoyPublicIpAddress (static IP for gateway)                │
│   - dnsManagedIdentityClientId (for cert-manager DNS-01)        │
│   - keyVaultUri, vaultBootstrapIdentityClientId                 │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ scripts/update-globals-from-bicep.sh <env>                      │
│                                                                  │
│ Reads: artifacts/bicep-outputs/<env>.json                       │
│ Updates: sources/values/<env>/globals.yaml                      │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ Git commit + push                                                │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ ArgoCD syncs                                                     │
│                                                                  │
│ Helm templates reference:                                        │
│   {{ .Values.azure.envoyPublicIpAddress }}                      │
│   {{ .Values.azure.dnsManagedIdentityClientId }}                │
│                                                                  │
│ globals.yaml auto-included via ApplicationSet templatePatch     │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration Files

| Purpose | File Path |
|---------|-----------|
| **ArgoCD (cluster-level)** | |
| ApplicationSet (all envs) | `argocd/applicationsets/ameide.yaml` |
| Projects | `argocd/projects/*.yaml` |
| **Environment globals** | |
| Dev globals | `sources/values/env/dev/globals.yaml` |
| Staging globals | `sources/values/env/staging/globals.yaml` |
| Production globals | `sources/values/env/production/globals.yaml` |
| **Infrastructure** | |
| Bicep outputs sync | `scripts/update-globals-from-bicep.sh` |
| Dev cert-manager | `sources/values/env/dev/platform/platform-cert-manager-config.yaml` |
| Staging cert-manager | `sources/values/env/staging/platform/platform-cert-manager-config.yaml` |
| Production cert-manager | `sources/values/env/production/platform/platform-cert-manager-config.yaml` |
| Gateway config (dev) | `sources/values/env/dev/platform/platform-gateway.yaml` |
| **Components** | |
| Shared components | `environments/_shared/components/` |

### Vendor Documentation References

- **AKS LoadBalancer static IP**: [Azure docs](https://learn.microsoft.com/en-us/azure/aks/static-ip) — Bicep creates IP, Service uses `loadBalancerIP`
- **cert-manager Azure DNS**: [cert-manager docs](https://cert-manager.io/docs/configuration/acme/dns01/azuredns/) — per-zone ClusterIssuers
- **ArgoCD multi-tenant**: [ArgoCD docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/multi-tenancy/) — single instance, Project isolation
- **Envoy Gateway**: [Gateway API](https://gateway-api.sigs.k8s.io/) — host-based routing via HTTPRoute

---

## Cross-References

| Backlog | Relationship |
|---------|--------------|
| [430-unified-test-infrastructure](./430-unified-test-infrastructure.md) | Test modes aligned with environment naming |
| [435-remote-first-development](./435-remote-first-development.md) | Telepresence + AKS workflow (refines this backlog) |
| [443-tenancy-models](./443-tenancy-models.md) | Tenancy model definitions (single source of truth) |
| [484-ameide-cli](./484-ameide-cli.md) | CLI `scaffold` outputs GitOps manifests following 434 structure |
