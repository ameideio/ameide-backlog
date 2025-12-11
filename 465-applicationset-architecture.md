# 465 – ApplicationSet Architecture

**Created**: 2025-12-07
**Updated**: 2025-12-07

> **Cross-References (Deployment Architecture Suite)**:
> This document is the **primary reference** for understanding how Ameide deploys components.
>
> | Document | Purpose |
> |----------|---------|
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Rollout phases & sub-phase patterns (`*10`, `*20`, `*50`, etc.) |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Chart folder structure & domain alignment |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | Secrets handling, OIDC client extraction patterns |
> | [473-ameide-technology.md](473-ameide-technology.md) | Technology architecture (how Backstage, Temporal fit in) |
>
> **Related (Implementation Details)**:
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Namespace isolation patterns
> - [466-postmortem-cluster-gateway-outage.md](466-postmortem-cluster-gateway-outage.md) – Incident from incorrect understanding

## Overview

Ameide uses **two ApplicationSets** to manage all deployments. Understanding the distinction is critical—confusing them has caused production incidents. ApplicationSets never apply raw Deployments for domain/process/agent services; instead they apply the declarative controller manifests (`IntelligentDomainController`, `IntelligentProcessController`, `IntelligentAgentController`) defined in [461-ipc-idc-iac.md](461-ipc-idc-iac.md), plus the operators/infra that reconcile them.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ArgoCD ApplicationSets                          │
├─────────────────────────────────┬───────────────────────────────────────┤
│         ameide.yaml             │           cluster.yaml                │
│    (Environment-Scoped)         │       (Cluster-Scoped)                │
├─────────────────────────────────┼───────────────────────────────────────┤
│  3 apps per component           │  1 app per component                  │
│  dev-*, staging-*, production-* │  cluster-*                            │
│  Namespace: from env list       │  Namespace: from component.yaml       │
│  Phases: 100-699                │  Phases: 010-030                      │
└─────────────────────────────────┴───────────────────────────────────────┘
```

## Quick Reference

### Where Do Files Go?

| What | Location | Example |
|------|----------|---------|
| Component definition | `environments/_shared/components/{domain}/{name}/component.yaml` | `observability/plausible/component.yaml` |
| Helm chart | `sources/charts/{domain}/{name}/` | `sources/charts/observability/plausible/` |
| Shared values (all envs) | `sources/values/_shared/{domain}/{name}.yaml` | `sources/values/_shared/observability/plausible.yaml` |
| Environment overrides | `sources/values/{env}/{domain}/{name}.yaml` | `sources/values/dev/observability/plausible.yaml` |
| Environment globals | `sources/values/{env}/globals.yaml` | `sources/values/dev/globals.yaml` |

### Cluster-Scoped Exception

| What | Location | Example |
|------|----------|---------|
| Component definition | `environments/_shared/components/cluster/{type}/{name}/component.yaml` | `cluster/configs/gateway/component.yaml` |
| Helm chart | `sources/charts/cluster/{name}/` | `sources/charts/cluster/gateway/` |
| Shared values | `sources/values/_shared/cluster/{name}.yaml` | `sources/values/_shared/cluster/gateway.yaml` |
| Cluster globals | `sources/values/cluster/globals.yaml` | (only one file) |

---

## The Two ApplicationSets

### 1. `ameide` ApplicationSet (Environment-Scoped)

**File**: `argocd/applicationsets/ameide.yaml`

#### Generator: Matrix (Environments × Components)

> **Updated 2025-12-11**: Now uses Git file generator for environments instead of hardcoded list.
> Cluster configs live in `config/clusters/*.yaml`, selected by Kustomize overlay.

```yaml
generators:
  - matrix:
      generators:
        # Git generator: reads cluster config (environments are defined in YAML)
        - git:
            files:
              - path: config/clusters/azure.yaml  # Patched to local.yaml by overlay
        # Git generator: scans for component.yaml files
        - git:
            files:
              - path: environments/_shared/components/apps/**/component.yaml
              - path: environments/_shared/components/data/**/component.yaml
              - path: environments/_shared/components/foundation/**/component.yaml
              - path: environments/_shared/components/observability/**/component.yaml
              - path: environments/_shared/components/platform/**/component.yaml
```

**Cluster config example** (`config/clusters/azure.yaml`):
```yaml
- cluster: aks-main
  clusterType: azure
  env: dev
  namespace: ameide-dev
  nodeProfile: dev-pool
  domain: dev.ameide.io
# ... staging, production entries
```

#### How Matrix Multiplication Works

```
List elements × Git files = Applications

dev         × plausible = dev-plausible
staging     × plausible = staging-plausible
production  × plausible = production-plausible

dev         × grafana   = dev-grafana
staging     × grafana   = staging-grafana
production  × grafana   = production-grafana
```

**Result**: Each component in `apps/`, `data/`, `foundation/`, `observability/`, or `platform/` generates **3 Applications** (one per environment).

#### Values Resolution Order (6 Layers)

> **Updated 2025-12-11**: Now uses 6-layer values with separate cluster and node-profile files.

For `dev-plausible` on Azure:

```yaml
valueFiles:
  - $values/sources/values/base/globals.yaml                     # 1. Cluster-agnostic defaults
  - $values/sources/values/cluster/azure/globals.yaml            # 2. Cluster-specific
  - $values/sources/values/env/dev/globals.yaml                  # 3. Environment-specific
  - $values/config/node-profiles/dev-pool.yaml                   # 4. Scheduling constraints
  - $values/sources/values/_shared/observability/plausible.yaml  # 5. Shared values
  - $values/sources/values/env/dev/observability/plausible.yaml  # 6. Environment override
```

All files use `ignoreMissingValueFiles: true`, so optional overrides don't cause errors.

#### Namespace Assignment

The namespace comes from the **cluster config file**, NOT the component:

```yaml
# config/clusters/azure.yaml
- env: dev
  namespace: ameide-dev  # ← All dev apps deploy here
  nodeProfile: dev-pool  # ← Used for layer 4
```

This means ALL environment-scoped components in an environment share the same namespace.

---

### 2. `cluster` ApplicationSet (Cluster-Scoped)

**File**: `argocd/applicationsets/cluster.yaml`

#### Generator: Single Git (No Matrix)

```yaml
generators:
  - git:
      files:
        - path: environments/_shared/components/cluster/**/component.yaml
```

**No list generator** = No multiplication = **1 Application per component**.

#### How It Works

```
Git files = Applications (1:1)

gateway        → cluster-gateway
coredns-config → cluster-coredns-config
```

#### Values Resolution Order

For `cluster-gateway`:

```yaml
valueFiles:
  - $values/sources/values/cluster/globals.yaml         # 1. Cluster globals
  - $values/sources/values/_shared/cluster/gateway.yaml # 2. Component values
```

**Note**: No environment-specific overrides because cluster components are environment-agnostic.

#### Namespace Assignment

The namespace comes from the **component.yaml itself**:

```yaml
# environments/_shared/components/cluster/configs/gateway/component.yaml
name: gateway
namespace: argocd        # ← Deploys to argocd namespace

# environments/_shared/components/cluster/configs/coredns-config/component.yaml
name: coredns-config
namespace: kube-system   # ← Deploys to kube-system
```

**There is no "cluster namespace"**—each component declares its target.

---

## Directory Structure

### Complete Tree

```
ameide-gitops/
│
├── argocd/applicationsets/
│   ├── ameide.yaml              # Environment-scoped ApplicationSet
│   └── cluster.yaml             # Cluster-scoped ApplicationSet
│
├── environments/_shared/components/
│   │
│   │   # ─── Environment-Scoped (→ ameide ApplicationSet) ───
│   ├── apps/                    # rollout-phase: 600-699
│   │   ├── gateway/
│   │   ├── keycloak/
│   │   └── .../component.yaml
│   │
│   ├── data/                    # rollout-phase: 200-299, 400-499
│   │   ├── clickhouse/
│   │   ├── postgres-clusters/
│   │   └── .../component.yaml
│   │
│   ├── foundation/              # rollout-phase: 100-199
│   │   ├── namespaces/
│   │   ├── secrets/
│   │   └── .../component.yaml
│   │
│   ├── observability/           # rollout-phase: 500-599
│   │   ├── grafana/
│   │   ├── plausible/
│   │   ├── prometheus/
│   │   └── .../component.yaml
│   │
│   ├── platform/                # rollout-phase: 300-399
│   │   ├── temporal/
│   │   └── .../component.yaml
│   │
│   │   # ─── Cluster-Scoped (→ cluster ApplicationSet) ───
│   └── cluster/
│       ├── crds/                # rollout-phase: 010
│       │   ├── cert-manager/
│       │   └── .../component.yaml
│       │
│       ├── operators/           # rollout-phase: 020
│       │   ├── external-secrets/
│       │   ├── cloudnative-pg/
│       │   └── .../component.yaml
│       │
│       └── configs/             # rollout-phase: 030
│           ├── gateway/component.yaml      # namespace: argocd
│           └── coredns-config/component.yaml  # namespace: kube-system
│
├── sources/charts/
│   ├── apps/
│   ├── data/
│   ├── foundation/
│   ├── observability/           # Created for plausible
│   │   └── plausible/
│   ├── platform/
│   ├── cluster/                 # Cluster-specific charts
│   │   └── gateway/
│   └── third_party/             # Vendored upstream charts
│
└── sources/values/
    ├── _shared/                 # Shared across ALL environments
    │   ├── apps/{name}.yaml
    │   ├── data/{name}.yaml
    │   ├── foundation/{name}.yaml
    │   ├── observability/{name}.yaml
    │   ├── platform/{name}.yaml
    │   └── cluster/{name}.yaml  # Cluster component values
    │
    ├── cluster/
    │   └── globals.yaml         # Only globals for cluster scope
    │
    ├── dev/
    │   ├── globals.yaml         # Dev environment globals
    │   ├── apps/{name}.yaml     # Dev-specific overrides
    │   ├── data/{name}.yaml
    │   ├── observability/{name}.yaml
    │   └── ...
    │
    ├── staging/
    │   ├── globals.yaml
    │   └── ...
    │
    └── production/
        ├── globals.yaml
        └── ...
```

---

## Rollout Phases

### Cluster ApplicationSet (010-030)

Runs **before** environment apps. Critical infrastructure.

| Phase | Type | Examples |
|-------|------|----------|
| 010 | CRDs | cert-manager CRDs, gateway-api CRDs |
| 020 | Operators | external-secrets, cloudnative-pg, envoy-gateway |
| 030 | Configs | cluster gateway, coredns-config |

### Ameide ApplicationSet (100-699)

| Phase | Domain | Type | Examples |
|-------|--------|------|----------|
| 100 | foundation | namespaces | namespace creation |
| 110-120 | foundation | CRDs, operators | — |
| 130 | foundation | secrets | vault SecretStore, pull secrets |
| 140-150 | foundation | configs, runtimes | — |
| 199 | foundation | smokes | — |
| 210-260 | data | full stack | clickhouse, postgres, redis |
| 299 | data | smokes | — |
| 310-355 | platform | full stack | temporal, keycloak |
| 399 | platform | smokes | — |
| 410-455 | data-ext | extended data | additional data services |
| 499 | data-ext | smokes | — |
| 510-550 | observability | full stack | grafana, prometheus, loki |
| 599 | observability | smokes | — |
| 610-650 | apps | full stack | gateway, inference, workflows |
| 699 | apps | smokes | — |

---

## Key Differences Summary

| Aspect | ameide (env-scoped) | cluster (cluster-scoped) |
|--------|---------------------|--------------------------|
| **File** | `argocd/applicationsets/ameide.yaml` | `argocd/applicationsets/cluster.yaml` |
| **Generator** | Matrix: list × git | Single git |
| **Apps per component** | 3 (dev, staging, prod) | 1 |
| **App naming** | `{env}-{name}` | `cluster-{name}` |
| **Namespace source** | List generator | Component's `namespace:` field |
| **Globals path** | `{env}/globals.yaml` | `cluster/globals.yaml` |
| **Shared values** | `_shared/{domain}/{name}.yaml` | `_shared/cluster/{name}.yaml` |
| **Env overrides** | `{env}/{domain}/{name}.yaml` | N/A |
| **Rollout phases** | 100-699 | 010-030 |
| **Use case** | Workloads per environment | Operators, CRDs, cluster infra |

---

## When to Use Which

### Use `cluster/` for:

- **CRDs** – Applied once, affect entire cluster
- **Operators** – One controller manages all namespaces
- **Cluster-scoped resources** – GatewayClass, ClusterRole, ClusterIssuer
- **Shared infrastructure** – CoreDNS config, cluster gateway for ArgoCD
- **Anything that must NOT be duplicated per environment**

### Use environment domains (`apps/`, `data/`, etc.) for:

- **Application workloads** – Your actual services
- **Per-environment databases** – Each env gets its own postgres
- **Per-environment secrets** – Isolated credentials
- **Per-environment config** – Different settings for dev vs prod
- **Anything needing isolation between dev/staging/prod**

---

## Adding Components

### Adding an Environment-Scoped Component

1. **Create component definition**:
   ```
   environments/_shared/components/{domain}/{name}/component.yaml
   ```

2. **Create/place Helm chart**:
   ```
   sources/charts/{domain}/{name}/
   ```
   Or reference an existing chart in `third_party/`.

3. **Create shared values**:
   ```
   sources/values/_shared/{domain}/{name}.yaml
   ```

4. **(Optional) Create environment overrides**:
   ```
   sources/values/dev/{domain}/{name}.yaml
   sources/values/staging/{domain}/{name}.yaml
   sources/values/production/{domain}/{name}.yaml
   ```

**Example component.yaml**:
```yaml
name: plausible
project: ameide
domain: observability
dependencyPhase: "observability"
componentType: "analytics"
rolloutPhase: "550"
chart:
  repoURL: https://github.com/ameideio/ameide-gitops.git
  path: sources/charts/observability/plausible
  version: main
```

**Note**: No `namespace:` field—it comes from the environment list.

> **Controllers-as-CRs**: For Domain/Process/Agent controllers the referenced chart/folder often just contains the IDC/IPC/IAC manifests (and supporting RBAC) rather than a Deployment. Operators (not the chart) create the workloads after Argo applies the controller CRs.

### Adding a Cluster-Scoped Component

1. **Create component definition**:
   ```
   environments/_shared/components/cluster/{type}/{name}/component.yaml
   ```
   Where `{type}` is `crds/`, `operators/`, or `configs/`.

2. **Create/place Helm chart**:
   ```
   sources/charts/cluster/{name}/
   ```

3. **Create shared values**:
   ```
   sources/values/_shared/cluster/{name}.yaml
   ```

**Example component.yaml**:
```yaml
name: gateway
project: ameide
namespace: argocd              # ← REQUIRED: target namespace
domain: cluster
componentType: "gateway"
rolloutPhase: "030"
chart:
  repoURL: https://github.com/ameideio/ameide-gitops.git
  path: sources/charts/cluster/gateway
  version: main
```

**Critical**: The `namespace:` field is REQUIRED for cluster components.

---

## Common Mistakes

### 1. Forgetting namespace in cluster components

❌ **Wrong**:
```yaml
# environments/_shared/components/cluster/configs/foo/component.yaml
name: foo
domain: cluster
# Missing namespace!
```

✅ **Correct**:
```yaml
name: foo
namespace: target-namespace  # Required!
domain: cluster
```

### 2. Putting values in wrong location

❌ **Wrong**: `sources/values/cluster/gateway.yaml`
✅ **Correct**: `sources/values/_shared/cluster/gateway.yaml`

The `sources/values/cluster/` folder should ONLY contain `globals.yaml`.

### 3. Manual Applications outside GitOps

Creating Applications manually via kubectl/UI bypasses the ApplicationSet.
This led to the [466 incident](466-postmortem-cluster-gateway-outage.md) where:
- A chart appeared "orphaned" because no component.yaml existed
- The manual Application wasn't visible in git
- Deleting the "orphaned" chart broke production

**Always create component.yaml files** for ApplicationSet management.

### 4. Confusing domain with folder

The `domain:` field in component.yaml should match the folder path:

| Folder | domain: |
|--------|---------|
| `environments/_shared/components/apps/` | `apps` |
| `environments/_shared/components/observability/` | `observability` |
| `environments/_shared/components/cluster/` | `cluster` |

---

## Troubleshooting

### "Application not being created"

1. Check ApplicationSet generator paths match your component location
2. Verify component.yaml is valid YAML
3. Check ArgoCD ApplicationSet controller logs

### "Values not being applied"

1. Verify values file exists at expected path
2. Check for typos in domain/name
3. Remember: `ignoreMissingValueFiles: true` means missing files are silent

### "Cluster component deploying to wrong namespace"

1. Check the `namespace:` field in component.yaml
2. Cluster components MUST specify their namespace

---

## Deployment Architecture Document Suite

This document is the entry point for understanding Ameide's GitOps deployment model. The suite consists of:

| Document | Scope | When to Read |
|----------|-------|--------------|
| **465-applicationset-architecture** (this doc) | How ApplicationSets generate apps | First – understand the basics |
| [447-waves-v3-cluster-scoped-operators](447-waves-v3-cluster-scoped-operators.md) | Rollout phases & sub-phases | When choosing rolloutPhase for a component |
| [464-chart-folder-alignment](464-chart-folder-alignment.md) | Chart folder conventions | When creating/moving charts |
| [426-keycloak-config-map](426-keycloak-config-map.md) | Secrets & OIDC patterns | When adding auth or secrets to a service |
| [473-ameide-technology](473-ameide-technology.md) | Technology architecture | When understanding Backstage, Temporal, SDKs |

### Quick Decision Tree

```
Adding a new component?
├── Is it a CRD or Operator?
│   └── YES → cluster/ domain, phase 010-020
│       See: 447 §Cluster Phases
├── Is it per-environment workload?
│   └── YES → apps/data/platform/observability domain
│       See: 447 §Environment Phases for phase selection
│       See: 464 for folder structure
├── Does it need database?
│   └── YES → Add to postgres_clusters databases list
│       See: CNPG patterns in 426
├── Does it need OIDC auth?
│   └── YES → Add Keycloak client + client-patcher
│       See: 426 §3.2 Client Secret Extraction
└── Does it need external secrets?
    └── YES → Add ExternalSecret referencing ameide-vault
        See: 426 §4 Secrets & ownership model
```

---

## Related Operational Runbooks

When troubleshooting specific components, these runbooks provide detailed procedures:

| Topic | Documents | Use When |
|-------|-----------|----------|
| **Temporal** | [305-workflow.md](305-workflow.md), [420](420-temporal-cnpg-dev-registry-runbook.md), [423](423-temporal-argocd-recovery.md) | Temporal not starting, schema issues |
| **Database (CNPG)** | [412-cnpg-owned-postgres-greds.md](412-cnpg-owned-postgres-greds.md) | DB credentials issues, connection failures |
| **Secrets/Vault** | [413](413-vault-bootstrap-readiness.md), [451](451-secrets-management.md), [452](452-vault-rbac-isolation.md) | Secrets not syncing, Vault sealed |
| **Kafka (Strimzi)** | [421-argocd-strimzi-kafkanodepool-health.md](421-argocd-strimzi-kafkanodepool-health.md) | Kafka health stuck, operator issues |
| **ClickHouse** | [419-clickhouse-argocd-resiliency.md](419-clickhouse-argocd-resiliency.md) | CHI not reconciling, CRD issues |
| **Keycloak** | [426-keycloak-config-map.md](426-keycloak-config-map.md) §5 | Realm import, client-patcher failures |
| **Tolerations** | [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md) | Pods stuck Pending on tainted nodes |
| **Observability** | [452-observability-namespace-isolation.md](452-observability-namespace-isolation.md) | RBAC conflicts, Prometheus/Grafana |
| **Gateway** | [466-postmortem-cluster-gateway-outage.md](466-postmortem-cluster-gateway-outage.md) | ArgoCD ingress down, envoy issues |

---

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-12-07 | Created documentation | Clarify how cluster vs environment ApplicationSets work |
| 2025-12-07 | Expanded with complete file paths | Prevent confusion about where files should go |
| 2025-12-07 | Added common mistakes section | Document lessons from 466 incident |
| 2025-12-07 | Added cross-references to deployment suite | Improve discoverability of related docs |
| 2025-12-07 | Added operational runbooks section | Link to troubleshooting procedures for common issues |
