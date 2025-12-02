# 435 - Remote-First Development Architecture

**Status:** Draft
**Created:** 2024-12-02
**Supersedes:** 432 (DevContainer Modes), parts of 367 (Bootstrap v2), parts of 429 (DevContainer Bootstrap)

## Overview

Pivot from dual-mode (offline-full k3d + online-telepresence) to a single remote-first development model where:

1. **ameide-gitops** owns all infrastructure-as-code (Argo CD, Helm, environments, bootstrap)
2. **ameide-core** owns only first-party application code + Tilt + Telepresence tooling

This eliminates the complexity of maintaining local k3d clusters while providing faster, more consistent development against a shared remote cluster.

## Goals

- **Single cluster strategy** - All development happens against shared AKS dev cluster
- **Simpler DevContainer** - Just `telepresence connect` + `tilt up`
- **GitOps purity** - Infrastructure changes go through ameide-gitops PRs
- **Faster dev loop** - No 15+ minute local bootstrap, instant cluster access
- **Reduced divergence** - No more k3d vs AKS configuration drift

## Non-Goals

- Offline development capability (accept network dependency)
- Per-developer isolated clusters (share dev namespace with developer-specific deployments)

---

## Architecture

### Target State

```
┌─────────────────────────────────────────────────────────────────────┐
│ ameide-gitops (infrastructure)                                       │
├─────────────────────────────────────────────────────────────────────┤
│ environments/                                                        │
│   ├── _shared/components/    # Shared component definitions          │
│   ├── dev/argocd/            # Dev cluster Argo CD config            │
│   ├── staging/argocd/        # Staging cluster Argo CD config        │
│   └── prod/argocd/           # Production cluster Argo CD config     │
│ sources/                                                             │
│   ├── charts/                # All Helm charts (apps + third-party)  │
│   └── values/                # Environment-specific values           │
│ bootstrap/                   # Cluster bootstrap scripts (NEW)       │
│   ├── bootstrap.sh           # Main bootstrap CLI                    │
│   ├── lib/                   # Library modules                       │
│   └── configs/               # Per-environment bootstrap configs     │
│ bicep/                       # Azure infrastructure (MOVE from core) │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ ameide-core (application code)                                       │
├─────────────────────────────────────────────────────────────────────┤
│ services/                    # First-party service source code       │
│ packages/                    # Shared packages                       │
│ .devcontainer/               # DevContainer (simplified)             │
│   ├── devcontainer.json                                              │
│   ├── Dockerfile                                                     │
│   └── postCreate.sh          # Just: telepresence connect            │
│ Tiltfile                     # Tilt for local iteration              │
│ tilt/                        # Tilt helpers                          │
│ tools/dev/                   # Developer tools                       │
│   └── telepresence.sh        # Telepresence helper                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Developer Workflow

```
1. Open DevContainer
   └─> postCreate.sh runs
       └─> az login + kubelogin (AKS auth)
       └─> telepresence connect --context ameide-dev-aks --namespace ameide
           └─> Developer now has DNS/network access to cluster services

2. Developer runs: tilt up -- --only=apps-www-ameide-platform-tilt
   └─> Tilt builds local service image
   └─> Tilt pushes to ACR/GHCR dev registry
   └─> Tilt deploys `apps-www-ameide-platform-tilt` Helm release (separate from Argo's `apps-www-ameide-platform`)
   └─> Live reload on file changes (sub-second for Go/Node/Python)

3. Developer tests against real cluster services
   └─> Keycloak, Postgres, Redis, Kafka, etc. are live in cluster (Argo-managed)
   └─> Tilt services run alongside Argo baseline with different hosts:
       - Argo: https://platform.dev.ameide.io (baseline)
       - Tilt: https://platform.local.ameide.io (inner-loop)
```

### Tilt + Argo Coexistence (from 373/424)

The key insight from backlogs 373 and 424 is **isolation, not adoption**:

| Component | Owner | Release Name | Host |
|-----------|-------|--------------|------|
| Baseline www-platform | Argo CD | `apps-www-ameide-platform` | `platform.dev.ameide.io` |
| Inner-loop www-platform | Tilt | `apps-www-ameide-platform-tilt` | `platform.local.ameide.io` |
| Keycloak | Argo CD (shared) | `platform-keycloak` | `auth.dev.ameide.io` |
| Databases | Argo CD | Various | Internal DNS |

This means:
- **No ownership conflicts** - Tilt and Argo never touch the same Kubernetes resources
- **No pause/resume dance** - Argo auto-syncs baseline; Tilt deploys separately
- **Shared infrastructure** - Tilt `-tilt` releases reuse Argo-managed secrets/services

---

## Migration Plan

### Phase 1: Move Infrastructure to GitOps Repo

| Task | Description | Owner |
|------|-------------|-------|
| **INFRA-10** | Move `infra/kubernetes/` contents to gitops repo | - |
| **INFRA-11** | Move `tools/bootstrap/` to gitops `bootstrap/` | - |
| **INFRA-12** | Move `infra/environments/` configs to gitops `bootstrap/configs/` | - |
| **INFRA-13** | Move `infra/bicep/` to gitops `bicep/` | - |
| **INFRA-14** | Move `infra/azure/` to gitops `azure/` | - |
| **INFRA-15** | Update gitops repo CI/CD for bootstrap validation | - |

### Phase 2: Simplify DevContainer

| Task | Description | Owner |
|------|-------------|-------|
| **DC-10** | Remove k3d cluster creation from postCreate.sh | - |
| **DC-11** | Remove ArgoCD bootstrap from postCreate.sh | - |
| **DC-12** | Remove offline-full mode support | - |
| **DC-13** | Simplify to: install deps + telepresence connect | - |
| **DC-14** | Update devcontainer.json (remove k3d features) | - |
| **DC-15** | Add AKS credential helper (kubelogin) | - |

### Phase 3: Adapt Tiltfile for Remote Cluster

The Tilt Helm architecture from 373/424 is **preserved** - it's essential for fast inner loop:
- Argo owns `apps-*` (GitOps baseline)
- Tilt owns `apps-*-tilt` (separate Helm releases, sub-second live updates)

| Task | Description | Owner |
|------|-------------|-------|
| **TILT-10** | Update registry config for remote ACR/GHCR push | - |
| **TILT-11** | Update kubectl context defaults (k3d → AKS) | - |
| **TILT-12** | Keep all `apps-*-tilt` Helm releases (373/424 architecture) | - |
| **TILT-13** | Keep live-update configs for Go/Node/Python | - |
| **TILT-14** | Remove k3d-specific registry wiring (`k3d-ameide.localhost:5001`) | - |
| **TILT-15** | Update values paths if gitops structure changes | - |

### Phase 4: Cleanup Core Repo

| Task | Description | Owner |
|------|-------------|-------|
| **CLEAN-10** | Delete `infra/kubernetes/` directory | - |
| **CLEAN-11** | Delete `infra/argocd/` directory | - |
| **CLEAN-12** | Delete `infra/environments/` directory | - |
| **CLEAN-13** | Delete `infra/bicep/` directory | - |
| **CLEAN-14** | Delete `infra/azure/` directory | - |
| **CLEAN-15** | Delete `tools/bootstrap/` directory | - |
| **CLEAN-16** | Update `.devcontainer/lib/` (remove unused modules) | - |
| **CLEAN-17** | Archive/retire backlogs 367, 429, 432 | - |

### Phase 5: Documentation & Validation

| Task | Description | Owner |
|------|-------------|-------|
| **DOC-10** | Update `docs/dev-workflows/` for remote-first | - |
| **DOC-11** | Update main README.md | - |
| **DOC-12** | Create troubleshooting guide for telepresence | - |
| **DOC-13** | Document cluster access prerequisites (Azure AD) | - |
| **VAL-10** | Test fresh DevContainer startup | - |
| **VAL-11** | Test Tilt up against remote cluster | - |
| **VAL-12** | Test full dev cycle (edit → build → test) | - |

---

## Files to Move (Core → GitOps)

### `infra/kubernetes/` → `gitops/`

Already partially done. Remaining:
- `infra/kubernetes/charts_old/` → Archive or delete (superseded by `sources/charts/`)
- `infra/kubernetes/environments/` → Already in gitops as `sources/values/`
- `infra/kubernetes/scripts/` → `gitops/scripts/` (cluster health checks, etc.)
- `infra/kubernetes/values/` → Already in gitops as `sources/values/`
- `infra/kubernetes/policies/` → `gitops/policies/`

### `tools/bootstrap/` → `gitops/bootstrap/`

```
tools/bootstrap/bootstrap-v2.sh     → gitops/bootstrap/bootstrap.sh
tools/bootstrap/lib/*.sh            → gitops/bootstrap/lib/
tools/bootstrap/README.md           → gitops/bootstrap/README.md
```

### `infra/environments/` → `gitops/bootstrap/configs/`

```
infra/environments/dev/bootstrap.yaml              → gitops/bootstrap/configs/dev.yaml
infra/environments/dev/bootstrap-telepresence.yaml → DELETE (no longer needed)
infra/environments/staging/bootstrap.yaml          → gitops/bootstrap/configs/staging.yaml
```

### `infra/bicep/` → `gitops/bicep/`

```
infra/bicep/managed-application/    → gitops/bicep/managed-application/
```

### `infra/azure/` → `gitops/azure/`

```
infra/azure/local/                  → DELETE (k3d-specific)
infra/azure/managed-application/    → gitops/azure/managed-application/
```

---

## Files to Delete from Core

After migration:
```
infra/                              # Entire directory
tools/bootstrap/                    # Entire directory
.devcontainer/lib/k3d.sh           # k3d helpers
.devcontainer/lib/argocd.sh        # ArgoCD bootstrap (moved)
.devcontainer/lib/helm.sh          # Helm bootstrap (moved)
```

---

## Files to Simplify in Core

### `.devcontainer/postCreate.sh`

**Current:** ~400 lines handling offline-full, online-telepresence, k3d, ArgoCD, etc.

**Target:** ~50 lines
```bash
#!/bin/bash
set -euo pipefail

# Install dependencies
./install-deps.sh

# Configure kubectl for remote cluster
az aks get-credentials --resource-group Ameide-Dev --name ameide-dev-aks
kubelogin convert-kubeconfig -l azurecli

# Connect telepresence
telepresence connect --context ameide-dev-aks --namespace ameide

echo "Ready! Run 'tilt up' to start developing."
```

### `Tiltfile`

**Current:** 1840 lines with Helm resources, ArgoCD integration, k3d handling

**Target:** ~1200 lines (simplified, not gutted)

**Keep (essential for inner loop per 373/424):**
- All `apps-*-tilt` Helm releases (separate from Argo's `apps-*`)
- Live-update configs for Go/Node/Python (sub-second reload)
- Docker builds for first-party services
- SDK/package build resources
- Integration/E2E test orchestration

**Remove:**
- k3d-specific registry wiring (`k3d-ameide.localhost:5001`)
- `argocd-sync-guard` pause/resume logic (no Argo/Tilt ownership conflicts in remote)
- `tilt-helm-adopt.sh` calls (no shared ownership in remote model)
- k3d context detection and fallbacks

**Update:**
- Registry defaults → ACR or GHCR
- Context defaults → `ameide-dev-aks`
- Values paths → reference gitops submodule or remote repo

---

## Shared Dev Cluster Requirements

### Cluster: `ameide-dev-aks`

- **Resource Group:** `Ameide-Dev`
- **Region:** Same as staging for latency
- **Node Pool:** Cost-optimized (B-series, spot instances)
- **Namespaces:**
  - `ameide` - Shared services (Keycloak, databases, etc.)
  - `ameide-dev-{user}` - Per-developer deployments (optional isolation)

### Access Control

- Azure AD authentication via `kubelogin`
- RBAC: Developers get `edit` role in `ameide` namespace
- Telepresence: Traffic Manager deployed cluster-side

### ArgoCD

- Deployed via bootstrap in gitops repo
- Manages all shared services
- Developers' Tilt deployments are "unmanaged" (not in ArgoCD)

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Network dependency | Document offline limitations; provide read-only mode for docs/planning |
| Shared cluster conflicts | Namespace isolation; resource quotas; clear ownership |
| Slower image push | Use cluster-local registry; optimize Dockerfile layers |
| Azure auth complexity | Pre-configure kubelogin in DevContainer; document troubleshooting |
| Cost of always-on cluster | Auto-scaling; spot instances; shutdown schedule for nights/weekends |

---

## Success Criteria

1. Fresh DevContainer starts in <5 minutes (vs. 15-20 min with k3d bootstrap)
2. `tilt up` connects to cluster and deploys service in <2 minutes
3. Zero infrastructure code in ameide-core repo
4. Single source of truth for cluster config in ameide-gitops
5. Developer can go from clone → running service in <10 minutes

---

## Related Backlogs

### Preserved (Critical Architecture)
- **373** (Argo + Tilt + Helm North Star v4) - **KEEP**: Defines two-path architecture (Argo baseline + Tilt overlay)
- **424** (Tilt-only releases) - **KEEP**: `apps-*-tilt` isolation pattern, sub-second inner loop

### Superseded
- **367** (Bootstrap v2) - Superseded for local dev; bootstrap logic moves to gitops
- **429** (DevContainer Bootstrap) - Simplified; most content moves to gitops
- **432** (DevContainer Modes) - Retired; single mode (remote-telepresence)

### Still Relevant
- **434** (Unified Environment Naming) - Still relevant for gitops structure

---

## Open Questions

1. **Per-developer namespaces?** Should each developer get `ameide-dev-{user}` namespace for isolation, or share single `ameide` namespace?

2. **Image registry strategy?** Push to:
   - Cluster-local registry (fastest, ephemeral)
   - GHCR with dev tags (persistent, shareable)
   - ACR dev registry (Azure-native)

3. **Telepresence vs. Tilt remote?**
   - Telepresence: Network-level access, intercepts
   - Tilt remote: First-class Tilt support, may not need telepresence

4. **Offline escape hatch?** Keep minimal docker-compose for truly offline scenarios?
