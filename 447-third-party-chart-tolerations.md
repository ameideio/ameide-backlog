# 447 – Third-Party Chart Tolerations

> **Note:** This document (447) covers chart tolerations. For rollout phases and cluster-scoped operators, see [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md).

> **Cross-References (Deployment Architecture Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | Per-environment deployment model |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Node tolerations for operators |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Third-party chart locations |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | Keycloak tolerations (§5.1) |
> | [490-vendor-charts-alignment.md](490-vendor-charts-alignment.md) | Vendoring workflow that keeps third-party charts reproducible before tolerations are applied |
>
> **Related documents:**
> - [442-environment-isolation.md](442-environment-isolation.md) – Environment isolation strategy (parent issue)
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Namespace isolation
> - [240-cluster-rightsizing.md](240-cluster-rightsizing.md) – Cluster resource planning
> - [452-observability-namespace-isolation.md](452-observability-namespace-isolation.md) – Observability RBAC namespace isolation
> - [456-ghcr-mirror.md](456-ghcr-mirror.md) – GHCR image mirroring for Docker Hub rate limits

## Summary

This document tracks the comprehensive effort to add tolerations and nodeSelector to all third-party Helm charts. While custom application charts inherit tolerations from `globals.yaml`, third-party charts require explicit configuration at chart-specific value paths.

**Status**: ✅ Complete (2025-12-05)

---

## Problem Statement

After deploying the 4-pool AKS strategy (system/dev/staging/prod), many pods remained in `Pending` state because:

1. **Third-party charts don't inherit globals.yaml**: Unlike custom app charts that use `{{ .Values.tolerations }}`, third-party charts have their own value structures
2. **Chart-specific value paths**: Each chart defines tolerations at different paths (e.g., `controller.tolerations`, `singleBinary.tolerations`, `deployment.pod.tolerations`)
3. **Subcomponents missed**: Charts with multiple components (operators, workers, web UIs) require tolerations at each subcomponent path
4. **Environment files missing**: Staging and production had fewer overrides than dev, causing inconsistent behavior

### Symptoms

```bash
# Pods stuck in Pending
kubectl get pods -n ameide-dev | grep Pending

# Example output before fix:
alloy-logs-89f774c98-k6vts           0/2     Pending   0     15h
platform-loki-0                       0/2     Pending   0     15h
envoy-gateway-76999f877b-4m54s       0/1     Pending   0     15h
platform-langfuse-web-...            0/1     Pending   0     15h
```

### Root Cause

Pods without `tolerations` for `ameide.io/environment` cannot be scheduled on tainted node pools:

```bash
# Node taints (example: dev pool)
kubectl describe node aks-dev-21365390-vmss000000 | grep Taints
Taints: ameide.io/environment=dev:NoSchedule
```

---

## Solution Overview

For each third-party chart:

1. **Identify the correct tolerations path** by reading the chart's `values.yaml`
2. **Create/update per-environment values files** at `sources/values/{dev,staging,production}/...`
3. **Apply consistent pattern**:
   ```yaml
   # Pattern for most charts
   <component>:
     tolerations:
       - key: "ameide.io/environment"
         value: "<env>"        # dev, staging, or production
         effect: "NoSchedule"
     nodeSelector:
       ameide.io/pool: <pool>  # dev, staging, or prod
   ```

---

## Charts Fixed

### 1. Grafana Alloy (Logs)

**Chart**: `grafana/alloy` (sources/charts/third_party/grafana/alloy)

**Issue**: Values were at `alloy.tolerations` but chart expects `controller.tolerations`

**Correct path**:
```yaml
# sources/values/env/{env}/observability/platform-alloy-logs.yaml
controller:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: dev
```

**Files modified**:
- `sources/values/env/dev/observability/platform-alloy-logs.yaml` (fixed path)
- `sources/values/env/staging/observability/platform-alloy-logs.yaml` (created)
- `sources/values/env/production/observability/platform-alloy-logs.yaml` (created)

---

### 2. Grafana Loki

**Chart**: `grafana/loki` (sources/charts/third_party/grafana/loki)

**Issue**: Values were at `loki.tolerations` but chart expects different paths for different modes

**Correct paths** (for singleBinary mode):
```yaml
# sources/values/env/{env}/observability/platform-loki.yaml
singleBinary:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: dev

gateway:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: dev
```

**Files modified**:
- `sources/values/env/dev/observability/platform-loki.yaml` (fixed path)
- `sources/values/env/staging/observability/platform-loki.yaml` (created)
- `sources/values/env/production/observability/platform-loki.yaml` (created)

---

### 3. Envoy Gateway

**Chart**: `envoy-gateway/gateway-helm` (sources/charts/third_party/envoy-gateway)

**Issue**: Values were at `deployment.envoyGateway.tolerations` but chart expects `deployment.pod.tolerations`

**Correct path**:
```yaml
# sources/values/env/{env}/platform/platform-envoy-gateway.yaml
deployment:
  pod:
    tolerations:
      - key: "ameide.io/environment"
        value: "dev"
        effect: "NoSchedule"
    nodeSelector:
      ameide.io/pool: dev
```

**Files modified**:
- `sources/values/env/dev/platform/platform-envoy-gateway.yaml` (fixed path)
- `sources/values/env/staging/platform/platform-envoy-gateway.yaml` (created)
- `sources/values/env/production/platform/platform-envoy-gateway.yaml` (created)

---

### 4. Temporal

**Chart**: `temporal/temporal` (sources/charts/third_party/temporal)

**Issue**: No environment-specific values files existed for staging/production. Temporal has many subcomponents.

**Correct paths** (multiple components):
```yaml
# sources/values/env/{env}/data/data-temporal.yaml
server:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: dev
  frontend:
    tolerations: [...]
    nodeSelector: {...}
  history:
    tolerations: [...]
    nodeSelector: {...}
  matching:
    tolerations: [...]
    nodeSelector: {...}
  worker:
    tolerations: [...]
    nodeSelector: {...}

admintools:
  tolerations: [...]
  nodeSelector: {...}

web:
  tolerations: [...]
  nodeSelector: {...}
```

**Files modified**:
- `sources/values/env/dev/data/data-temporal.yaml` (already existed, verified correct)
- `sources/values/env/staging/data/data-temporal.yaml` (created)
- `sources/values/env/production/data/data-temporal.yaml` (created)

---

### 5. Langfuse

**Chart**: `langfuse/langfuse` (sources/charts/third_party/langfuse)

**Issue**: No environment-specific values files existed for dev.

**Correct path**:
```yaml
# sources/values/env/{env}/observability/platform-langfuse.yaml
langfuse:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: dev
```

**Files modified**:
- `sources/values/env/dev/observability/platform-langfuse.yaml` (created - was using _shared only)
- `sources/values/env/staging/observability/platform-langfuse.yaml` (created)
- `sources/values/env/production/observability/platform-langfuse.yaml` (created)

**Image Path Note**: The Langfuse chart expects image repositories at:
- `langfuse.web.image.repository` (for web pods)
- `langfuse.worker.image.repository` (for worker pods)

NOT at `langfuse.image.repository` (which only sets global tag/pullSecrets). Fixed in commit `8ffc0e0`.

---

### 6. Prometheus (kube-prometheus-stack)

**Chart**: `prometheus-community/kube-prometheus-stack` (sources/charts/third_party/prometheus)

**Issue**: Multiple subcomponents, and prometheus-node-exporter is a DaemonSet requiring special handling

**Correct paths**:
```yaml
# sources/values/env/{env}/observability/platform-prometheus.yaml
prometheus:
  prometheusSpec:
    tolerations:
      - key: "ameide.io/environment"
        value: "dev"
        effect: "NoSchedule"
    nodeSelector:
      ameide.io/pool: dev

alertmanager:
  alertmanagerSpec:
    tolerations: [...]
    nodeSelector: {...}

prometheusOperator:
  tolerations: [...]
  nodeSelector: {...}

kube-state-metrics:
  tolerations: [...]
  nodeSelector: {...}

grafana:
  tolerations: [...]
  nodeSelector: {...}

# IMPORTANT: Each environment runs node-exporter on its own nodes only
# Using operator: "Exists" across all environments causes hostPort 9100 conflicts
prometheus-node-exporter:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: dev
```

**DaemonSet Correction**: Originally documented as using `operator: "Exists"` to run on all nodes. This causes **hostPort 9100 conflicts** when multiple environments deploy node-exporter to the same cluster. Each environment's node-exporter must use `nodeSelector` to run only on its own node pool. Fixed in commit `1beb8cc`.

**Files modified**:
- `sources/values/env/dev/observability/platform-prometheus.yaml` (added node-exporter)
- `sources/values/env/staging/observability/platform-prometheus.yaml` (created)
- `sources/values/env/production/observability/platform-prometheus.yaml` (created)

---

### 7. Plausible

**Chart**: `plausible/plausible` (sources/charts/platform-layers/plausible)

**Issue**: Chart accepts tolerations at root level, but no environment files existed.

**Correct path**:
```yaml
# sources/values/env/{env}/apps/plausible.yaml
tolerations:
  - key: "ameide.io/environment"
    value: "dev"
    effect: "NoSchedule"
nodeSelector:
  ameide.io/pool: dev
```

**Files modified**:
- `sources/values/env/dev/apps/plausible.yaml` (created)
- `sources/values/env/staging/apps/plausible.yaml` (created)
- `sources/values/env/production/apps/plausible.yaml` (created)

---

### 8. Strimzi Kafka Operator

**Chart**: `strimzi/strimzi-kafka-operator` (sources/charts/third_party/strimzi)

**Issue**: Dev file was minimal placeholder. Chart accepts tolerations at root level.

**Correct path**:
```yaml
# sources/values/env/{env}/data/foundation-strimzi-operator.yaml
tolerations:
  - key: "ameide.io/environment"
    value: "dev"
    effect: "NoSchedule"
nodeSelector:
  ameide.io/pool: dev
```

**Files modified**:
- `sources/values/env/dev/data/foundation-strimzi-operator.yaml` (updated from placeholder)
- `sources/values/env/staging/data/foundation-strimzi-operator.yaml` (created)
- `sources/values/env/production/data/foundation-strimzi-operator.yaml` (created)

---

### 9. Kafka Cluster (Strimzi CR)

**Chart**: Custom chart wrapping Strimzi Kafka CR (sources/charts/data/kafka-cluster)

**Issue**: No staging/production values files existed. Strimzi uses `template.pod.tolerations` inside the Kafka CR.

**Correct paths** (Strimzi CR format):
```yaml
# sources/values/env/{env}/data/data-kafka-cluster.yaml
kafka:
  template:
    pod:
      tolerations:
        - key: "ameide.io/environment"
          value: "dev"
          effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: ameide.io/pool
                    operator: In
                    values:
                      - dev

zookeeper:
  template:
    pod:
      tolerations: [...]
      affinity: {...}

entityOperator:
  template:
    pod:
      tolerations: [...]
      affinity: {...}
```

**Note**: Strimzi CRs use `affinity.nodeAffinity` instead of `nodeSelector` for pod placement.

**Files modified**:
- `sources/values/env/dev/data/data-kafka-cluster.yaml` (already existed, verified correct)
- `sources/values/env/staging/data/data-kafka-cluster.yaml` (created)
- `sources/values/env/production/data/data-kafka-cluster.yaml` (created)

---

### 10. Redis Failover (Spotahome Operator)

**Chart**: `spotahome/redis-operator` RedisFailover CR (sources/charts/data/redis-failover)

**Issue**: Staging and production had no environment-specific values files. Redis and Sentinel pods remained Pending.

**Correct paths**:
```yaml
# sources/values/env/{env}/data/data-redis-failover.yaml
redis:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: dev

sentinel:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: dev
```

**Files modified**:
- `sources/values/env/dev/data/data-redis-failover.yaml` (already existed)
- `sources/values/env/staging/data/data-redis-failover.yaml` (created)
- `sources/values/env/production/data/data-redis-failover.yaml` (created)

---

### 11. Langfuse Bootstrap

**Chart**: Custom chart (sources/charts/platform-layers/langfuse-bootstrap)

**Issue**: Bootstrap job (ArgoCD PostSync hook) had no tolerations, causing jobs to stay Pending in all environments.

**Correct path**:
```yaml
# sources/values/env/{env}/platform/platform-langfuse-bootstrap.yaml
tolerations:
  - key: "ameide.io/environment"
    value: "dev"
    effect: "NoSchedule"
nodeSelector:
  ameide.io/pool: dev
```

**Files modified**:
- `sources/values/env/dev/platform/platform-langfuse-bootstrap.yaml` (updated)
- `sources/values/env/staging/platform/platform-langfuse-bootstrap.yaml` (created)
- `sources/values/env/production/platform/platform-langfuse-bootstrap.yaml` (created)

---

### 12. Keycloak (Operator-Managed)

**Chart**: Custom chart wrapping Keycloak operator CR (sources/charts/foundation/operators-config/keycloak_instance)

**Issue**: Realm import jobs (created by Keycloak operator) were pending because tolerations in `unsupported.podTemplate` don't propagate to jobs.

**Root cause**: Per [Keycloak operator docs](https://www.keycloak.org/operator/advanced-configuration):
- Import jobs **inherit** from `spec.scheduling` on the Keycloak CR
- `spec.unsupported.podTemplate` applies **only to server pods**, NOT import jobs

**Correct paths** (unique dual-path requirement):
```yaml
# sources/values/env/{env}/platform/platform-keycloak.yaml

# Import jobs inherit from spec.scheduling
scheduling:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"

# Server pods use unsupported.podTemplate for nodeSelector
# (spec.scheduling.nodeSelector not yet supported by operator)
unsupported:
  podTemplate:
    spec:
      nodeSelector:
        ameide.io/pool: dev
```

**Files modified**:
- `sources/values/env/dev/platform/platform-keycloak.yaml`
- `sources/values/env/staging/platform/platform-keycloak.yaml`
- `sources/values/env/production/platform/platform-keycloak.yaml`

**Commit**: `8b85655`

> **Cross-reference**: See [426-keycloak-config-map.md](426-keycloak-config-map.md) §5.1 for import job lifecycle details.

---

## Commits

| Commit | Description |
|--------|-------------|
| `fdf9593` | fix(tolerations): add tolerations to third-party chart components |
| `1beb8cc` | fix(prometheus): restrict node-exporter to environment-specific nodes |
| `8ffc0e0` | fix(langfuse): correct image path for GHCR mirror |
| `dbb8dd1` | fix(tolerations): add tolerations for redis-failover and langfuse-bootstrap |
| `8b85655` | fix(keycloak): add spec.scheduling for staging/production import jobs |

---

## Verification

### Before Fix

```bash
kubectl get pods -n ameide-dev | grep Pending | wc -l
# 17+ pods pending
```

### After Fix

```bash
# Pods now scheduling on correct node pools
kubectl get pods -n ameide-dev -o wide | grep alloy
alloy-logs-668dbb8995-cpwrs   2/2   Running   aks-dev-21365390-vmss000001

kubectl get pods -n ameide-dev -o wide | grep loki
platform-loki-0   2/2   Running   aks-dev-21365390-vmss000001

# Verify tolerations applied
kubectl get pod platform-langfuse-web-67b8c44785-pxcph -n ameide-dev -o yaml | grep -A3 tolerations
tolerations:
  - effect: NoSchedule
    key: ameide.io/environment
    value: dev
```

---

## Reference: Finding Correct Tolerations Paths

When adding a new third-party chart, follow this process:

### Step 1: Find the chart's values.yaml

```bash
# Example for Alloy
cat sources/charts/third_party/grafana/alloy/*/values.yaml | grep -A5 tolerations
```

### Step 2: Identify the correct path

Common patterns:
| Chart Type | Typical Path |
|------------|--------------|
| Simple deployment | `tolerations:` (root) |
| Controller-based | `controller.tolerations:` |
| Multi-component | `<component>.tolerations:` per component |
| Nested spec | `<component>.<spec>.tolerations:` |
| CRD-based (Strimzi) | `<resource>.template.pod.tolerations:` |

### Step 3: Create per-environment files

```bash
# Template for new chart
for env in dev staging production; do
  cat > sources/values/$env/<layer>/<chart>.yaml << 'EOF'
# ${env^}-specific values for <chart>

# Tolerations for $env node pool - see backlog/442-environment-isolation.md
<path>:
  tolerations:
    - key: "ameide.io/environment"
      value: "$env"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: $pool
EOF
done
```

---

## Outstanding Items

### Not Applicable / Already Handled

| Chart | Status | Notes |
|-------|--------|-------|
| MinIO | ✅ Fixed in 442 | `console.tolerations`, `provisioning.tolerations` |
| ClickHouse | ✅ Fixed in 442 | `clickhouse.tolerations` |
| CNPG Postgres | ✅ Fixed in 442 | `cluster.affinity.tolerations` |
| Custom app charts (14) | ✅ Inherit | Use `{{ .Values.tolerations }}` from globals.yaml |

### Future Considerations

1. **New third-party charts**: When adding new charts, always check tolerations path
2. **Chart upgrades**: Major version upgrades may change tolerations paths
3. **DaemonSets with hostPort**: Restrict to per-environment nodeSelector to avoid port conflicts (e.g., prometheus-node-exporter uses hostPort 9100)
4. **GHCR mirrors**: Docker Hub images should be mirrored to GHCR to avoid rate limiting - see [456-ghcr-mirror.md](456-ghcr-mirror.md)

---

## Related Issues

- Temporal pods in CrashLoopBackOff are **not** toleration issues - they are application-level errors
- ClickHouse operator reconciliation issues are tracked in [442-environment-isolation.md](442-environment-isolation.md#altinity-clickhouse-operator-issue-2025-12-05)
