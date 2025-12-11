# 503 – Operators Helm Chart

**Status:** Active (Phase 1 Complete)
**Audience:** Platform engineers, GitOps team
**Scope:** Unified Helm chart for deploying AMEIDE primitive operators

> **Related**:
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
> - [498-domain-operator.md](498-domain-operator.md) – Domain operator development
> - [499-process-operator.md](499-process-operator.md) – Process operator development
> - [500-agent-operator.md](500-agent-operator.md) – Agent operator development
> - [501-uisurface-operator.md](501-uisurface-operator.md) – UISurface operator development

---

## 1. Overview

The operators Helm chart provides a unified deployment mechanism for all four AMEIDE primitive operators:

| Operator | CRD | Purpose |
|----------|-----|---------|
| **domain-operator** | `domains.ameide.io` | Manages Domain primitives (gRPC services + DB) |
| **process-operator** | `processes.ameide.io` | Manages Process primitives (Temporal workers) |
| **agent-operator** | `agents.ameide.io` | Manages Agent primitives (LLM-powered actors) |
| **uisurface-operator** | `uisurfaces.ameide.io` | Manages UISurface primitives (Next.js apps) |

**Key insight**: Single chart, shared RBAC, independent enable/disable per operator.

---

## 2. Chart Location

```
operators/helm/
├── Chart.yaml              # Chart metadata (v0.1.0)
├── values.yaml             # Default values for all operators
├── README.md               # GitOps deployment guide
├── crds/                   # CRDs (auto-installed by Helm)
│   ├── ameide.io_domains.yaml
│   ├── ameide.io_processes.yaml
│   ├── ameide.io_agents.yaml
│   └── ameide.io_uisurfaces.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── serviceaccount.yaml
│   ├── clusterrole.yaml
│   ├── clusterrolebinding.yaml
│   ├── domain-operator-deployment.yaml
│   ├── process-operator-deployment.yaml
│   ├── agent-operator-deployment.yaml
│   ├── uisurface-operator-deployment.yaml
│   └── NOTES.txt
└── examples/               # Sample CRs for testing
    ├── domain-sample.yaml
    ├── process-sample.yaml
    ├── agent-sample.yaml
    └── uisurface-sample.yaml
```

---

## 3. Image Registry

Operator images are published to GHCR via `cd-service-images.yml`:

| Image | Tags |
|-------|------|
| `ghcr.io/ameideio/domain-operator` | `dev`, `main`, `v1.2.3`, `<sha>` |
| `ghcr.io/ameideio/process-operator` | `dev`, `main`, `v1.2.3`, `<sha>` |
| `ghcr.io/ameideio/agent-operator` | `dev`, `main`, `v1.2.3`, `<sha>` |
| `ghcr.io/ameideio/uisurface-operator` | `dev`, `main`, `v1.2.3`, `<sha>` |

### Tag Strategy

| Branch/Tag | Image Tag | Use Case |
|------------|-----------|----------|
| `dev` branch | `:dev` | Development environments |
| `main` branch | `:main` | Staging / pre-production |
| `v*` tags | `:1.2.3`, `:1.2` | Production releases |
| Any commit | `:<sha>` | Pinned deployments |

---

## 4. Development Phases

### Phase 1: Core Chart ✅ IMPLEMENTED

| Task | Description | Status |
|------|-------------|--------|
| **Chart.yaml** | Chart metadata, version 0.1.0 | ✅ |
| **values.yaml** | Default values for all operators | ✅ |
| **CRDs** | Copy from operator config/crd/bases | ✅ |
| **ServiceAccount** | Shared ServiceAccount for all operators | ✅ |
| **ClusterRole** | RBAC for all four CRDs + owned resources | ✅ |
| **ClusterRoleBinding** | Bind ClusterRole to ServiceAccount | ✅ |
| **Deployments** | One Deployment per operator | ✅ |
| **NOTES.txt** | Post-install instructions | ✅ |
| **Examples** | Sample CRs for testing | ✅ |
| **README** | GitOps deployment guide | ✅ |

### Phase 2: ArgoCD Integration (ameide-gitops)

The chart integrates with the existing ameide-gitops ApplicationSet architecture.

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Component definition** | Create `component.yaml` in `environments/_shared/components/foundation/operators/ameide-operators/` | ApplicationSet discovers component |
| **Shared values** | Create `sources/values/_shared/foundation/foundation-ameide-operators.yaml` | Defaults applied to all envs |
| **Dev values** | Create `sources/values/dev/foundation/foundation-ameide-operators.yaml` | Dev-specific config (`:dev` tag, tolerations) |
| **Staging values** | Create `sources/values/staging/foundation/foundation-ameide-operators.yaml` | Staging config (`:main` tag) |
| **Production values** | Create `sources/values/production/foundation/foundation-ameide-operators.yaml` | Prod config (pinned version, HA) |

#### Component Definition

```yaml
# environments/_shared/components/foundation/operators/ameide-operators/component.yaml
name: foundation-ameide-operators
project: ameide
domain: foundation
dependencyPhase: "controllers"
componentType: "operator"
rolloutPhase: "120"  # After CRDs (110), with other operators
chart:
  repoURL: https://github.com/ameideio/ameide-core.git
  path: operators/helm
  version: main
syncOptions:
  - CreateNamespace=false
  - RespectIgnoreDifferences=true
  - ServerSideApply=true
```

#### Resulting Applications

| Application Name | Environment | Namespace |
|------------------|-------------|-----------|
| `dev-foundation-ameide-operators` | dev | ameide-dev |
| `staging-foundation-ameide-operators` | staging | ameide-staging |
| `production-foundation-ameide-operators` | production | ameide-prod |

### Phase 3: Production Hardening

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **PodDisruptionBudget** | PDB for each operator | HA during node drain |
| **ServiceMonitor** | Prometheus scraping | Metrics in Grafana |
| **PrometheusRule** | Alerting rules | Alerts on reconcile errors |
| **NetworkPolicy** | Restrict operator network access | Only API server access |
| **Pod anti-affinity** | Spread operators across nodes | HA topology |

### Phase 4: Release Automation

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Chart versioning** | Bump chart version on release | Chart version matches app |
| **Helm package** | Package chart as .tgz | Artifact published |
| **Helm repository** | OCI registry or GitHub Pages | `helm repo add` works |
| **Release notes** | Auto-generate from commits | Notes in GitHub release |

---

## 5. Values Schema

```yaml
global:
  imageRegistry: ghcr.io/ameideio
  imagePullPolicy: IfNotPresent
  imagePullSecrets: []

operators:
  domain:
    enabled: true
    replicas: 1
    image:
      repository: domain-operator
      tag: "latest"
    resources:
      limits:
        cpu: 500m
        memory: 128Mi
      requests:
        cpu: 10m
        memory: 64Mi
    leaderElection: true
    metricsBindAddress: ":8080"
    healthProbeBindAddress: ":8081"

  process:
    enabled: true
    # ... same structure

  agent:
    enabled: true
    # ... same structure

  uisurface:
    enabled: true
    # ... same structure

rbac:
  create: true

serviceAccount:
  create: true
  name: ""
  annotations: {}

podSecurityContext:
  runAsNonRoot: true

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL

nodeSelector: {}
tolerations: []
affinity: {}
```

---

## 6. Usage Examples

### Install All Operators

```bash
helm install ameide ./operators/helm \
  -n ameide-system \
  --create-namespace
```

### Install Only Domain Operator

```bash
helm install ameide ./operators/helm \
  -n ameide-system \
  --create-namespace \
  --set operators.process.enabled=false \
  --set operators.agent.enabled=false \
  --set operators.uisurface.enabled=false
```

### Use Dev Images

```bash
helm install ameide ./operators/helm \
  -n ameide-system \
  --create-namespace \
  --set global.imagePullPolicy=Always \
  --set operators.domain.image.tag=dev \
  --set operators.process.image.tag=dev \
  --set operators.agent.image.tag=dev \
  --set operators.uisurface.image.tag=dev
```

### Verify Installation

```bash
# Check CRDs
kubectl get crd | grep ameide.io

# Check operators
kubectl -n ameide-system get deploy

# Check RBAC
kubectl get clusterrole | grep ameide

# Test with sample CR
kubectl apply -f operators/helm/examples/domain-sample.yaml
kubectl get domain sample-domain -o yaml
```

---

## 7. ClusterRole Permissions

The shared ClusterRole covers:

| API Group | Resources | Verbs |
|-----------|-----------|-------|
| `ameide.io` | domains, processes, agents, uisurfaces | * |
| `ameide.io` | */status, */finalizers | * |
| `apps` | deployments | * |
| `core` | services, configmaps, secrets | * |
| `batch` | jobs, cronjobs | * |
| `networking.k8s.io` | networkpolicies | * |
| `gateway.networking.k8s.io` | httproutes | * |
| `external-secrets.io` | externalsecrets | * |
| `coordination.k8s.io` | leases | * |
| `core` | events | create, patch |

---

## 8. Non-Goals

| What | Why |
|------|-----|
| Helm repository hosting | Use OCI registry or git-based install |
| Operator Lifecycle Manager (OLM) | ArgoCD is primary deployment method |
| Multi-cluster deployment | Handled by ArgoCD ApplicationSet |
| CRD conversion webhooks | Defer to individual operators |

---

## 9. Dependencies

| Dependency | Purpose |
|------------|---------|
| **Kubernetes 1.28+** | Gateway API support |
| **Helm 3.12+** | OCI registry support |
| **ArgoCD 2.8+** | GitOps deployment |
| **Gateway API CRDs** | UISurface HTTPRoutes |
| **CNPG** | Domain database management |
| **External Secrets Operator** | Agent secrets |

---

## 10. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [495-ameide-operators.md](495-ameide-operators.md) | CRD shapes & responsibilities |
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Go patterns & reference implementation |
| [498-domain-operator.md](498-domain-operator.md) | Domain operator development |
| [499-process-operator.md](499-process-operator.md) | Process operator development |
| [500-agent-operator.md](500-agent-operator.md) | Agent operator development |
| [501-uisurface-operator.md](501-uisurface-operator.md) | UISurface operator development |
| [502-domain-vertical-slice.md](502-domain-vertical-slice.md) | Phase E (Helm chart) implementation |
