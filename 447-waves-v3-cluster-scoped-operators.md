# ArgoCD Rollout Phases v3: Dual ApplicationSet Architecture

> **Note:** This document (447) covers rollout phases and cluster-scoped operators.
> For third-party chart tolerations, see [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md).

**Status**: Implemented
**Supersedes**: 387-argocd-waves-v2.md (partially - phase numbering retained, architecture updated)

> **Cross-References (Deployment Architecture Suite)**:
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – **READ THIS FIRST**: How the two ApplicationSets work
> - [464-chart-folder-alignment.md](464-chart-folder-alignment.md) – Chart folder structure and domain alignment
> - [426-keycloak-config-map.md](426-keycloak-config-map.md) – Secrets and OIDC client patterns
> - [473-ameide-technology.md](473-ameide-technology.md) – Technology architecture (Backstage, Temporal, etc.)
>
> **Related (Implementation Details)**:
> - [446-namespace-isolation.md](446-namespace-isolation.md), [448-per-environment-service-verification.md](448-per-environment-service-verification.md), [449-per-environment-infrastructure.md](449-per-environment-infrastructure.md), [452-observability-namespace-isolation.md](452-observability-namespace-isolation.md), [455-traffic-manager-ignoredifferences.md](455-traffic-manager-ignoredifferences.md), [456-ghcr-mirror.md](456-ghcr-mirror.md)

## Problem Statement

The v2 design (387) assumed operators could be deployed per-environment using a single ApplicationSet. This caused:

1. **ClusterRole conflicts**: Operators create ClusterRoles with hardcoded names (e.g., `cloudnative-pg`). Multiple environments deploying the same operator created conflicts.
2. **Webhook conflicts**: MutatingWebhookConfiguration/ValidatingWebhookConfiguration resources are cluster-scoped with fixed names.
3. **ArgoCD ownership conflicts**: Multiple Applications trying to manage the same cluster-scoped resource causes sync failures.
4. **Resource waste**: Running 3 copies of each operator (dev/staging/production) when one suffices.

## Solution: Dual ApplicationSet Model

Split deployment into two ApplicationSets:

| ApplicationSet | Scope | Deploys | Namespace Pattern |
|----------------|-------|---------|-------------------|
| `cluster` | Cluster-wide | CRDs, Operators | `-system` namespaces |
| `ameide` | Per-environment | Workloads (CRs) | `ameide-{env}` |

### Cluster ApplicationSet (`argocd/applicationsets/cluster.yaml`)

Deploys resources that are **cluster-scoped by nature**:
- CRDs (CustomResourceDefinitions are always cluster-scoped)
- Operators (create ClusterRoles, Webhooks, watch multiple namespaces)

```yaml
generators:
  - git:
      files:
        - path: environments/_shared/components/cluster/**/component.yaml
strategy:
  type: RollingSync
  rollingSync:
    steps:
      - maxUpdate: 10  # CRDs first
        matchExpressions:
          - key: rollout-phase
            operator: In
            values: ["010"]
      - maxUpdate: 5   # Operators
        matchExpressions:
          - key: rollout-phase
            operator: In
            values: ["020"]
```

### Environment ApplicationSet (`argocd/applicationsets/ameide.yaml`)

Deploys **environment-scoped workloads** (Custom Resources managed by the cluster operators):
- Database clusters (CNPG Cluster, ClickHouseInstallation, RedisFailover)
- Kafka clusters (Kafka CR)
- Keycloak instances (Keycloak CR)
- Application deployments

```yaml
generators:
  - matrix:
      generators:
        - list:
            elements:
              - env: dev
                namespace: ameide-dev
              - env: staging
                namespace: ameide-staging
              - env: production
                namespace: ameide-prod
        - git:
            files:
              - path: environments/_shared/components/apps/**/component.yaml
              - path: environments/_shared/components/data/**/component.yaml
              - path: environments/_shared/components/foundation/**/component.yaml
              - path: environments/_shared/components/platform/**/component.yaml
```

## Directory Structure

```
environments/_shared/components/
├── cluster/                    # Cluster-scoped (deployed ONCE)
│   ├── crds/                   # CRDs at rollout-phase 010
│   │   ├── cert-manager/
│   │   ├── cnpg/
│   │   ├── clickhouse/
│   │   ├── external-secrets/
│   │   ├── gateway-api/
│   │   ├── keycloak/
│   │   ├── prometheus-operator/
│   │   ├── redis/
│   │   ├── strimzi/
│   │   └── temporal-operator/
│   └── operators/              # Operators at rollout-phase 020
│       ├── cert-manager/       → cert-manager namespace
│       ├── clickhouse/         → clickhouse-system namespace
│       ├── cloudnative-pg/     → cnpg-system namespace
│       ├── external-secrets/   → external-secrets namespace
│       ├── keycloak/           → keycloak-system namespace
│       ├── redis/              → redis-system namespace
│       ├── strimzi/            → strimzi-system namespace
│       └── temporal-operator/  → ameide-system namespace (see note below)
├── foundation/                 # Per-environment foundation
├── data/                       # Per-environment data workloads
├── platform/                   # Per-environment platform
└── apps/                       # Per-environment applications
```

## Operator Namespace Strategy

Each operator deploys to a dedicated `-system` namespace:

| Operator | Namespace | Watches |
|----------|-----------|---------|
| CloudNative-PG | `cnpg-system` | All namespaces |
| Strimzi | `strimzi-system` | All namespaces |
| ClickHouse | `clickhouse-system` | All namespaces |
| Redis | `redis-system` | All namespaces |
| Keycloak | `keycloak-system` | All namespaces |
| cert-manager | `cert-manager` | All namespaces |
| external-secrets | `external-secrets` | All namespaces |
| Temporal operator | `ameide-system` | All namespaces |

**Note on Temporal operator namespace:** the Temporal operator bundles admission webhooks and requires cert-manager to mint its serving certificate and inject the CA bundle. With the **single cert-manager per cluster** policy, this is handled by `cluster-cert-manager` (in `cert-manager`) and works for webhooks in `ameide-system` as long as rollout ordering keeps cert-manager healthy before the operator is installed.

Operators are configured with `watchNamespaces: []` (empty = all namespaces) so they can manage CRs in any environment namespace.

## Node Tolerations

Cluster nodes have environment-specific taints:
- `ameide.io/environment: dev`
- `ameide.io/environment: staging`
- `ameide.io/environment: production`
- `CriticalAddonsOnly: true` (system nodes)

Cluster-scoped operators run on system nodes with `CriticalAddonsOnly` toleration:

```yaml
tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
```

## Rollout Phase Bands

### Cluster Phases (000-099)

| Phase | Type | Components |
|-------|------|------------|
| 010 | CRDs | All CRDs (cert-manager, cnpg, clickhouse, external-secrets, gateway-api, keycloak, prometheus, redis, strimzi) |
| 020 | Operators | All operators (cert-manager, cloudnative-pg, clickhouse, external-secrets, keycloak, redis, strimzi) |
| 030 | Post-operator config | Reserved for cluster-wide configs |

### Environment Phases (100-699)

Retained from v2 design for workloads:

| Band | Purpose | Phases |
|------|---------|--------|
| Foundation | Core infrastructure | 100-199 |
| Data Core | Database/messaging workloads | 200-299 |
| Platform | Platform services | 300-399 |
| Data Extended | Temporal, etc. | 400-499 |
| Observability | Monitoring/logging | 500-599 |
| Apps | Application workloads | 600-699 |

#### Sub-phase Convention

Within each band, use these sub-phases for consistent ordering:

| Sub-phase | Purpose | Example |
|-----------|---------|---------|
| `*10` | CRDs (environment-specific, if any) | 310: platform-envoy-crds |
| `*20` | Controllers (environment-specific, if any) | 320: envoy-gateway |
| `*30` | Secrets | 330: platform Secret/ExternalSecret payloads |
| `*40` | Configs/bootstraps | 340: Gateway, cert-manager-config |
| `*50` | Runtimes/workloads | 350: Keycloak |
| `*55` | Post-runtime bootstraps | 355: keycloak-realm, client-patcher |
| `*56+` | Services dependent on bootstraps | 356: Backstage (needs client-patcher secret) |
| `*99` | Smokes | 399: control-plane-smoke, auth-smoke |

#### Band Details

**Foundation (100-199)**
- 110: CRDs (foundation CRDs such as cert-manager)
- 120: Operators/controllers (cert-manager, external-secrets)
- 130: Secret providers and shared secrets (vault-webhook-certs)
- 140: Configs/bootstraps (managed-storage, coredns-config)
- 150: Runtimes (Vault server)
- 155: Post-runtime bootstraps (vault-bootstrap, vault-secret-store)
- 199: Smokes (foundation-operators-smoke, foundation-bootstrap-smoke)

**Data Core (200-299)**
- 210: CRDs (redis, clickhouse, CNPG, strimzi)
- 220: Operators (cloudnative-pg, clickhouse-operator, redis-operator, strimzi)
- 230: Secrets (clickhouse-secrets, data DB creds)
- 240: Configs (cnpg-monitoring, db-migrations prep)
- 250: Runtimes (postgres-clusters, clickhouse, kafka-cluster, redis-failover, minio)
- 260: Post-runtime (db-migrations execution)
- 299: Smokes (data-plane-smoke)

**Platform (300-399)**
- 310: CRDs (envoy-crds, gateway-api, keycloak-crds)
- 320: Operators (envoy-gateway, keycloak-operator)
- 330: Secrets (platform Secret/ExternalSecret payloads, PKI chain)
- 340: Configs (cert-manager-config, Gateway/HTTPRoute CRs)
- 350: Runtimes (Keycloak, envoy data-plane)
- 355: Post-runtime bootstraps (keycloak-realm, client-patcher)
- 356: Post-bootstrap services (Backstage – needs OIDC secret from 355)
- 399: Smokes (control-plane-smoke, auth-smoke, secrets-smoke)

**Data Extended (400-499)**
- 410: CRDs (temporal operator CRDs)
- 420: Operators (temporal operator)
- 430: Secrets (temporal/db secrets from CNPG)
- 440: Configs (pre-runtime configs)
- 450: Runtimes (temporal, pgadmin)
- 499: Smokes (data-plane-ext-smoke)

**Observability (500-599)**
- 510: CRDs (prometheus-operator CRDs)
- 520: Operators (prometheus operator)
- 530: Secrets (vault-secrets-observability)
- 540: Configs (grafana-datasources)
- 550: Runtimes (grafana, loki, tempo, otel-collector, alloy-logs, langfuse, prometheus)
- 599: Smokes (observability-smoke)

**Apps (600-699)**
- 610: CRDs (unlikely/none)
- 620: Operators (if any app-side operator)
- 630: Secrets (app Secret/ExternalSecret payloads)
- 640: Configs (app configmaps)
- 650: Runtimes (platform, graph, transformation, workflows, threads, agents, www-*)
- 699: Smokes (apps-platform-smoke)

## Component Definition Examples

### Cluster CRD Component

```yaml
# environments/_shared/components/cluster/crds/cnpg/component.yaml
name: crds-cnpg
project: ameide
namespace: argocd           # CRDs tracked in argocd namespace
domain: crds
componentType: "crd"
rolloutPhase: "010"
chart:
  repoURL: https://github.com/ameideio/ameide-gitops.git
  path: sources/charts/foundation/common/raw-manifests
  valueFiles:
    - $values/sources/values/_shared/data/data-crds-cnpg.yaml
syncOptions:
  - ServerSideApply=true
```

### Cluster Operator Component

```yaml
# environments/_shared/components/cluster/operators/cloudnative-pg/component.yaml
name: cloudnative-pg
project: ameide
namespace: cnpg-system       # Dedicated operator namespace
domain: operators
componentType: "operator"
rolloutPhase: "020"
chart:
  repoURL: https://github.com/ameideio/ameide-gitops.git
  path: sources/charts/third_party/cnpg/cloudnative-pg/0.26.1
  skipCrds: true            # CRDs installed separately
syncOptions:
  - CreateNamespace=true
  - ServerSideApply=true
```

### Environment Workload Component

```yaml
# environments/_shared/components/platform/postgres-clusters/component.yaml
name: platform-postgres-clusters
project: ameide
namespace: "{{ .namespace }}"  # Resolved per-environment
domain: platform
componentType: "runtime"
rolloutPhase: "250"
chart:
  repoURL: https://github.com/ameideio/ameide-gitops.git
  path: sources/charts/platform/postgres-clusters
```

## Values File Structure

```
sources/values/
├── cluster/                    # Cluster-scoped values
│   └── globals.yaml            # Cluster globals (tolerations, Azure config)
├── _shared/
│   ├── cluster/                # Shared operator configs
│   │   ├── cloudnative-pg.yaml
│   │   ├── strimzi-operator.yaml
│   │   ├── cert-manager.yaml
│   │   └── ...
│   ├── foundation/             # Shared foundation configs
│   ├── data/                   # Shared data configs
│   └── platform/               # Shared platform configs
├── dev/                        # Dev environment overrides
├── staging/                    # Staging environment overrides
└── production/                 # Production environment overrides
```

## Deployment Order

1. **Cluster ApplicationSet syncs first** (sync-wave: -1)
   - Phase 010: All CRDs installed
   - Phase 020: All operators running in `-system` namespaces

2. **Environment ApplicationSet syncs second** (sync-wave: 0)
   - Phases 100+: Workloads deployed per-environment
   - Operators already running, ready to reconcile CRs

## Key Differences from v2

| Aspect | v2 (387) | v3 (447) |
|--------|----------|----------|
| ApplicationSets | 1 (single) | 2 (cluster + ameide) |
| Operator deployment | Per-environment | Once per cluster |
| Operator namespaces | `ameide-{env}` | `-system` namespaces |
| CRD ownership | Per-environment apps | Single cluster app |
| Phase 010-020 | Prereqs in ameide | Dedicated cluster scope |
| Phase 110-120 | Foundation CRDs/operators | Workloads only (operators moved to cluster) |

## Migration Notes

When migrating from v2:
1. Create `cluster` ApplicationSet
2. Move operators from `environments/_shared/components/foundation/operators/` to `environments/_shared/components/cluster/operators/`
3. Move CRDs from domain-specific locations to `environments/_shared/components/cluster/crds/`
4. Update `ameide.yaml` to exclude `cluster/` directory
5. Delete old per-environment operator Applications
6. Add tolerations to operator values for node scheduling

## Validation

Check cluster operators are running:
```bash
kubectl get pods -n cnpg-system
kubectl get pods -n strimzi-system
kubectl get pods -n clickhouse-system
kubectl get pods -n redis-system
kubectl get pods -n keycloak-system
kubectl get pods -n cert-manager
kubectl get pods -n external-secrets
```

Check ApplicationSets:
```bash
kubectl get applicationsets -n argocd
# Should show: cluster, ameide
```

Check cluster apps:
```bash
kubectl get applications -n argocd -l scope=cluster
```

---

## Troubleshooting

### Altinity ClickHouse Operator CHI Informer Issue (2025-12-05)

**Symptom**: CHIs exist but have empty `.status`, no StatefulSets created, operator logs show no `chiInformer` events.

**Root cause**: The Helm-deployed operator in `clickhouse-system` had a broken CHI informer. Operator logs showed:
- `podInformer.AddFunc/UpdateFunc` ✓
- `configMapInformer.AddFunc` ✓
- `endpointSliceInformer.UpdateFunc` ✓
- **No `chiInformer` events at all** ✗

The CHI controller workers started but never received any work items.

**Resolution**: Reinstall from official bundle instead of Helm chart:

```bash
# Install from official bundle (goes to kube-system namespace by default)
kubectl apply -f https://raw.githubusercontent.com/Altinity/clickhouse-operator/0.25.5/deploy/operator/clickhouse-operator-install-bundle.yaml

# Delete the broken Helm-managed operator
kubectl delete deployment clickhouse-operator -n clickhouse-system

# Add CriticalAddonsOnly toleration for system node scheduling
kubectl patch deployment clickhouse-operator -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/tolerations", "value": [{"key": "CriticalAddonsOnly", "operator": "Exists"}]}]'
```

**Verification**:
```bash
# Check operator is processing CHIs (should see ENQUEUE lines)
kubectl logs -n kube-system deployment/clickhouse-operator | grep -E "ENQUEUE.*ReconcileCHI"

# Check CHI status (should not be empty)
kubectl get chi -A

# Check StatefulSets created with correct tolerations
kubectl get sts -n ameide-dev -l clickhouse.altinity.com/chi=data-clickhouse
kubectl get sts <name> -o jsonpath='{.spec.template.spec.tolerations}' | jq .
```

**Key difference**: The official bundle creates informers correctly; the Helm chart deployment had a broken informer initialization path. After fix, CHI dev completed in ~1 minute with correct tolerations propagated to StatefulSet/Pod.

### CNPG Postgres Affinity Configuration

CNPG uses `spec.affinity` (not `spec.tolerations`) on the Cluster CR:

```yaml
# Correct structure per CloudNativePG docs
cluster:
  affinity:
    tolerations:
      - key: "ameide.io/environment"
        value: "dev"
        effect: "NoSchedule"
    nodeSelector:
      ameide.io/pool: dev
```

Reference: https://cloudnative-pg.io/documentation/current/scheduling/
