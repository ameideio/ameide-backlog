# 452 – Observability Stack Namespace Isolation

> **Note:** This document (452) covers observability namespace isolation. For Vault RBAC isolation, see [452-vault-rbac-isolation.md](452-vault-rbac-isolation.md).

> **Cross-References (Deployment Architecture Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | Per-environment observability deployment |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Prometheus operator at cluster scope (020) |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Charts at `sources/charts/observability/` |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | OIDC patterns for Grafana |

**Status**: Implemented
**Created**: 2025-12-05
**Related**: [445-argocd-namespace-isolation.md](445-argocd-namespace-isolation.md), [446-namespace-isolation.md](446-namespace-isolation.md), [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md), [454-coredns-cluster-scoped.md](454-coredns-cluster-scoped.md)

---

## Problem Statement

The observability stack (Loki, Grafana, Prometheus, Alloy, kube-state-metrics) was creating **cluster-scoped RBAC resources** (ClusterRole/ClusterRoleBinding) with hardcoded names. This caused:

1. **ArgoCD OutOfSync conflicts**: Multiple environments trying to manage the same cluster-scoped resource
2. **Ownership races**: Whichever environment synced last "owned" the shared resource
3. **Violates isolation principle**: Environments should be fully isolated; shared resources belong to cluster scope

### Symptoms

```bash
kubectl get applications -n argocd | grep -E "loki|grafana|prometheus|alloy"
# dev-platform-loki         OutOfSync
# dev-platform-grafana      OutOfSync
# production-platform-loki  OutOfSync
# etc.
```

ArgoCD conditions showed:
```
ClusterRole/platform-loki-clusterrole is part of applications argocd/dev-platform-loki and staging-platform-loki
```

---

## Design Principle

> **What belongs to shared goes to cluster, what belongs to environments goes to environments.**

For observability:
- Each environment has its **own** Prometheus, Grafana, Loki, Alloy instance
- Each instance should **only** monitor/collect from its own namespace
- No cluster-scoped RBAC needed - use namespace-scoped Roles instead

---

## Solution

Configure all observability components to use **namespace-scoped RBAC** and **namespace-scoped discovery**.

### Component Configuration

| Component | Setting | Effect |
|-----------|---------|--------|
| **Loki** | `rbac.namespaced: true` | Creates Role instead of ClusterRole |
| **Grafana** | `rbac.namespaced: true` | Creates Role instead of ClusterRole |
| **Alloy** | `rbac.namespaces: [ameide-{env}]` | Creates Role only for own namespace |
| **Prometheus** | `*NamespaceSelector.matchLabels` | Only discovers monitors in own namespace |
| **kube-state-metrics** | `namespaces: "ameide-{env}"` | Only collects metrics from own namespace |

### Trade-offs

With full namespace isolation:
- ❌ kube-state-metrics cannot see cluster-scoped resources (nodes, PVs, storageclasses)
- ❌ Alloy cannot collect logs from other namespaces
- ❌ Prometheus cannot scrape metrics from other namespaces
- ✅ Full environment isolation achieved
- ✅ No ArgoCD ownership conflicts
- ✅ Each environment is self-contained

---

## Files Modified

### Shared (applies to all environments)

| File | Change |
|------|--------|
| `sources/values/_shared/observability/platform-loki.yaml` | Added `rbac.namespaced: true` |
| `sources/values/_shared/platform/platform-grafana.yaml` | Added `rbac.namespaced: true` |
| `sources/values/_shared/observability/platform-grafana.yaml` | Added `rbac.namespaced: true` |
| `sources/values/_shared/cluster/prometheus-operator.yaml` | Leaves `prometheusOperator` cluster-scoped but explicitly disables the chart’s `httproute` template so cluster-gateway no longer renders unused traffic resources |

### Per-Environment

| File | Change |
|------|--------|
| `sources/values/env/dev/observability/platform-alloy-logs.yaml` | Added `rbac.namespaces: [ameide-dev]` |
| `sources/values/env/staging/observability/platform-alloy-logs.yaml` | Added `rbac.namespaces: [ameide-staging]` |
| `sources/values/env/production/observability/platform-alloy-logs.yaml` | Added `rbac.namespaces: [ameide-prod]` |
| `sources/values/env/dev/observability/platform-prometheus.yaml` | Added namespace selectors + kube-state-metrics namespace |
| `sources/values/env/staging/observability/platform-prometheus.yaml` | Added namespace selectors + kube-state-metrics namespace |
| `sources/values/env/production/observability/platform-prometheus.yaml` | Added namespace selectors + kube-state-metrics namespace |
| `sources/values/env/local/observability/platform-prometheus.yaml` | Mirrors the same selectors for `ameide-local` and pins local hostnames (`prometheus.local.ameide.io`, `alertmanager.local.ameide.io`) so the local Application matches the isolation contract |
| `sources/values/env/local/observability/platform-alloy-logs.yaml` | Restricts `rbac.namespaces` to `ameide-local`, keeping log collection scoped even when only a single namespace exists |
| `sources/values/env/local/observability/platform-loki.yaml` | Drops the empty `singleBinary` override so local Loki inherits the shared annotations instead of rendering `nil` |

---

## Post-Sync Cleanup

After ArgoCD syncs, the following **orphaned cluster-scoped resources** should be deleted:

```bash
# ClusterRoles (replaced by namespace-scoped Roles)
kubectl delete clusterrole platform-loki-clusterrole
kubectl delete clusterrole platform-grafana-clusterrole
kubectl delete clusterrole alloy-logs
kubectl delete clusterrole platform-prometheus-kube-state-metrics

# ClusterRoleBindings
kubectl delete clusterrolebinding platform-loki-clusterrolebinding
kubectl delete clusterrolebinding platform-grafana-clusterrolebinding
kubectl delete clusterrolebinding alloy-logs
kubectl delete clusterrolebinding platform-prometheus-kube-state-metrics
```

---

## Verification

### Check no cluster-scoped RBAC for observability

```bash
# Should return empty after cleanup
kubectl get clusterrole | grep -E "loki|grafana|alloy|kube-state-metrics"
kubectl get clusterrolebinding | grep -E "loki|grafana|alloy|kube-state-metrics"
```

### Check namespace-scoped Roles created

```bash
# Should show Roles in each environment namespace
kubectl get role -n ameide-dev | grep -E "loki|grafana|alloy"
kubectl get role -n ameide-staging | grep -E "loki|grafana|alloy"
kubectl get role -n ameide-prod | grep -E "loki|grafana|alloy"
```

### Check ArgoCD apps are Synced

```bash
kubectl get applications -n argocd | grep -E "loki|grafana|prometheus|alloy"
# All should show Synced/Healthy
```

## Addendum (2025-12-16): Loki memberlist DNS bootstrap (local readiness)

When running Loki in `SingleBinary` mode with memberlist enabled, local clusters can hit a bootstrap loop:
- `platform-loki-0` readiness returns `503` while Loki is trying to join the memberlist ring.
- The memberlist **headless Service** (`platform-loki-memberlist`) may not have DNS records until endpoints exist; endpoints are only published for **Ready** pods by default.

**Vendor-aligned fix:** set `memberlist.service.publishNotReadyAddresses: true` so the headless Service publishes endpoints before readiness, allowing DNS resolution and ring join to succeed deterministically.

---

## Chart Value Paths Reference

For future reference when adding new observability components:

| Chart | Namespace RBAC Setting |
|-------|----------------------|
| grafana/loki | `rbac.namespaced: true` |
| grafana/grafana | `rbac.namespaced: true` |
| grafana/alloy | `rbac.namespaces: [<namespace>]` |
| prometheus-community/kube-prometheus-stack | `prometheus.prometheusSpec.*NamespaceSelector` |
| prometheus-community/kube-state-metrics | `namespaces: "<namespace>"` |

---

## Prometheus Operator Architecture

The kube-prometheus-stack chart bundles several components with different scope requirements:

| Component | Scope | Deployed By | Status |
|-----------|-------|-------------|--------|
| Prometheus CRDs | cluster | `cluster-crds-prometheus` | ✅ Done |
| Prometheus Operator | cluster | `cluster-prometheus-operator` | ✅ Done |
| Prometheus Server | per-env | `{env}-platform-prometheus` | ✅ Done |
| Alertmanager | per-env | `{env}-platform-prometheus` | ✅ Done |
| kube-state-metrics | per-env | `{env}-platform-prometheus` | ✅ Done |
| Grafana | per-env | `{env}-platform-grafana` | ✅ Done |

### Implementation

The Prometheus Operator is deployed at cluster scope via:
- **Component**: `environments/_shared/components/cluster/operators/prometheus-operator/component.yaml`
- **Values**: `sources/values/_shared/cluster/prometheus-operator.yaml`
- **Namespace**: `prometheus-system`

Per-environment deployments disable the operator:
```yaml
# sources/values/_shared/observability/platform-prometheus.yaml
prometheusOperator:
  enabled: false
```

---

## Operator Pattern for Third-Party Charts

**This is the expected approach for ANY operator from third-party Helm charts.**

### When to Use Cluster Scope

Deploy to cluster scope if the component:
1. Creates **ClusterRole/ClusterRoleBinding** with hardcoded names
2. Creates **MutatingWebhookConfiguration/ValidatingWebhookConfiguration**
3. Creates **CRDs** (always cluster-scoped)
4. Manages resources across **ALL namespaces**

### How to Implement

1. **Create cluster component**: `environments/_shared/components/cluster/operators/<name>/component.yaml`
2. **Create cluster values**: `sources/values/_shared/cluster/<name>.yaml`
3. **Disable in per-env**: Set `<operator>.enabled: false` in shared values
4. **Deploy to dedicated namespace**: e.g., `<name>-system`

### Example: Splitting a Bundled Chart

Many charts bundle operators with workloads (like kube-prometheus-stack). Split them:

```
Chart: kube-prometheus-stack
├── cluster-prometheus-operator (cluster scope)
│   └── prometheusOperator.enabled: true
│   └── prometheus.enabled: false
│   └── alertmanager.enabled: false
│
└── {env}-platform-prometheus (per-env)
    └── prometheusOperator.enabled: false
    └── prometheus.enabled: true
    └── alertmanager.enabled: true
```

### Existing Cluster-Scoped Operators

| Operator | Namespace | Watches |
|----------|-----------|---------|
| cloudnative-pg | cnpg-system | All namespaces for Cluster CRs |
| strimzi | strimzi-system | All namespaces for Kafka CRs |
| external-secrets | external-secrets-system | All namespaces for ExternalSecret CRs |
| keycloak | keycloak-system | All namespaces for Keycloak CRs |
| clickhouse | clickhouse-system | All namespaces for ClickHouseInstallation CRs |
| redis | redis-system | All namespaces for RedisFailover CRs |
| prometheus-operator | prometheus-system | All namespaces for Prometheus/ServiceMonitor CRs |

---

## Related Changes

This change is part of the broader namespace isolation effort:

- **445**: ArgoCD namespace isolation
- **446**: Environment namespace isolation (`ameide-dev`, `ameide-staging`, `ameide-prod`)
- **447**: Dual ApplicationSet model (cluster vs environment scope)
- **447**: Third-party chart tolerations
- **448**: Per-environment service verification
- **450**: Envoy Gateway per-environment architecture
