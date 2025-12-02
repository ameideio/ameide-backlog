# 435 - Remote-First Development Architecture

**Status:** In Progress
**Created:** 2024-12-02
**Supersedes:** 432 (DevContainer Modes), parts of 367 (Bootstrap v2), parts of 429 (DevContainer Bootstrap)

## Overview

Pivot from dual-mode (offline-full k3d + online-telepresence) to a single **AKS + Telepresence + Tilt** model:

1. **ameide-gitops** owns all infrastructure-as-code (Argo CD, Helm, environments, bootstrap)
2. **ameide-core** owns first-party application code + Tilt + Telepresence tooling
3. **No local k3d cluster** - AKS dev cluster is the target

## Goals

- **Single cluster strategy** - All development happens against shared AKS dev cluster
- **Simpler DevContainer** - Just `telepresence connect` + `tilt up`
- **GitOps purity** - Infrastructure changes go through ameide-gitops PRs
- **Faster inner loop** - Telepresence intercepts bypass image build/push/deploy cycle
- **Reduced divergence** - No more k3d vs AKS configuration drift

## Non-Goals

- Offline development capability (accept network dependency)
- Per-developer isolated clusters (share dev namespace with developer-specific deployments)
- Local k3d clusters (fully remote-first)

---

## Architecture

### How It Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AKS Dev Cluster                                 │
│                                                                              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │
│  │  ArgoCD         │    │  Traffic        │    │  Shared         │          │
│  │  (GitOps)       │    │  Manager        │    │  Services       │          │
│  │                 │    │  (Telepresence) │    │  (Keycloak,     │          │
│  │  apps-*         │    │                 │    │   Postgres,     │          │
│  │  (baseline)     │    │                 │    │   Redis, etc)   │          │
│  └─────────────────┘    └────────┬────────┘    └─────────────────┘          │
│                                  │                                           │
│  ┌─────────────────┐             │                                           │
│  │  apps-*-tilt    │             │  Intercept                                │
│  │  (Tilt-owned)   │◄────────────┼──────────────────────┐                    │
│  │                 │             │                      │                    │
│  └─────────────────┘             │                      │                    │
└──────────────────────────────────┼──────────────────────┼────────────────────┘
                                   │                      │
                    ┌──────────────┴──────────────────────┴───────────────┐
                    │                  DevContainer                        │
                    │                                                      │
                    │  ┌────────────────┐    ┌────────────────────────┐   │
                    │  │ Telepresence   │    │ Tilt                   │   │
                    │  │ CLI            │    │                        │   │
                    │  │                │    │ - docker_build → ACR   │   │
                    │  │ - connect      │    │ - k8s_yaml → AKS       │   │
                    │  │ - intercept    │    │ - local_resource       │   │
                    │  └────────────────┘    └────────────────────────┘   │
                    │                                                      │
                    │  ┌────────────────────────────────────────────────┐ │
                    │  │ Local Process (intercepted service)            │ │
                    │  │ npm run dev / go run / uvicorn                 │ │
                    │  │ ← Traffic from AKS routed here                 │ │
                    │  └────────────────────────────────────────────────┘ │
                    └──────────────────────────────────────────────────────┘
```

### Key Insight: Telepresence vs Tilt Roles

| Tool | Role | Builds Images? |
|------|------|----------------|
| **Telepresence** | Network tunnel + traffic intercepts | ❌ No |
| **Tilt** | Image builds, deployments, local resources | ✅ Yes |
| **ArgoCD** | GitOps baseline deployments | ❌ No (CI builds) |

**Telepresence** is purely networking:
- `telepresence connect` → VPN-like tunnel to cluster (DNS, service discovery)
- `telepresence intercept` → Route traffic from cluster pod to local process

**Tilt** handles builds and deployments:
- `docker_build` → Build image locally
- Push to ACR (via `default_registry`)
- Apply manifests to AKS
- `local_resource` → Run local processes (including intercepted services)

### Developer Workflow Options

#### Option A: Full Tilt (image build per change)

```bash
# 1. Connect to cluster
telepresence connect --namespace ameide

# 2. Run Tilt - builds images, pushes to ACR, deploys to AKS
tilt up -- --only=apps-www-ameide-platform-tilt
```

- Every code change → image rebuild → push to ACR → pod restart
- Slower but tests full container environment
- Good for: Dockerfile changes, dependency updates, final testing

#### Option B: Telepresence Intercept (no image rebuild)

```bash
# 1. Connect to cluster
telepresence connect --namespace ameide

# 2. Intercept the service
telepresence intercept www-ameide-platform --port 3000:3000

# 3. Run locally (traffic from AKS → your laptop)
npm run dev
```

- Code changes → instant (no container rebuild)
- Traffic from cluster routed to local process
- Good for: Rapid iteration, debugging, hot reload

#### Option C: Hybrid (Tilt orchestrates Telepresence)

```python
# Tiltfile
local_resource(
    'www-platform-dev',
    serve_cmd='npm run dev',
    deps=['services/www_ameide_platform/'],
    resource_deps=['telepresence-intercept-platform']
)

local_resource(
    'telepresence-intercept-platform',
    cmd='telepresence intercept www-ameide-platform --port 3000:3000',
    resource_deps=['telepresence-connect']
)
```

- Tilt manages the intercept lifecycle
- Best of both worlds: Tilt UI + Telepresence speed

### Tilt + Argo Coexistence (from 373/424)

The **isolation pattern** from backlogs 373 and 424 is preserved:

| Component | Owner | Release Name | Purpose |
|-----------|-------|--------------|---------|
| Baseline www-platform | ArgoCD | `apps-www-ameide-platform` | GitOps-managed baseline |
| Inner-loop www-platform | Tilt | `apps-www-ameide-platform-tilt` | Developer iteration |
| Shared services | ArgoCD | Various | Keycloak, databases, etc. |

**Why this matters:**
- No ownership conflicts (Tilt and Argo never touch same resources)
- No pause/resume dance with ArgoCD
- Tilt `-tilt` releases can have different configs (debug mode, local volumes, etc.)

---

## Repository Structure

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
│ bootstrap/                   # Cluster bootstrap scripts             │
│   ├── bootstrap.sh           # Main bootstrap CLI                    │
│   ├── lib/                   # Library modules                       │
│   └── configs/               # Per-environment bootstrap configs     │
│ bicep/                       # Azure infrastructure                  │
│ backlog/                     # Submodule → ameide-backlog            │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ ameide-core (application code)                                       │
├─────────────────────────────────────────────────────────────────────┤
│ services/                    # First-party service source code       │
│ packages/                    # Shared packages                       │
│ .devcontainer/               # DevContainer (simplified)             │
│   ├── devcontainer.json                                              │
│   ├── Dockerfile                                                     │
│   └── postCreate.sh          # telepresence connect + setup          │
│ Tiltfile                     # Tilt for local iteration              │
│ tilt/                        # Tilt helpers                          │
│ tools/dev/                   # Developer tools                       │
│   └── telepresence.sh        # Telepresence helper                   │
│ gitops/ameide-gitops/        # Submodule → ameide-gitops             │
│ backlog/                     # Submodule → ameide-backlog            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Migration Plan

### Phase 1: Move Infrastructure to GitOps Repo ✅ COMPLETE

| Task | Description | Status |
|------|-------------|--------|
| **INFRA-10** | Move `infra/kubernetes/scripts/` and `policies/` to gitops | ✅ |
| **INFRA-11** | Move `tools/bootstrap/` to gitops `bootstrap/` | ✅ |
| **INFRA-12** | Move `infra/environments/` configs to gitops `bootstrap/configs/` | ✅ |
| **INFRA-13** | Move `infra/bicep/` to gitops `bicep/` | ✅ |
| **INFRA-14** | Move `infra/azure/managed-application/` to gitops `azure/` | ✅ |
| **INFRA-15** | Update READMEs | ✅ |

### Phase 2: Simplify DevContainer (Remove k3d)

| Task | Description | Owner |
|------|-------------|-------|
| **DC-10** | Remove k3d cluster creation from postCreate.sh | - |
| **DC-11** | Remove k3d feature from devcontainer.json | - |
| **DC-12** | Remove ArgoCD bootstrap logic (Argo lives in AKS, managed by gitops) | - |
| **DC-13** | Add AKS credential setup (az login + kubelogin) | - |
| **DC-14** | Add Telepresence connect to postCreate.sh | - |
| **DC-15** | Install Telepresence CLI in Dockerfile | - |
| **DC-16** | Remove `.devcontainer/lib/k3d.sh` and related modules | - |

**Target postCreate.sh (~50 lines):**
```bash
#!/bin/bash
set -euo pipefail

# Install dependencies
pnpm install

# Configure kubectl for AKS
az aks get-credentials --resource-group Ameide-Dev --name ameide-dev-aks
kubelogin convert-kubeconfig -l azurecli

# Connect telepresence to cluster
telepresence connect --namespace ameide

echo "✅ Ready! Run 'tilt up' to start developing."
```

### Phase 3: Adapt Tiltfile for Remote Cluster

| Task | Description | Owner |
|------|-------------|-------|
| **TILT-10** | Add `default_registry('ameidedev.azurecr.io')` for ACR | - |
| **TILT-11** | Update kubectl context defaults (remove k3d references) | - |
| **TILT-12** | Keep all `apps-*-tilt` Helm releases (373/424 architecture) | - |
| **TILT-13** | Add `local_resource` for Telepresence intercepts | - |
| **TILT-14** | Remove k3d-specific registry wiring (`k3d-ameide.localhost:5001`) | - |
| **TILT-15** | Update values paths to reference gitops submodule | - |
| **TILT-16** | Add Telepresence intercept helpers to `tilt/` | - |

**Example Tiltfile additions:**
```python
# Registry for AKS
default_registry('ameidedev.azurecr.io')

# Telepresence intercept mode (optional, for rapid iteration)
if config.tilt_subcommand == 'up' and os.getenv('TELEPRESENCE_INTERCEPT'):
    local_resource(
        'www-platform-intercept',
        serve_cmd='cd services/www_ameide_platform && npm run dev',
        deps=['services/www_ameide_platform/'],
        labels=['intercept'],
    )
```

### Phase 4: Cleanup Core Repo

| Task | Description | Owner |
|------|-------------|-------|
| **CLEAN-10** | Delete `infra/kubernetes/` directory | - |
| **CLEAN-11** | Delete `infra/argocd/` directory | - |
| **CLEAN-12** | Delete `infra/environments/` directory | - |
| **CLEAN-13** | Delete `infra/bicep/` directory | - |
| **CLEAN-14** | Delete `infra/azure/` directory | - |
| **CLEAN-15** | Delete `tools/bootstrap/` directory | - |
| **CLEAN-16** | Update `.devcontainer/lib/` (remove k3d, argocd modules) | - |
| **CLEAN-17** | Remove k3d-related scripts from `tilt/` | - |

### Phase 5: Documentation & Validation

| Task | Description | Owner |
|------|-------------|-------|
| **DOC-10** | Update `docs/dev-workflows/telepresence.md` | - |
| **DOC-11** | Update main README.md | - |
| **DOC-12** | Create troubleshooting guide for Telepresence | - |
| **DOC-13** | Document AKS access prerequisites (Azure AD) | - |
| **VAL-10** | Test fresh DevContainer startup | - |
| **VAL-11** | Test Tilt up against AKS | - |
| **VAL-12** | Test Telepresence intercept workflow | - |

---

## AKS Dev Cluster Requirements

### Cluster: `ameide-dev-aks`

- **Resource Group:** `Ameide-Dev`
- **Region:** Same as staging for latency
- **Node Pool:** Cost-optimized (B-series, spot instances)
- **Namespaces:**
  - `ameide` - Shared services + developer deployments
  - `argocd` - ArgoCD installation

### Pre-installed Components

| Component | Purpose | Managed By |
|-----------|---------|------------|
| ArgoCD | GitOps deployments | gitops bootstrap |
| Traffic Manager | Telepresence networking | gitops bootstrap |
| Envoy Gateway | Ingress/routing | ArgoCD |
| Keycloak | Auth | ArgoCD |
| PostgreSQL (CNPG) | Databases | ArgoCD |
| Redis | Caching | ArgoCD |

### Access Control

- Azure AD authentication via `kubelogin`
- RBAC: Developers get `edit` role in `ameide` namespace
- Telepresence: Traffic Manager deployed cluster-side

### Registry: ACR

- **Registry:** `ameidedev.azurecr.io`
- **Auth:** Azure AD (same as AKS)
- **Used by:** Tilt image pushes, ArgoCD deployments

---

## Telepresence Reference

### Installation (in DevContainer)

```dockerfile
# In .devcontainer/Dockerfile
RUN curl -fL https://app.getambassador.io/download/tel2oss/releases/download/v2.19.1/telepresence-linux-amd64 \
    -o /usr/local/bin/telepresence && chmod +x /usr/local/bin/telepresence
```

### Core Commands

```bash
# Connect to cluster (VPN-like tunnel)
telepresence connect --namespace ameide

# Check connection status
telepresence status

# List available services to intercept
telepresence list

# Intercept a service (route traffic to local)
telepresence intercept www-ameide-platform --port 3000:3000

# Intercept with replace (stop original container)
telepresence intercept www-ameide-platform --port 3000:3000 --replace

# Leave intercept
telepresence leave www-ameide-platform

# Disconnect from cluster
telepresence quit
```

### Environment Variables

When intercepting, Telepresence injects pod environment variables into your local process:
- `TELEPRESENCE_INTERCEPT_ID`
- `TELEPRESENCE_ROOT` (mount point for pod filesystem)
- All env vars from the intercepted pod (ConfigMaps, Secrets)

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Your DevContainer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Telepresence │  │ User Daemon  │  │ Root Daemon  │               │
│  │ CLI          │──│ (cluster     │──│ (VIF/DNS     │               │
│  │              │  │  traffic)    │  │  routing)    │               │
│  └──────────────┘  └──────┬───────┘  └──────────────┘               │
└───────────────────────────┼─────────────────────────────────────────┘
                            │ tunnel
                            ▼
┌───────────────────────────────────────────────────────────────────────┐
│                         AKS Cluster                                    │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Traffic Manager (telepresence-traffic-manager)                    │ │
│  │ - Manages intercepts                                              │ │
│  │ - Injects Traffic Agent sidecars                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌─────────────────────────────────────────────────┐                  │
│  │ Pod: www-ameide-platform                         │                  │
│  │  ┌─────────────────┐  ┌─────────────────┐       │                  │
│  │  │ Traffic Agent   │  │ App Container   │       │                  │
│  │  │ (sidecar)       │──│ (original)      │       │                  │
│  │  │                 │  │                 │       │                  │
│  │  │ if intercept:   │  │ if intercept:   │       │                  │
│  │  │ → route to      │  │ → paused or     │       │                  │
│  │  │   devcontainer  │  │   still running │       │                  │
│  │  └─────────────────┘  └─────────────────┘       │                  │
│  └─────────────────────────────────────────────────┘                  │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Network dependency | Document offline limitations; accept this trade-off |
| Shared cluster conflicts | Namespace isolation; `-tilt` suffix releases; resource quotas |
| Slower image push to ACR | Use Telepresence intercepts for rapid iteration; only push for full tests |
| Azure auth complexity | Pre-configure kubelogin in DevContainer; document troubleshooting |
| Cost of always-on cluster | Auto-scaling; spot instances; shutdown schedule for nights/weekends |
| Telepresence sidecar drift | ArgoCD ignores Traffic Agent annotations in dev namespace |

---

## Success Criteria

1. ✅ Fresh DevContainer starts in <5 minutes (no k3d bootstrap)
2. ✅ `telepresence connect` establishes cluster access in <30 seconds
3. ✅ `telepresence intercept` routes traffic to local process in <10 seconds
4. ✅ Zero infrastructure code in ameide-core repo
5. ✅ Single source of truth for cluster config in ameide-gitops
6. ✅ Developer can go from clone → running service in <10 minutes

---

## References

- [Telepresence Home](https://telepresence.io/)
- [Telepresence Architecture](https://telepresence.io/docs/reference/architecture)
- [Telepresence Intercept CLI](https://telepresence.io/docs/2.19/reference/intercepts/cli)
- [Tilt: Local vs Remote](https://docs.tilt.dev/local_vs_remote.html)
- [Tilt: Setting up any Image Registry](https://docs.tilt.dev/personal_registry.html)
- [Kubernetes: Local Debugging with Telepresence](https://kubernetes.io/docs/tasks/debug/debug-cluster/local-debugging/)
- [Azure AKS: Use Telepresence](https://learn.microsoft.com/en-us/azure/aks/use-telepresence-aks)

---

## Related Backlogs

### Preserved (Critical Architecture)
- **373** (Argo + Tilt + Helm North Star v4) - Defines two-path architecture (Argo baseline + Tilt overlay)
- **424** (Tilt-only releases) - `apps-*-tilt` isolation pattern

### Superseded
- **367** (Bootstrap v2) - Bootstrap logic moves to gitops
- **429** (DevContainer Bootstrap) - Simplified; most content moves to gitops
- **432** (DevContainer Modes) - Retired; single mode (AKS + Telepresence)
