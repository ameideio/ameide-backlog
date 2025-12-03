# 435 - Remote-First Development Architecture

**Status:** Phase 1 Complete, Phase 2 In Progress
**Created:** 2024-12-02
**Updated:** 2025-12-03
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

## Why Remote Dev Cluster (No k3d)

Tilt's [Choosing a Local Dev Cluster](https://docs.tilt.dev/choosing_clusters.html) guide recommends remote dev clusters when:
- You have a dedicated infra team maintaining cluster setup
- Team is large enough to justify shared infrastructure
- You want consistent environments across all developers

We meet all three criteria:
- `ameide-gitops` repo + bootstrap scripts = dedicated infra
- GitOps team maintains AKS clusters
- Eliminates "works on my k3d" issues

**Trade-off accepted:** Network dependency. No offline development.

## Version Pinning

We standardize on **Telepresence 2.x** (currently 2.25.1). The exact version is pinned in `.devcontainer/Dockerfile`.

Some CLI flags and behaviors evolve between minor versions. If you see unexpected behavior, check:
1. Your installed version: `telepresence version`
2. The pinned version in Dockerfile
3. The [Telepresence changelog](https://github.com/telepresenceio/telepresence/releases)

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

### Developer Workflow

```bash
# 1. Connect to cluster (VPN-like tunnel)
telepresence connect --context ameide-dev --namespace ameide-dev

# 2. Intercept the service you're developing
telepresence intercept www-ameide-platform --context ameide-dev --namespace ameide-dev --port 3000:3000 --env-file=.env.telepresence

# 3. Run locally with pod environment
source .env.telepresence && npm run dev
```

**What happens:**
- Traffic from AKS cluster → routed to your local process
- Code changes → instant (no container rebuild)
- Pod's ConfigMaps/Secrets → available via `--env-file`

**When to use full image builds (Tilt):**
- Dockerfile changes
- Dependency updates (package.json, go.mod, requirements.txt)
- Final integration testing before PR

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

## Infra bootstrap automation (2025-12-03)

- `scripts/deploy-managed-app.sh` replaces bespoke `az deployment` incantations. It loads `.env` / `.env.local`, reuses the gitignored AKS admin key pair generated by `scripts/ensure-aks-admin-key.sh`, injects the GitHub token securely, and wraps `az deployment group create`. After every run it captures the outputs JSON under `artifacts/bicep-outputs/<env>.json`.
- `scripts/update-globals-from-bicep.sh` consumes those outputs and rewrites `sources/values/<env>/globals.yaml` with the latest `azure.envoyPublicIpAddress` and DNS managed identity IDs, so Helm/Argo never drift from the Bicep truth. Commit the regenerated globals whenever the control plane IP or identity rotates.
- Managed application parameter files now set `enableDnsFederatedIdentity=true`, `envSecrets` is marked `@secure()`, and the new `modules/keyvaultEnvSecrets.bicep` mirrors every `.env` entry into Azure Key Vault automatically. Cert-manager’s Azure DNS workload identity credential appears on day one with no `az identity federated-credential` hand edits.
- `scripts/setup-service-principal.sh` provisions a Contributor-scoped service principal with a locally stored certificate (under `creds/service-principal/`). When its profile JSON exists, `scripts/deploy-managed-app.sh` automatically runs `az login --service-principal ...` so CI/automation can deploy without interactive `az login`.

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

> **Status Update (2025-12-03):** DevContainer in ameide-gitops simplified. Telepresence workflow
> documented in `scripts/devcontainer/README.md`. Helper script (DC-14) not yet created; CLI
> installation (DC-15) uses cluster-side Traffic Manager (chart v2.25.1) but DevContainer-side
> CLI install is pending.

| Task | Description | Owner | Status |
|------|-------------|-------|--------|
| **DC-10** | Remove k3d cluster creation from postCreate.sh | - | ✅ Done |
| **DC-11** | Remove k3d feature from devcontainer.json | - | ✅ Done |
| **DC-12** | Remove ArgoCD bootstrap logic (Argo lives in AKS, managed by gitops) | - | ✅ Done |
| **DC-13** | Add AKS credential setup (az login + kubelogin) | - | ✅ Done |
| **DC-14** | Create `tools/dev/telepresence.sh` helper script | - | ⏳ Workflow in README, script not created |
| **DC-15** | Install Telepresence CLI in Dockerfile | - | ⏳ Cluster-side ready, CLI install pending |
| **DC-16** | Remove `.devcontainer/lib/k3d.sh` and related modules | - | ✅ Done |

**Target postCreate.sh (~30 lines):**
```bash
#!/bin/bash
set -euo pipefail

# Install dependencies
pnpm install

# Configure kubectl for AKS (dev namespace)
az aks get-credentials --resource-group Ameide --name ameide --overwrite-existing --context ameide-dev
kubelogin convert-kubeconfig -l azurecli
kubectl config set-context ameide-dev --namespace=ameide-dev >/dev/null 2>&1 || true

echo "✅ Ready! Run './tools/dev/telepresence.sh connect' to connect to the cluster."
```

> **Note:** `telepresence connect` is an explicit developer step, not run automatically in postCreate.
> This lets developers see connection errors clearly and choose their namespace/context.

**Helper script `tools/dev/telepresence.sh`:**
```bash
#!/bin/bash
set -euo pipefail

case "${1:-}" in
  connect)
    telepresence connect --context ameide-dev --namespace ameide-dev
    ;;
  intercept)
    SERVICE="${2:?Usage: $0 intercept <service> [port]}"
    PORT="${3:-3000:3000}"
    telepresence intercept "$SERVICE" --context ameide-dev --namespace ameide-dev --port "$PORT" --env-file=.env.telepresence
    echo "Run: source .env.telepresence && <your-dev-command>"
    ;;
  status)
    telepresence status
    ;;
  quit)
    telepresence quit
    ;;
  *)
    echo "Usage: $0 {connect|intercept|status|quit}"
    exit 1
    ;;
esac
```

### Phase 3: Adapt Tiltfile for Remote Cluster

| Task | Description | Owner |
|------|-------------|-------|
| **TILT-10** | Add `default_registry('ghcr.io/ameideio')` for GHCR | - |
| **TILT-11** | Update kubectl context defaults (remove k3d references) | - |
| **TILT-12** | Keep all `apps-*-tilt` Helm releases (373/424 architecture) | - |
| **TILT-13** | Add `local_resource` for Telepresence intercepts | - |
| **TILT-14** | Remove k3d-specific registry wiring (`k3d-ameide.localhost:5001`) | - |
| **TILT-15** | Update values paths to reference gitops submodule | - |
| **TILT-16** | Add Telepresence intercept helpers to `tilt/` | - |

**Example Tiltfile additions:**
```python
# Registry for AKS (GitHub Container Registry)
default_registry('ghcr.io/ameideio')

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

## AKS Shared Cluster Requirements

### Cluster: `ameide`

- **Resource Group:** `Ameide`
- **Region:** `westeurope` (shared across all environments)
- **Node Pool:** Cost-optimized (B-series, spot instances) with autoscaling
- **Namespaces:**
  - `ameide-dev` – Dev workloads (default Telepresence namespace)
  - `ameide-staging` – Staging workloads
  - `ameide-prod` – Reserved for production workloads
  - `argocd`, `observability`, etc. – Shared platform components

### Argo CD control plane

- A single Argo installation in the `argocd` namespace reconciles every environment namespace (`ameide-dev`, `ameide-staging`, `ameide-prod`) using ApplicationSets and Projects for boundaries.
- There is no per-environment Argo; isolation is achieved through Projects + RBAC and the GitOps repo structure.

### Image & tag authority

- **Source of truth:** `ameide-gitops` defines the repository + tag per environment. CI builds push to GitHub Container Registry (`ghcr.io/ameideio/...`) and update the GitOps values; Argo syncs those tags.
- **Tilt-only tags:** Developer iterations use the `*-tilt` releases and developer-scoped GHCR tags, never overwriting Argo-managed baselines.
- **Telepresence:** Only hijacks traffic; it does not inject or override images/tags.

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
- RBAC: Developers get `edit` role in `ameide-dev` namespace
- Telepresence: Traffic Manager deployed cluster-side

### RBAC Requirements for Telepresence

Telepresence `intercept` and `replace` require permissions to patch workloads (Deployment/StatefulSet) in the target namespace. The `edit` ClusterRole provides this.

Required permissions:
- `get`, `list`, `watch` on deployments, statefulsets, services
- `patch`, `update` on deployments, statefulsets (for Traffic Agent injection)
- `create` on pods/exec (for some debugging features)

### ArgoCD + Telepresence Drift Avoidance

**Critical rule:** Only intercept Tilt-owned workloads (`apps-*-tilt`), never ArgoCD-managed baselines.

When Telepresence intercepts a workload, it:
- Injects a Traffic Agent sidecar container
- Adds annotations like `telepresence.getambassador.io/*`
- Modifies the pod spec

If you intercept an ArgoCD-managed workload, ArgoCD will see constant drift and fight to restore the original spec.

**Safe pattern:**
```bash
# ✅ Safe: Intercept Tilt-owned release
telepresence intercept apps-www-ameide-platform-tilt --port 3000:3000

# ❌ Avoid: Intercept ArgoCD-managed baseline
telepresence intercept apps-www-ameide-platform --port 3000:3000
```

If you must intercept ArgoCD workloads in dev, configure ArgoCD to ignore Telepresence annotations:
```yaml
# In ArgoCD Application
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/template/metadata/annotations/telepresence.getambassador.io~1inject-traffic-agent
```

### Registry: GitHub Container Registry

- **Registry:** `ghcr.io/ameideio`
- **Auth:** GitHub token scoped for the release workflows
- **Used by:** Tilt image pushes, CI publishes, ArgoCD deployments (via GitOps values)
---

## Telepresence Reference

### Installation (in DevContainer)

```dockerfile
# In .devcontainer/Dockerfile
RUN curl -fL https://app.getambassador.io/download/tel2oss/releases/download/v2.25.1/telepresence-linux-amd64 \
    -o /usr/local/bin/telepresence && chmod +x /usr/local/bin/telepresence
```

### Core Commands

```bash
# Connect to cluster (VPN-like tunnel)
telepresence connect --context ameide-dev --namespace ameide-dev

# Check connection status
telepresence status

# List available services to intercept
telepresence list

# Intercept a service (app container keeps running, traffic mirrored)
telepresence intercept www-ameide-platform --context ameide-dev --namespace ameide-dev --port 3000:3000 --env-file=.env.telepresence

# Replace a service (app container stopped, only your local process handles traffic)
# Use this for queue consumers or when you need exclusive access
telepresence replace www-ameide-platform --port 3000:3000 -- npm run dev

# Leave intercept
telepresence leave www-ameide-platform

# Disconnect from cluster
telepresence quit
```

> **Note:** `--replace` flag on `intercept` is deprecated. Use `telepresence replace` instead.

### Environment Variables

Telepresence can import the intercepted pod's environment into your local process via:

1. **`--env-file`** (recommended) - Write env vars to a file:
   ```bash
   telepresence intercept my-service --port 3000:3000 --env-file=.env.telepresence
   source .env.telepresence && npm run dev
   ```

2. **`replace -- <command>`** - Run command with pod env applied:
   ```bash
   telepresence replace my-service --port 3000:3000 -- npm run dev
   ```

**Available variables:**
- `TELEPRESENCE_INTERCEPT_ID` - Unique intercept identifier
- `TELEPRESENCE_ROOT` - Mount point for pod filesystem (volumes)
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
- [Telepresence Intercept CLI](https://telepresence.io/docs/2.25/reference/intercepts/cli)
- [Tilt: Local vs Remote](https://docs.tilt.dev/local_vs_remote.html)
- [Tilt: Setting up any Image Registry](https://docs.tilt.dev/personal_registry.html)
- [Kubernetes: Local Debugging with Telepresence](https://kubernetes.io/docs/tasks/debug/debug-cluster/local-debugging/)
- [Azure AKS: Use Telepresence](https://learn.microsoft.com/en-us/azure/aks/use-telepresence-aks)

---

## Related Backlogs

### Preserved (Critical Architecture)
- **373** (Argo + Tilt + Helm North Star v4) - Defines two-path architecture (Argo baseline + Tilt overlay)
- **424** (Tilt-only releases) - `apps-*-tilt` isolation pattern
- **[434](434-unified-environment-naming.md)** (Unified Environment Naming) - Domain matrix, static IP flow, ArgoCD at `argocd.ameide.io`
- **[436](436-envoy-gateway-observability.md)** (Envoy Gateway Observability) - EnvoyProxy telemetry and static IP config
- **[438](438-cert-manager-dns01-azure-workload-identity.md)** (cert-manager DNS-01) - Certificate issuance per environment

### Superseded
- **367** (Bootstrap v2) - Bootstrap logic moves to gitops
- **429** (DevContainer Bootstrap) - Simplified; most content moves to gitops
- **432** (DevContainer Modes) - Retired; single mode (AKS + Telepresence)
