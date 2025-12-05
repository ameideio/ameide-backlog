# 452 – Observability Stack Namespace Isolation

**Status**: Partial - Prometheus Operator requires additional work
**Created**: 2025-12-05
**Related**: [445-argocd-namespace-isolation.md](445-argocd-namespace-isolation.md), [446-namespace-isolation.md](446-namespace-isolation.md), [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md)

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

### Per-Environment

| File | Change |
|------|--------|
| `sources/values/dev/observability/platform-alloy-logs.yaml` | Added `rbac.namespaces: [ameide-dev]` |
| `sources/values/staging/observability/platform-alloy-logs.yaml` | Added `rbac.namespaces: [ameide-staging]` |
| `sources/values/production/observability/platform-alloy-logs.yaml` | Added `rbac.namespaces: [ameide-prod]` |
| `sources/values/dev/observability/platform-prometheus.yaml` | Added namespace selectors + kube-state-metrics namespace |
| `sources/values/staging/observability/platform-prometheus.yaml` | Added namespace selectors + kube-state-metrics namespace |
| `sources/values/production/observability/platform-prometheus.yaml` | Added namespace selectors + kube-state-metrics namespace |

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

## Remaining Work: Prometheus Operator

The kube-prometheus-stack chart bundles several components with different scope requirements:

| Component | Current | Should Be | Status |
|-----------|---------|-----------|--------|
| Prometheus CRDs | cluster (`cluster-crds-prometheus`) | cluster | ✅ Done |
| Prometheus Operator | per-env (ClusterRole conflicts) | cluster | ❌ Needs work |
| Prometheus Server | per-env | per-env | ✅ Done |
| Alertmanager | per-env | per-env | ✅ Done |
| kube-state-metrics | per-env | per-env | ✅ Done |
| Grafana | per-env (separate chart) | per-env | ✅ Done |

### Prometheus Operator Cluster-Scoped Resources

The Prometheus Operator creates these cluster-scoped resources that cause conflicts:

```
ClusterRole/platform-prometheus-kube-p-operator
ClusterRole/platform-prometheus-kube-p-prometheus
ClusterRoleBinding/platform-prometheus-kube-p-operator
ClusterRoleBinding/platform-prometheus-kube-p-prometheus
MutatingWebhookConfiguration/platform-prometheus-kube-p-admission
ValidatingWebhookConfiguration/platform-prometheus-kube-p-admission
Service/platform-prometheus-kube-p-coredns (kube-dns ServiceMonitor)
```

### Options to Resolve

**Option A: Single Cluster-Scoped Prometheus Operator** (Recommended)
1. Create `cluster-prometheus-operator` application
2. Deploy operator + webhooks to cluster scope (e.g., `prometheus-system` namespace)
3. Disable operator in per-env deployments: `prometheusOperator.enabled: false`
4. Per-env apps deploy only Prometheus server, Alertmanager, kube-state-metrics

**Option B: Unique Names per Environment**
1. Configure unique ClusterRole/ClusterRoleBinding names per environment
2. Requires chart modifications or post-render patches
3. Wastes resources (3 identical operators)

**Option C: ArgoCD Ignore Differences**
1. Configure ArgoCD to ignore cluster-scoped resources
2. Let one environment "own" them
3. Fragile - ownership can shift unexpectedly

### Next Steps

1. Implement Option A by creating `cluster-prometheus-operator` component
2. Add to cluster ApplicationSet (wave 1, with other operators)
3. Update per-env prometheus values: `prometheusOperator.enabled: false`
4. Test and verify

---

## Related Changes

This change is part of the broader namespace isolation effort:

- **445**: ArgoCD namespace isolation
- **446**: Environment namespace isolation (`ameide-dev`, `ameide-staging`, `ameide-prod`)
- **447**: Dual ApplicationSet model (cluster vs environment scope)
- **447**: Third-party chart tolerations
- **448**: Per-environment service verification
- **450**: Envoy Gateway per-environment architecture
