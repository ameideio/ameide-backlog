# 465 – ApplicationSet Architecture

**Created**: 2025-12-07

> **Related documents:**
> - [464-chart-folder-alignment.md](464-chart-folder-alignment.md) – Chart folder structure
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Namespace isolation patterns

## Overview

This document describes how ApplicationSets manage component deployment across environments and cluster-scoped resources.

## Two ApplicationSets

| ApplicationSet | File | Purpose | Naming Pattern |
|----------------|------|---------|----------------|
| `ameide` | `argocd/applicationsets/ameide.yaml` | Environment-scoped apps | `{env}-{name}` |
| `cluster` | `argocd/applicationsets/cluster.yaml` | Cluster-scoped resources | `cluster-{name}` |

## ameide ApplicationSet (Environment-Scoped)

### Generator: Matrix (Environments × Components)

```yaml
generators:
  - matrix:
      generators:
        # Environment list
        - list:
            elements:
              - env: dev
                namespace: ameide-dev
              - env: staging
                namespace: ameide-staging
              - env: production
                namespace: ameide-prod
        # Component definitions
        - git:
            files:
              - path: environments/_shared/components/apps/**/component.yaml
              - path: environments/_shared/components/data/**/component.yaml
              - path: environments/_shared/components/foundation/**/component.yaml
              - path: environments/_shared/components/observability/**/component.yaml
              - path: environments/_shared/components/platform/**/component.yaml
```

### How it works

1. **Matrix multiplication**: Each component × each environment = one Application
2. **Example**: `plausible` component generates:
   - `dev-plausible` → deploys to `ameide-dev`
   - `staging-plausible` → deploys to `ameide-staging`
   - `production-plausible` → deploys to `ameide-prod`

### Values resolution

```yaml
valueFiles:
  - $values/sources/values/{env}/globals.yaml           # Environment globals
  - $values/sources/values/_shared/{domain}/{name}.yaml # Shared component values
  - $values/sources/values/{env}/{domain}/{name}.yaml   # Environment overrides
```

### Namespace

The namespace comes from the **environment list**, not the component:
- All components in an environment share the same namespace (`ameide-dev`, etc.)

## cluster ApplicationSet (Cluster-Scoped)

### Generator: Single Git (No Environment Matrix)

```yaml
generators:
  - git:
      files:
        - path: environments/_shared/components/cluster/**/component.yaml
```

### How it works

1. **No matrix**: Each component generates exactly ONE Application
2. **Example**: `gateway` component generates only `cluster-gateway`

### Values resolution

```yaml
valueFiles:
  - $values/sources/values/cluster/globals.yaml         # Cluster globals
  - $values/sources/values/_shared/cluster/{name}.yaml  # Component values
```

### Namespace

The namespace comes from the **component.yaml itself**:

```yaml
# gateway/component.yaml
namespace: argocd      # ← Gateway deploys here

# coredns-config/component.yaml
namespace: kube-system # ← CoreDNS config deploys here
```

**There is no "cluster namespace"** – each component specifies its target.

## Component Folder Structure

```
environments/_shared/components/
├── apps/                    # → ameide ApplicationSet
├── data/                    # → ameide ApplicationSet
├── foundation/              # → ameide ApplicationSet
├── observability/           # → ameide ApplicationSet
├── platform/                # → ameide ApplicationSet
└── cluster/                 # → cluster ApplicationSet (separate!)
    ├── crds/                # rollout-phase: 010
    ├── operators/           # rollout-phase: 020
    └── configs/             # rollout-phase: 030
        ├── coredns-config/
        └── gateway/
```

## Values Folder Structure

```
sources/values/
├── _shared/
│   ├── apps/{name}.yaml
│   ├── data/{name}.yaml
│   ├── foundation/{name}.yaml
│   ├── observability/{name}.yaml
│   ├── platform/{name}.yaml
│   └── cluster/{name}.yaml      # ← Cluster component values
│
├── cluster/
│   └── globals.yaml             # ← Only globals here
│
├── dev/
│   ├── globals.yaml
│   ├── apps/{name}.yaml
│   └── ...
├── staging/
└── production/
```

## Rollout Phases

### cluster ApplicationSet (010-030)

| Phase | Type | Description |
|-------|------|-------------|
| 010 | CRDs | Custom Resource Definitions |
| 020 | Operators | Controllers and operators |
| 030 | Configs | Post-operator configurations |

### ameide ApplicationSet (100-699)

| Phase Range | Domain | Description |
|-------------|--------|-------------|
| 100-199 | foundation | Namespaces, secrets, base infra |
| 200-299 | data | Data layer (first pass) |
| 300-399 | platform | Platform services |
| 400-499 | data | Data layer (extended) |
| 500-599 | observability | Monitoring, logging, tracing |
| 600-699 | apps | Application workloads |

## Key Differences Summary

| Aspect | ameide (env-scoped) | cluster (cluster-scoped) |
|--------|---------------------|-------------------------|
| Generator | Matrix: envs × components | Single git generator |
| Apps per component | 3 (one per env) | 1 |
| App naming | `{env}-{name}` | `cluster-{name}` |
| Namespace source | Environment list | Component's `namespace:` field |
| Values globals | `{env}/globals.yaml` | `cluster/globals.yaml` |
| Values shared | `_shared/{domain}/` | `_shared/cluster/` |
| Use case | App workloads | Operators, CRDs, shared infra |

## When to Use Which

### Use `cluster/` for:
- CRDs (applied once cluster-wide)
- Operators (one instance manages all namespaces)
- Cluster-scoped resources (ClusterRole, GatewayClass)
- Shared infrastructure (CoreDNS config, cluster gateway)

### Use environment domains for:
- Application workloads
- Per-environment databases
- Per-environment secrets
- Anything that needs isolation between dev/staging/prod

## Adding a New Cluster Component

1. Create component at `environments/_shared/components/cluster/{type}/{name}/component.yaml`
2. Set `namespace:` to the target namespace
3. Create values at `sources/values/_shared/cluster/{name}.yaml`
4. The `cluster` ApplicationSet automatically discovers it

Example:
```yaml
# environments/_shared/components/cluster/configs/my-config/component.yaml
name: my-config
project: ameide
namespace: my-target-namespace  # ← Deploys here
domain: cluster
rolloutPhase: "030"
chart:
  repoURL: https://github.com/ameideio/ameide-gitops.git
  path: sources/charts/cluster/my-config
  version: main
```

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-12-07 | Created documentation | Clarify how cluster vs environment ApplicationSets work |
