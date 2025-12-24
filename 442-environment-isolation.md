# 442 – Environment Dev/Staging/Prod Isolation

> **Related documents:**
> - [443-tenancy-models.md](443-tenancy-models.md) – **Tenancy model overview (single source of truth)**
> - [440-storage-concerns.md](440-storage-concerns.md) – Storage improvements
> - [441-networking.md](441-networking.md) – Networking improvements
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) – Environment naming conventions
> - [435-remote-first-development.md](435-remote-first-development.md) – Remote-first development
> - [240-cluster-rightsizing.md](240-cluster-rightsizing.md) – Cluster resource planning
> - [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md) – Implementation status tracking
> - [451-secrets-management.md](451-secrets-management.md) – Per-environment secrets architecture

## Implementation Status (2025-12-05)

| Component | Status | Notes |
|-----------|--------|-------|
| **Bicep 4-pool strategy** | ✅ Done | `infra/bicep/managed-application/modules/aks.bicep` - system/dev/staging/prod pools |
| **Bicep per-env Envoy IPs** | ✅ Done | 4 public IPs (ArgoCD + 3 env-specific Envoys) matching Terraform |
| **IaC output parity** | ✅ Done | 23 normalized snake_case outputs (test via `test-iac-consistency.sh`) |
| **Terraform remote state** | ✅ Done | `use_azuread_auth` backend enabled in `versions.tf` |
| **Service tier labels** | ✅ Done | `tier` field in 20+ values files (`_shared/apps/*.yaml`, `_shared/data/*.yaml`) |
| **Pod tier labels** | ✅ Done | `tierLabels` helper in 14 app charts (`_helpers.tpl` + `deployment.yaml`) |
| **Environment state field** | ✅ Done | `environmentState: "on"` in all 3 environment `globals.yaml` |
| **Node affinity config** | ✅ Done | `nodeSelector` + `tolerations` in all 3 environment `globals.yaml` |
| **Deployment scheduling** | ✅ Done | All 14 app deployment templates use `nodeSelector`/`tolerations` from values |
| **Remove empty overrides** | ✅ Done | Fixed `6c805de` - removed `nodeSelector: {}` and `tolerations: []` from 11 shared values that blocked globals.yaml inheritance |
| **Cross-env NetworkPolicy** | ✅ Done | `deny-cross-environment` policy in `foundation-namespaces.yaml` |
| **3rd-party chart subcomponents** | ✅ Done | Fixed `9e54a78` - added `console.tolerations`, `provisioning.tolerations` for Bitnami MinIO |
| **Data layer tolerations** | ✅ Done | Added tolerations to CNPG Postgres and ClickHouse values (see below) |
| **Deploy node pools** | ✅ Done | 4 pools deployed: system(2), dev(3), staging(2), prod(3) - verified 2025-12-05 |

### Data Layer Tolerations (2025-12-05)

Added tolerations/nodeSelector to data layer components for environment isolation:

| Component | Files | Commit | Notes |
|-----------|-------|--------|-------|
| **CNPG Postgres** | `sources/values/{dev,staging,production}/data/platform-postgres-clusters.yaml` | `1b622e9` | Uses `cluster.affinity.tolerations` |
| **ClickHouse** | `sources/values/{dev,staging,production}/data/data-clickhouse.yaml` | `b0e63b5` | Uses `clickhouse.tolerations` |
| **Keycloak** | `sources/values/{dev,staging,production}/platform/platform-keycloak.yaml` | `8b85655` | Uses `spec.scheduling` for import jobs (NOT `unsupported.podTemplate`) - see [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md) §12 |

**Postgres structure** (uses CNPG `cluster.affinity` format):
```yaml
cluster:
  affinity:
    tolerations:
      - key: "ameide.io/environment"
        value: "dev"  # or staging/production
        effect: "NoSchedule"
    nodeSelector:
      ameide.io/pool: dev  # or staging/prod
```

**ClickHouse structure** (uses Altinity chart `clickhouse.tolerations` format):
```yaml
clickhouse:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"
  nodeSelector:
    ameide.io/pool: dev
```

### Altinity ClickHouse Operator Issue (2025-12-05)

**Problem**: Altinity ClickHouse operator v0.25.5 is not reconciling CHI resources. Investigation shows:

- CHI specs have correct tolerations/nodeSelector in `spec.templates.podTemplates[*].spec`
- Operator logs show `podInformer`, `configMapInformer`, `serviceInformer` activity
- **No `chiInformer` events are logged** - the CHI informer isn't watching resources
- CHI status remains empty (never processed)
- StatefulSets are not created

**Root cause**: Suspected bug in Altinity operator v0.25.5 where the CHI informer factory isn't properly initialized.

**Workaround options**:
1. **Wait for fix**: Monitor Altinity releases for v0.25.6+ with informer fixes
2. **Manual StatefulSet patch** (temporary):
   ```bash
   # Create StatefulSet manually based on CHI spec (not recommended)
   kubectl patch sts <name> -n <ns> --type merge -p '{"spec":{"template":{"spec":{"tolerations":[...]}}}}'
   ```
3. **Downgrade operator**: Try earlier version (e.g., 0.24.x) if informer worked there

**Verification commands**:
```bash
# Check if operator is receiving CHI events
kubectl logs -n clickhouse-system deployment/clickhouse-operator -c altinity-clickhouse-operator | grep -i "chiInformer"

# Check CHI status (should show status, not empty)
kubectl get chi -n ameide-dev data-clickhouse -o jsonpath='{.status.status}'
```

### Secret Management (2025-12-05)

Per-environment unique passwords implemented using Helm lookup+randAlphaNum pattern:
- Template: `sources/charts/foundation/operators-config/postgres_clusters/templates/app-secrets.yaml`
- Commit: `d10ab77`

**How it works:**
1. First install: Helm generates random 32-char passwords for all Postgres users
2. Subsequent upgrades: Helm `lookup` function finds existing secrets and preserves passwords
3. Each namespace (dev/staging/prod) gets unique passwords automatically

This achieves environment isolation for database secrets without requiring Vault path migration. See [412-cnpg-owned-postgres-greds.md](412-cnpg-owned-postgres-greds.md) for the full CNPG credential ownership model.

### GitOps Implementation Complete ✅

All Helm chart and values file changes for environment isolation are complete. The remaining work is:
1. Deploy the 4-pool AKS configuration via Bicep (`az deployment`)
2. Verify pods receive correct labels after ArgoCD sync
3. Test environment shutdown procedure

## Problem Statement

Need clear isolation and differentiation between environments:

1. **No compute isolation**: All environments share single node pool
2. **No per-environment shutdown**: Cannot turn off dev/staging independently for cost savings
3. **Missing service labels**: Pods lack `ameide.io/tier` labels for NetworkPolicy
4. **Bicep template conflict**: Two AKS modules with different VM SKUs
5. **Resource allocation**: All environments use similar resource configs
6. **High availability**: Production runs single replicas like dev
7. **Scaling configuration**: HPA disabled everywhere
8. **No environment state management**: No GitOps flag to scale workloads to zero

## Tenancy Models (How This Applies)

All mechanisms described in this document (node pools, environmentState, HPA, PDBs) apply to three deployment models (see [443-tenancy-models.md](443-tenancy-models.md)):

| Tenancy Model | Cluster | Environment Isolation | Node Pool Usage |
|---------------|---------|----------------------|-----------------|
| **Shared SaaS** | `ameide` cluster | `ameide-dev`, `ameide-staging`, `ameide-prod` | Each env uses matching pool |
| **Namespace-per-tenant** | Shared `ameide` cluster | `tenant-<id>-prod` namespaces | Share **prod** pool with `ameide-prod` |
| **Private cloud** | Dedicated `ameide-<id>` cluster | Same 4-pool pattern per tenant | Tenant-owned pools |

**Key points:**

- For **Shared SaaS**, environment isolation is between `ameide-dev`, `ameide-staging`, `ameide-prod` on separate node pools.
- For **Namespace-per-tenant**, each `tenant-<id>-prod` namespace behaves like a small production environment that shares the **prod node pool** with `ameide-prod`. Tenants are isolated via namespaces and NetworkPolicies, not dedicated pools.
- For **Private cloud**, the same 4-pool strategy is applied inside each tenant cluster.

> **Premium tenants**: Large namespace-per-tenant customers MAY receive dedicated node pools
> (e.g., `tenant-acme`) as an upsell option. This extends the 4-pool strategy rather than replacing it.

## Current State (Verified 2025-12-04)

### AKS Cluster Configuration

```bash
az aks show --name ameide --resource-group Ameide
az aks nodepool list --cluster-name ameide --resource-group Ameide
```

| Property | Documented | Actual | Gap |
|----------|-----------|--------|-----|
| **VM SKU** | D4as (prod) / D2as (dev) | `Standard_D8as_v6` | ⚠️ Over-provisioned |
| **Node Count** | 3-4 (prod) / 1 (dev) | 1 | Matches dev |
| **Autoscaling** | Yes (prod) | No | Missing for prod |
| **K8s Version** | 1.32.7 | 1.32.7 | ✅ |

### Bicep Template Conflict

Two AKS Bicep modules exist with **different configurations**:

| File | Non-Prod SKU | Prod SKU | Status |
|------|--------------|----------|--------|
| `azure/managed-application/modules/aks.bicep` | D2as_v6 | D4as_v6 | ✅ Aligned with 240 |
| `infra/bicep/managed-application/modules/aks.bicep` | D8as_v6 | D8as_v6 | ⚠️ **Deployed** (over-sized) |

> **Action Required**: Consolidate to single Bicep source. The `azure/` version follows 240 recommendations.

### Environment Matrix

| Aspect | Dev | Staging | Production |
|--------|-----|---------|------------|
| **Namespace** | ameide-dev | ameide-staging | ameide-prod |
| **Domain** | dev.ameide.io | staging.ameide.io | ameide.io |

### Namespace Conventions (with Tenants)

Internal environments on shared cluster:
- `ameide-dev`, `ameide-staging`, `ameide-prod`

Namespace-per-tenant:
- `tenant-<slug>-prod` (and optionally `tenant-<slug>-staging`)

All namespaces are labeled:
- `ameide.io/environment`: `dev` | `staging` | `production`
- `ameide.io/tenant-kind`: `shared` | `namespace` | `private`
- `ameide.io/tenant-id`: `<slug>` (for tenant namespaces only)

> **Node affinity**: Namespace-per-tenant workloads in `tenant-*-prod` default to
> `nodeSelector: ameide.io/pool=prod` and tolerate the `ameide.io/environment=production`
> taint, just like `ameide-prod`.

### Resource Configuration by Environment

| Aspect | Dev | Staging | Production |
|--------|-----|---------|------------|
| **App Replicas** | 1 | 1 | 1 ⚠️ |
| **HPA Enabled** | No | No | No ⚠️ |
| **PDB Configured** | No | No | No ⚠️ |
| **CPU Requests** | 150m | 200m | 250m |
| **Memory Requests** | 256Mi | 256Mi | 512Mi |
| **Image Pull Policy** | Always | IfNotPresent | IfNotPresent |
| **Log Level** | debug | info | info |

### Pod Labels (Current)

```bash
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name,TIER:.metadata.labels.ameide\.io/tier'
```

| Namespace | Has `ameide.io/tier` label? | Has `ameide.io/domain` label? |
|-----------|----------------------------|------------------------------|
| ameide-dev | ❌ No | ❌ No |
| ameide-staging | ❌ No | ❌ No |
| ameide-prod | ❌ No | ❌ No |

> **Blocker**: Tier-based NetworkPolicies (441) require these labels on pods.

### Data Layer by Environment

| Component | Dev | Staging | Production |
|-----------|-----|---------|------------|
| PostgreSQL Instances | 3 | 3 | 2 |
| PostgreSQL Storage | 10Gi | 10Gi | 200Gi |
| Redis Replicas | 3+3 | 3+3 | 3+3 |
| MinIO | disabled | 50Gi | 100Gi |
| ClickHouse | 20Gi | 20Gi | 20Gi |

### Key Files

- `sources/values/env/dev/globals.yaml`
- `sources/values/env/staging/globals.yaml`
- `sources/values/env/production/globals.yaml`
- `sources/values/_shared/apps/*.yaml`
- `argocd/applicationsets/ameide.yaml`
- `azure/managed-application/modules/aks.bicep` ← Correct sizing
- `infra/bicep/managed-application/modules/aks.bicep` ← Currently deployed

---

## Proposed Changes

### 0. Consolidate Bicep Templates

**Priority**: HIGH (Technical Debt)

Remove duplicate Bicep module and align with 240 recommendations:

**Option A**: Use `azure/managed-application/` as canonical source
- Already has correct D4as/D2as sizing
- Delete `infra/bicep/managed-application/` or symlink

**Option B**: Update `infra/bicep/managed-application/` to match
- Change `vmSize: 'Standard_D8as_v6'` → conditional D4as/D2as

**Recommended Bicep configuration:**

```bicep
var systemAgentPoolBase = {
  name: 'system'
  vmSize: isProduction ? 'Standard_D4as_v6' : 'Standard_D2as_v6'
  // ... rest unchanged
}
```

---

### 1. Node Pool Strategy

**Priority**: HIGH

#### AKS Platform Constraints

From [Azure documentation](https://learn.microsoft.com/en-us/azure/aks/use-system-pools):

- **System pool**: Must always have ≥1 node (best practice: ≥2)
- **User pools**: Can scale to **0 nodes** - this enables per-environment shutdown
- **Cluster stop**: Freezes ALL environments, not selective

#### Option A: Shared Non-Prod Pool (3 pools)

Simpler, but dev and staging shut down together:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  system      │ D2as_v6 × 2  │ Taints: CriticalAddonsOnly                │
│              │              │ Workloads: argocd, cert-manager, envoy    │
├─────────────────────────────────────────────────────────────────────────┤
│  nonprod     │ D2as_v6 × 0-2│ Labels: ameide.io/pool=nonprod            │
│              │ (can be 0)   │ Workloads: ameide-dev + ameide-staging    │
├─────────────────────────────────────────────────────────────────────────┤
│  prod        │ D4as_v6 × 3-4│ Labels: ameide.io/pool=prod               │
│              │ (autoscale)  │ Taints: ameide.io/environment=production  │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Option B: Per-Environment Pools (4 pools) ← **Recommended for independent shutdown**

Each environment can be shut down independently:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  system      │ D4as_v6 × 2  │ Taints: CriticalAddonsOnly                │
│              │ (always on)  │ Workloads: argocd, cert-manager, envoy,   │
│              │              │   CNPG/redis/kafka operators              │
├─────────────────────────────────────────────────────────────────────────┤
│  dev         │ D2as_v6 × 0-2│ Labels: ameide.io/pool=dev                │
│              │ (can be 0)   │ Workloads: ameide-dev only                │
├─────────────────────────────────────────────────────────────────────────┤
│  staging     │ D2as_v6 × 0-2│ Labels: ameide.io/pool=staging            │
│              │ (can be 0)   │ Workloads: ameide-staging only            │
├─────────────────────────────────────────────────────────────────────────┤
│  prod        │ D4as_v6 × 3-8│ Labels: ameide.io/pool=prod               │
│              │ (autoscale)  │ Taints: ameide.io/environment=production  │
│              │              │ Workloads: ameide-prod only               │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insight**: User pools can scale to 0 while system pool stays up. PVs (Azure Disks) remain attached to the scale set; when you scale up again, pods reattach.

#### Addendum (2025-12-17): Avoid tainting *all* user pools

We observed that tainting every AKS user pool (`ameide.io/environment=<env>:NoSchedule`) breaks vendor operators that create Pods/Jobs without configurable tolerations/nodeSelectors (e.g., Temporal operator schema setup Jobs). If **all** pools are tainted (including the system pool with `CriticalAddonsOnly=true:NoSchedule`), these operator-managed Jobs can become permanently `Pending`, keeping Argo apps stuck `Progressing/Degraded`.

**Policy adjustment (vendor-aligned):**
- Keep `CriticalAddonsOnly` taint on the system pool if desired.
- Do **not** taint dev/staging pools by default; rely on labels + `nodeSelector`/affinity in workloads we control.
- If production isolation requires taints, validate that all operator-managed workloads can tolerate them (or keep an untainted pool available for setup Jobs).

#### Bicep Implementation (Option B)

**File**: `azure/managed-application/modules/aks.bicep`

```bicep
// System pool - always on, runs cluster infrastructure
var systemPool = {
  name: 'system'
  vmSize: 'Standard_D4as_v6'
  count: 2
  mode: 'System'
  osDiskSizeGB: 128
  maxPods: 110
  nodeTaints: ['CriticalAddonsOnly=true:NoSchedule']
}

// Dev pool - can scale to 0
resource devPool 'Microsoft.ContainerService/managedClusters/agentPools@2024-05-01' = {
  parent: cluster
  name: 'dev'
  properties: {
    vmSize: 'Standard_D2as_v6'
    count: 0  // Start at 0, scale up when needed
    enableAutoScaling: true
    minCount: 0
    maxCount: 2
    mode: 'User'
    nodeLabels: {
      'ameide.io/pool': 'dev'
      'ameide.io/environment': 'dev'
    }
  }
}

// Staging pool - can scale to 0
resource stagingPool 'Microsoft.ContainerService/managedClusters/agentPools@2024-05-01' = {
  parent: cluster
  name: 'staging'
  properties: {
    vmSize: 'Standard_D2as_v6'
    count: 0
    enableAutoScaling: true
    minCount: 0
    maxCount: 2
    mode: 'User'
    nodeLabels: {
      'ameide.io/pool': 'staging'
      'ameide.io/environment': 'staging'
    }
  }
}

// Prod pool - always has minimum nodes
resource prodPool 'Microsoft.ContainerService/managedClusters/agentPools@2024-05-01' = {
  parent: cluster
  name: 'prod'
  properties: {
    vmSize: 'Standard_D4as_v6'
    count: 3
    enableAutoScaling: true
    minCount: 3
    maxCount: 8
    mode: 'User'
    nodeLabels: {
      'ameide.io/pool': 'prod'
      'ameide.io/environment': 'production'
    }
    nodeTaints: ['ameide.io/environment=production:NoSchedule']
  }
}
```

#### Cost Comparison

| Config | Always-On Nodes | Monthly Est. | Notes |
|--------|-----------------|--------------|-------|
| Current (D8as × 1) | 1 | ~$280 | Over-provisioned |
| Option A (3 pools) | system×2 + prod×3 = 5 | ~$490 | Dev+staging share |
| Option B (4 pools) | system×2 + prod×3 = 5 | ~$490 | Dev+staging at 0 nights/weekends |
| Option B (all on) | system×2 + dev×1 + staging×1 + prod×3 = 7 | ~$630 | All environments running |

> **Cost savings**: With Option B, turning off dev+staging nights/weekends saves ~$140/month (2 D2as nodes × ~14 hours/day × 20 days).

---

### 2. Service Tier Labels

**Priority**: HIGH (Required for 441 NetworkPolicies)

Add `ameide.io/tier` label to all pods via Helm charts.

**Label taxonomy:**

| Tier | Services | Source |
|------|----------|--------|
| `frontend` | www-ameide, www-ameide-platform | `apps/web/*` |
| `backend` | graph, inference, platform, agents, workflows, threads | `apps/core/*`, `apps/runtime/*` |
| `data` | postgres, redis, kafka, temporal, minio | `data/*` |
| `platform` | keycloak, grafana, prometheus, envoy-gateway | `platform/*` |
| `foundation` | cert-manager, external-secrets, vault | `foundation/*` |

**Implementation approach:**

**Option A**: Add to globals.yaml and reference in charts

```yaml
# sources/values/_shared/apps/apps-graph.yaml
tier: backend
domain: apps

# In Helm template
labels:
  ameide.io/tier: {{ .Values.tier | default "backend" }}
  ameide.io/domain: {{ .Values.domain | default "apps" }}
```

**Option B**: Derive from component.yaml `domain` field

The `component.yaml` already has `domain: apps|data|platform|foundation`. Map to tier:

| Domain | Tier |
|--------|------|
| `apps` (web/*) | frontend |
| `apps` (core/*, runtime/*) | backend |
| `data` | data |
| `platform` | platform |
| `foundation` | foundation |

**File**: `sources/charts/_helpers/common-labels.tpl`

```yaml
{{- define "common.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
ameide.io/tier: {{ .Values.tier | default "backend" }}
ameide.io/environment: {{ .Values.environment }}
{{- end -}}
```

---

### 3. Node Affinity for Workloads

**Priority**: MEDIUM (After node pools exist)

Add nodeSelector and tolerations to production workloads:

**File**: `sources/values/env/production/globals.yaml`

```yaml
# Production workloads must run on prod pool
nodeSelector:
  ameide.io/pool: prod

tolerations:
  - key: "ameide.io/environment"
    value: "production"
    effect: "NoSchedule"
```

**File**: `sources/values/env/dev/globals.yaml`

```yaml
# Dev workloads run on dev pool
nodeSelector:
  ameide.io/pool: dev
```

**File**: `sources/values/env/staging/globals.yaml`

```yaml
# Staging workloads run on staging pool
nodeSelector:
  ameide.io/pool: staging
```

---

### 4. Per-Environment Shutdown Strategy

**Priority**: HIGH

Enable turning individual environments on/off for cost savings.

#### What "Environment OFF" Means

1. **Node pool scaled to 0** - No compute costs
2. **Workloads at 0 replicas** - No pending pods in scheduler
3. **PVs preserved** - Data persists, reattaches on scale-up

#### 4.1 GitOps State Flag

Add environment state to globals.yaml:

**File**: `sources/values/env/dev/globals.yaml`

```yaml
environment:
  name: dev
  state: "on"  # or "off"

namespace: ameide-dev
domain: dev.ameide.io
# ... rest of config
```

#### 4.2 Helm Helper for Replica Count

**File**: `sources/charts/_helpers/environment.tpl`

```yaml
{{/*
Returns replica count based on environment state.
If state is "off", returns 0 regardless of configured replicas.
*/}}
{{- define "ameide.replicaCount" -}}
{{- if eq ((.Values.environment).state | default "on") "off" -}}
0
{{- else -}}
{{- .Values.replicaCount | default 1 -}}
{{- end -}}
{{- end -}}

{{/*
Returns true if environment is active (state != "off")
*/}}
{{- define "ameide.isActive" -}}
{{- ne ((.Values.environment).state | default "on") "off" -}}
{{- end -}}
```

#### 4.3 Apply to Deployments

**File**: `sources/charts/apps/*/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
spec:
  replicas: {{ include "ameide.replicaCount" . }}
  # ...
```

#### 4.4 Gate CronJobs on Environment State

**File**: `sources/charts/*/templates/cronjob.yaml`

```yaml
{{- if eq (include "ameide.isActive" .) "true" }}
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ include "app.fullname" . }}-cleanup
spec:
  schedule: "0 2 * * *"
  # ...
{{- end }}
```

#### 4.5 Shutdown Procedure

**To turn OFF an environment (e.g., dev):**

```bash
# Step 1: Update GitOps state
# Edit sources/values/env/dev/globals.yaml
environment:
  state: "off"

# Step 2: Commit and push
git add sources/values/env/dev/globals.yaml
git commit -m "chore: turn off dev environment"
git push

# Step 3: Wait for ArgoCD sync (workloads → 0 replicas)
argocd app sync ameide-dev --prune

# Step 4: Scale node pool to 0
az aks nodepool scale \
  --resource-group Ameide \
  --cluster-name ameide \
  --name dev \
  --node-count 0
```

**To turn ON an environment:**

```bash
# Step 1: Scale node pool up
az aks nodepool scale \
  --resource-group Ameide \
  --cluster-name ameide \
  --name dev \
  --node-count 1

# Step 2: Update GitOps state
# Edit sources/values/env/dev/globals.yaml
environment:
  state: "on"

# Step 3: Commit and push
git add sources/values/env/dev/globals.yaml
git commit -m "chore: turn on dev environment"
git push

# Step 4: ArgoCD syncs, pods schedule on new nodes
```

#### 4.6 Automation (Optional)

Use Azure Automation or GitHub Actions for scheduled shutdown:

**GitHub Actions example** (`.github/workflows/env-scheduler.yml`):

```yaml
name: Environment Scheduler

on:
  schedule:
    # Turn off dev+staging at 8 PM UTC weekdays
    - cron: '0 20 * * 1-5'
    # Turn on dev+staging at 6 AM UTC weekdays
    - cron: '0 6 * * 1-5'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment (dev/staging)'
        required: true
      action:
        description: 'Action (on/off)'
        required: true

jobs:
  toggle-environment:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Determine action from schedule
        id: schedule
        run: |
          HOUR=$(date -u +%H)
          if [ "$HOUR" -eq "20" ]; then
            echo "action=off" >> $GITHUB_OUTPUT
          else
            echo "action=on" >> $GITHUB_OUTPUT
          fi

      - name: Update globals.yaml
        run: |
          ACTION=${{ github.event.inputs.action || steps.schedule.outputs.action }}
          for ENV in dev staging; do
            yq -i '.environment.state = "'$ACTION'"' sources/values/$ENV/globals.yaml
          done

      - name: Commit changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add sources/values/*/globals.yaml
          git commit -m "chore: scheduled environment $ACTION" || true
          git push

      - name: Scale node pools
        env:
          AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
        run: |
          az login --service-principal ...
          ACTION=${{ github.event.inputs.action || steps.schedule.outputs.action }}
          COUNT=$([[ "$ACTION" == "on" ]] && echo 1 || echo 0)
          for POOL in dev staging; do
            az aks nodepool scale \
              --resource-group Ameide \
              --cluster-name ameide \
              --name $POOL \
              --node-count $COUNT
          done
```

#### 4.7 Data Persistence During Shutdown

> **Important**: PVs (Azure Disks) are NOT deleted when node pool scales to 0.

| Resource | Behavior During Shutdown |
|----------|-------------------------|
| PVCs (Azure Disk) | ✅ Preserved, reattaches on scale-up |
| ConfigMaps/Secrets | ✅ Preserved in etcd |
| CronJob history | ✅ Preserved |
| Logs (Loki) | ✅ Preserved in MinIO/Azure Blob |
| Metrics (Prometheus) | ⚠️ Gap during downtime |

#### 4.8 Tenants and environmentState

> **How environmentState applies to tenants:**
>
> - For **Shared SaaS** and **namespace-per-tenant** on the shared cluster, `environmentState`
>   controls the internal environments (`dev`, `staging`, `ameide-prod`) and is **not**
>   used to selectively shut down a single tenant namespace.
> - For **Private cloud**, `environmentState` is applied per tenant cluster
>   (e.g., only their `ameide-dev` namespace is scaled to zero).
>
> Per-tenant shutdown (namespace-per-tenant) is **future work** and would be modelled
> as a per-tenant flag in `sources/values/tenants/<tenant-id>/globals.yaml`:
>
> ```yaml
> # sources/values/tenants/acme/globals.yaml
> tenant:
>   id: acme
>   state: "off"  # Future: scale tenant workloads to 0
> ```

---

### 5. Resource Tier Definitions

**Priority**: MEDIUM

Define clear resource tiers per environment:

**File**: `sources/values/env/dev/globals.yaml`

```yaml
namespace: ameide-dev
environment: dev
domain: dev.ameide.io

# Minimal resources for development
resources:
  defaults:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

# Single replica, no autoscaling
replicas:
  default: 1

autoscaling:
  enabled: false

logLevel: debug
imagePullPolicy: Always
```

Note: `imagePullPolicy` is a Kubernetes runtime behavior knob; the GitOps rollout/determinism policy (pin by digest/SHA, avoid floating tags) is tracked separately under `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.

**File**: `sources/values/env/staging/globals.yaml`

```yaml
namespace: ameide-staging
environment: staging
domain: staging.ameide.io

# Production-like resources, lower scale
resources:
  defaults:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi

# Match production replica count
replicas:
  default: 2

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5

logLevel: info
imagePullPolicy: IfNotPresent
```

**File**: `sources/values/env/production/globals.yaml`

```yaml
namespace: ameide-prod
environment: production
domain: ameide.io

# Full production resources
resources:
  defaults:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 2Gi

# High availability
replicas:
  default: 3

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPU: 70
  targetMemory: 80

logLevel: info
imagePullPolicy: IfNotPresent
```

### 2. Enable HPA for Production Services

**Priority**: HIGH

Update app values to use HPA in production:

**File**: `sources/values/_shared/apps/platform.yaml`

```yaml
# HPA configuration (enabled per-environment)
autoscaling:
  enabled: false  # Overridden in production
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

**File**: `sources/values/env/production/apps/platform/platform.yaml`

```yaml
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
```

### 3. Add PodDisruptionBudgets

**Priority**: MEDIUM

Add PDB template to application charts:

**File**: `sources/charts/apps/platform/templates/pdb.yaml`

```yaml
{{- if .Values.podDisruptionBudget.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "platform.fullname" . }}
  labels:
    {{- include "platform.labels" . | nindent 4 }}
spec:
  {{- if .Values.podDisruptionBudget.minAvailable }}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- else }}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable | default 1 }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "platform.selectorLabels" . | nindent 6 }}
{{- end }}
```

**Values**:

```yaml
# sources/values/_shared/apps/platform.yaml
podDisruptionBudget:
  enabled: false

# sources/values/env/production/apps/platform/platform.yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

### 4. Namespace Network Policies

**Priority**: MEDIUM

Create per-namespace isolation:

**File**: `sources/charts/foundation/common/raw-manifests/templates/networkpolicy-namespace.yaml`

```yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: {{ .Values.namespace }}
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow from same namespace
    - from:
        - podSelector: {}
    # Allow from argocd for sync
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: argocd
    # Allow from observability
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ .Values.namespace }}
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # Allow same namespace
    - to:
        - podSelector: {}
    # Allow external (internet)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
{{- end }}
```

### 5. Environment-Specific Vault Paths

**Priority**: LOW

Organize secrets by environment:

**Current**: All environments share `/ameide/*` paths

**Proposed**: Environment-prefixed paths

```yaml
# Vault secret structure
ameide/
├── dev/
│   ├── postgres-ameide-password
│   ├── keycloak-admin-password
│   └── ...
├── staging/
│   ├── postgres-ameide-password
│   └── ...
└── production/
    ├── postgres-ameide-password
    └── ...
```

**Update ExternalSecret paths**:

```yaml
# sources/values/_shared/foundation/foundation-vault-secrets-platform.yaml
vaultSecretPath: "ameide/{{ .Values.environment }}"
```

---

## Observability Stack Scaling

### Current State

| Component | Replicas | Mode |
|-----------|----------|------|
| Prometheus | 1 | Single |
| Loki | 1 | Single Binary |
| Tempo | 1 | Single Binary |
| OTEL Collector | 1 (prod: 2) | Deployment |
| Grafana | 1 | Single |

### Production Recommendations

| Component | Replicas | Mode | Rationale |
|-----------|----------|------|-----------|
| Prometheus | 2 | HA Pair | Redundancy |
| Loki | 3+3+1 | SimpleScalable | Read/Write/Backend |
| Tempo | 3 | Distributed | Ingesters |
| OTEL Collector | 3 | Deployment | Load distribution |
| Grafana | 2 | HA | Redundancy |

---

## Implementation Steps

### Phase 0: Bicep Consolidation
1. [x] Decide canonical Bicep source (`azure/` vs `bicep/`) → **`bicep/` is canonical**
2. [x] Update VM SKU to D4as/D2as per 240 recommendations → **Done in aks.bicep**
3. [ ] Test Bicep deployment in dev subscription
4. [ ] Remove or deprecate duplicate Bicep module (`azure/` directory)

### Phase 1: Service Tier Labels
5. [x] Add `tier` field to all values files (`_shared/apps/*.yaml`, etc.) → **2025-12-04**
6. [x] Create common-labels helper template → **per-chart `<name>.tierLabels` helpers**
7. [x] Update Helm charts to include `ameide.io/tier` label → **14 app charts updated 2025-12-04**
8. [ ] Deploy and verify labels with `kubectl get pods --show-labels`

### Phase 2: Node Pool Strategy (Option B - Per-Environment)
9. [x] Add `system`, `dev`, `staging`, `prod` node pool definitions to Bicep → **2025-12-04**
10. [x] Configure dev/staging pools with `minCount: 0` for shutdown capability → **Done**
11. [x] Deploy node pools to AKS cluster → **Deployed via Terraform 2025-12-05**
12. [x] Add nodeSelector to globals.yaml per environment → **2025-12-04**
13. [x] Add prod tolerations for production taint → **2025-12-04**
14. [x] Verify workloads schedule on correct pools → **Verified 2025-12-05** (dev→aks-dev-*, staging→aks-staging-*, prod→aks-prod-*)

### Phase 3: Environment Shutdown Capability
15. [x] Add `environmentState` field to all globals.yaml files → **2025-12-04**
16. [x] Create `ameide.replicaCount` Helm helper template → **ameide-library chart**
17. [x] Create `ameide.isActive` Helm helper template → **ameide-library chart**
18. [ ] Update Deployment templates to use `ameide.replicaCount`
19. [ ] Update CronJob templates to gate on `ameide.isActive`
20. [ ] Test shutdown procedure manually (dev first)
21. [ ] Document shutdown/startup runbook

### Phase 4: Shutdown Automation (Optional)
22. [ ] Create GitHub Actions workflow for scheduled shutdown
23. [ ] Configure Azure service principal for nodepool scaling
24. [ ] Test scheduled shutdown (nights/weekends)
25. [ ] Monitor cost savings

### Phase 5: Resource Tiers
26. [x] Update dev globals.yaml with minimal resources → **Done (150m/256Mi)**
27. [x] Update staging globals.yaml with medium resources → **Done (200m/256Mi)**
28. [x] Update production globals.yaml with full resources → **Done (250m/512Mi)**
29. [ ] Validate apps inherit global defaults

### Phase 6: High Availability
30. [ ] Enable HPA for production critical services
31. [ ] Add PDB templates to app charts
32. [ ] Configure PDB values for production
33. [ ] Test node drain behavior

### Phase 7: Secret Management
34. [ ] Plan Vault path migration (optional)
35. [ ] Create environment-prefixed secrets
36. [ ] Update ExternalSecret references
37. [ ] Rotate credentials

---

## Validation Checklist

### Bicep & Node Pools
- [x] Single canonical Bicep source exists → `infra/bicep/managed-application/modules/aks.bicep`
- [x] Node pools created: system, dev, staging, prod → **Deployed 2025-12-05** (system:2, dev:3, staging:2, prod:3)
- [x] Dev/staging pools have `minCount: 0` configured in Bicep
- [x] Prod pool has autoscaling enabled (3-8 nodes) configured in Bicep
- [x] All pools have correct `ameide.io/pool` labels configured in Bicep

### Service Tier Labels
- [x] `tier` field added to all app values files (20+ files)
- [x] `<chartname>.tierLabels` helper created in each app chart `_helpers.tpl`
- [x] Helm charts updated to include tierLabels in deployment templates (14 charts)
- [ ] All pods have `ameide.io/tier` label (needs ArgoCD sync after node pool deploy)
- [ ] All pods have `ameide.io/environment` label (needs ArgoCD sync after node pool deploy)

### Node Affinity
- [x] nodeSelector configured in globals.yaml per environment
- [x] Tolerations configured in globals.yaml per environment
- [x] Third-party chart subcomponents have tolerations (console, provisioning jobs) → Fixed `9e54a78`
- [x] Production workloads run on prod pool only → **Verified 2025-12-05** (pods→aks-prod-*)
- [x] Dev workloads run on dev pool only → **Verified 2025-12-05** (pods→aks-dev-*)
- [x] Staging workloads run on staging pool only → **Verified 2025-12-05** (pods→aks-staging-*)
- [x] System workloads run on system pool → **Verified 2025-12-05** (argocd, cert-manager on system nodes)

### Environment Shutdown
- [x] `environmentState` field exists in all globals.yaml
- [x] `ameide.replicaCount` helper created in ameide-library chart
- [x] `ameide.isActive` helper created in ameide-library chart
- [ ] Deployment templates updated to use `ameide.replicaCount`
- [ ] CronJob templates gate on `ameide.isActive`
- [ ] Setting `environmentState: "off"` scales workloads to 0 (needs testing)
- [ ] Dev pool can scale to 0 nodes (needs testing)
- [ ] PVs persist when pool is at 0 nodes (needs testing)
- [ ] Shutdown/startup runbook documented

### Shutdown Automation (if implemented)
- [ ] GitHub Actions workflow triggers on schedule
- [ ] Azure credentials configured securely
- [ ] Scheduled shutdown works (nights/weekends)
- [ ] Cost savings visible in Azure Cost Management

### Resource Tiers
- [x] Dev apps use minimal resources → `defaultResources` in `globals.yaml` (150m/256Mi)
- [x] Staging apps use medium resources → `defaultResources` in `globals.yaml` (200m/256Mi)
- [x] Production apps use full resources → `defaultResources` in `globals.yaml` (250m/512Mi)
- [ ] Resource limits enforced (apps must reference global defaults)

### High Availability
- [ ] Production services have min 2 replicas
- [ ] HPA scales under load
- [ ] PDB prevents all pods going down
- [ ] Node drain respects PDB

### Secret Management
- [x] Each environment has isolated secrets → **Helm lookup+randAlphaNum pattern generates unique passwords per namespace**
- [x] No cross-environment secret access → **Secrets generated in-namespace, not shared**
- [ ] Secret rotation works per-environment → **Planned: Delete secret + ArgoCD sync regenerates**
