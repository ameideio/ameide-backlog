# 446: Namespace Isolation Architecture

## Status: Implemented (Revised)

## Summary

This document describes the namespace isolation architecture for the Ameide platform. The architecture separates **cluster-scoped operators** (deployed once) from **environment-scoped workloads** (deployed per environment).

## Related Documents

- [442-environment-isolation.md](442-environment-isolation.md) - Environment dev/staging/prod isolation (node pools, shutdown)
- [445-argocd-namespace-isolation.md](445-argocd-namespace-isolation.md) - ArgoCD namespace isolation details
- [443-tenancy-models.md](443-tenancy-models.md) - Multi-tenant architecture
- [441-networking.md](441-networking.md) - Network policies

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CLUSTER-SCOPED (deployed once via cluster ApplicationSet)                  │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                         │ │
│  │  CRDs (cluster-scoped)           Operators (dedicated namespaces)       │ │
│  │  ├── cert-manager CRDs           ├── cert-manager    (cert-manager ns)  │ │
│  │  ├── external-secrets CRDs       ├── external-secrets (external-secrets)│ │
│  │  ├── cnpg CRDs                   ├── cloudnative-pg  (cnpg-system)      │ │
│  │  ├── strimzi CRDs                ├── strimzi-operator (strimzi-system)  │ │
│  │  ├── redis CRDs                  ├── redis-operator  (redis-system)     │ │
│  │  ├── clickhouse CRDs             ├── clickhouse-operator (clickhouse-system)│
│  │  ├── keycloak CRDs               └── keycloak-operator (keycloak-system)│ │
│  │  ├── gateway-api CRDs                                                   │ │
│  │  └── prometheus CRDs                                                    │ │
│  │                                                                         │ │
│  │  ArgoCD (argocd namespace - self-managed)                              │ │
│  │  ├── argocd-server, repo-server, controller                            │ │
│  │  ├── cert-manager (isolated instance for ArgoCD TLS)                   │ │
│  │  └── envoy-gateway (isolated controller for cluster gateway)           │ │
│  │                                                                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                              │                                               │
│                              │ Operators watch ALL namespaces                │
│                              ▼                                               │
│  ENVIRONMENT-SCOPED (deployed per environment via ameide ApplicationSet)    │
│  ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐   │
│  │     ameide-dev      │ │   ameide-staging    │ │    ameide-prod      │   │
│  │                     │ │                     │ │                     │   │
│  │  Workloads (CRs):   │ │  Workloads (CRs):   │ │  Workloads (CRs):   │   │
│  │  ├── Cluster (pg)   │ │  ├── Cluster (pg)   │ │  ├── Cluster (pg)   │   │
│  │  ├── Kafka          │ │  ├── Kafka          │ │  ├── Kafka          │   │
│  │  ├── RedisFailover  │ │  ├── RedisFailover  │ │  ├── RedisFailover  │   │
│  │  ├── ClickHouse     │ │  ├── ClickHouse     │ │  ├── ClickHouse     │   │
│  │  ├── Keycloak       │ │  ├── Keycloak       │ │  ├── Keycloak       │   │
│  │  ├── SecretStore    │ │  ├── SecretStore    │ │  ├── SecretStore    │   │
│  │  ├── Certificates   │ │  ├── Certificates   │ │  ├── Certificates   │   │
│  │  └── Applications   │ │  └── Applications   │ │  └── Applications   │   │
│  │                     │ │                     │ │                     │   │
│  │  Stateful services: │ │  Stateful services: │ │  Stateful services: │   │
│  │  ├── Vault          │ │  ├── Vault          │ │  ├── Vault          │   │
│  │  └── ExternalSecrets│ │  └── ExternalSecrets│ │  └── ExternalSecrets│   │
│  │      (ready check)  │ │      (ready check)  │ │      (ready check)  │   │
│  │                     │ │                     │ │                     │   │
│  └─────────────────────┘ └─────────────────────┘ └─────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Namespace Categories

### 1. Operator Namespaces (Cluster-Scoped, Always Running)

These namespaces contain operators that watch ALL namespaces:

| Namespace | Operator | Watches | Can Scale to Zero? |
|-----------|----------|---------|-------------------|
| `argocd` | ArgoCD + cert-manager + envoy-gateway | All namespaces | No |
| `cert-manager` | cert-manager | All namespaces | No |
| `external-secrets` | external-secrets | All namespaces | No |
| `cnpg-system` | cloudnative-pg | All namespaces | No |
| `strimzi-system` | strimzi-kafka | All namespaces | No |
| `redis-system` | redis-operator | All namespaces | No |
| `clickhouse-system` | clickhouse-operator | All namespaces | No |
| `keycloak-system` | keycloak-operator | All namespaces | No |
| `kube-system` | Kubernetes system | N/A | No |

### 2. Environment Namespaces (Workloads, Scalable)

These namespaces contain Custom Resources (workloads) managed by the operators:

| Namespace | Purpose | Dependencies | Can Scale to Zero? |
|-----------|---------|--------------|-------------------|
| `ameide-dev` | Development workloads | Operators | Yes |
| `ameide-staging` | Staging workloads | Operators | Yes |
| `ameide-prod` | Production workloads | Operators | Yes (not recommended) |

### 3. Tenant Namespaces (Future)

For namespace-per-tenant deployments:

| Namespace | Purpose | Dependencies | Can Scale to Zero? |
|-----------|---------|--------------|-------------------|
| `tenant-<id>-prod` | Tenant production | Operators | Per-tenant decision |

## Key Principles

### Principle 1: Operators are Cluster-Scoped

Operators deploy ONCE per cluster and watch ALL namespaces:

```
✅ CORRECT: One operator watches all environments

   cnpg-system namespace:
   └── cloudnative-pg operator (watches: *)
           │
           │ Manages Cluster CRs in:
           ├── ameide-dev
           ├── ameide-staging
           └── ameide-prod

❌ WRONG: Operator per environment (causes conflicts)

   ameide-dev:
   └── cloudnative-pg operator ──┐
                                 │ CONFLICT!
   ameide-staging:               │ Same webhooks,
   └── cloudnative-pg operator ──┘ ClusterRoles
```

### Principle 2: Workloads are Environment-Scoped

Custom Resources (the actual databases, caches, etc.) deploy per environment:

```yaml
# ameide-dev namespace
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-ameide
  namespace: ameide-dev  # ← Environment-scoped
spec:
  instances: 3
  storage:
    size: 10Gi
```

### Principle 3: Data Isolation via Namespaces

Each environment has completely isolated data:

| Resource | Dev | Staging | Prod |
|----------|-----|---------|------|
| PostgreSQL data | `ameide-dev/postgres-*` | `ameide-staging/postgres-*` | `ameide-prod/postgres-*` |
| Kafka topics | `ameide-dev/kafka-*` | `ameide-staging/kafka-*` | `ameide-prod/kafka-*` |
| Redis data | `ameide-dev/redis-*` | `ameide-staging/redis-*` | `ameide-prod/redis-*` |
| Vault secrets | `ameide-dev/vault` | `ameide-staging/vault` | `ameide-prod/vault` |

### Principle 4: ArgoCD Independence

ArgoCD has its own isolated cert-manager and envoy-gateway to avoid dependency on any environment:

```
argocd namespace:
├── argocd-server
├── argocd-repo-server
├── argocd-application-controller
├── argocd-cert-manager (isolated instance)
├── envoy-gateway (isolated controller for cluster gateway)
├── argocd-ameide-io-tls (certificate)
└── cluster-gateway (Gateway + HTTPRoute for argocd.ameide.io)
```

### Principle 5: CRDs are Cluster-Scoped

CRDs must be installed once per cluster, separate from operators:

```
✅ CORRECT: CRDs managed by cluster ApplicationSet

   cluster-crds-gateway (ArgoCD App)
   └── Gateway API CRDs (installed once)
           │
           │ Used by controllers in:
           ├── argocd (cluster-gateway)
           ├── ameide-dev (envoy-gateway)
           ├── ameide-staging (envoy-gateway)
           └── ameide-prod (envoy-gateway)

❌ WRONG: CRDs bundled in each Helm chart

   argocd/cluster-gateway (Helm chart with CRDs)
   ameide-dev/envoy-gateway (Helm chart with CRDs)
   ameide-staging/envoy-gateway (Helm chart with CRDs)
   └── CRD version conflicts and ownership issues
```

When including Helm charts that bundle CRDs (like Envoy Gateway), use `skipCrds: true`:

```yaml
# argocd/applications/cluster-gateway.yaml
helm:
  releaseName: cluster-gateway
  skipCrds: true  # CRDs managed by cluster ApplicationSet
```

## ApplicationSet Architecture

### Two ApplicationSets

```yaml
# 1. Cluster ApplicationSet (operators + CRDs)
argocd/applicationsets/cluster.yaml
  generators:
    - git:
        files:
          - path: environments/_shared/components/cluster/**/component.yaml
  # Creates: cluster-cloudnative-pg, cluster-crds-cnpg, etc.

# 2. Environment ApplicationSet (workloads)
argocd/applicationsets/ameide.yaml
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
                - path: environments/_shared/components/**/component.yaml
  # Creates: dev-data-postgres-cluster, staging-data-postgres-cluster, etc.
```

### Component Directory Structure

```
environments/_shared/components/
├── cluster/                    # Cluster-scoped (deployed once)
│   ├── crds/                   # CRDs for all operators
│   │   ├── cert-manager/
│   │   ├── cnpg/
│   │   ├── strimzi/
│   │   ├── redis/
│   │   ├── clickhouse/
│   │   ├── keycloak/
│   │   ├── external-secrets/
│   │   ├── gateway-api/
│   │   └── prometheus-operator/
│   └── operators/              # Operator deployments
│       ├── cert-manager/
│       ├── cloudnative-pg/
│       ├── strimzi/
│       ├── redis/
│       ├── clickhouse/
│       ├── keycloak/
│       └── external-secrets/
│
├── foundation/                 # Environment-scoped foundation
│   ├── operators/              # Environment-specific stateful services
│   │   ├── vault/              # Vault (stateful, per-environment)
│   │   ├── vault-bootstrap/
│   │   ├── vault-webhook-certs/
│   │   └── external-secrets-ready/
│   └── ...
│
├── data/                       # Environment-scoped data workloads
│   └── core/
│       ├── postgres-cluster/   # Cluster CR (managed by CNPG operator)
│       ├── kafka-cluster/      # Kafka CR (managed by Strimzi)
│       └── redis-failover/     # RedisFailover CR (managed by Redis operator)
│
├── platform/                   # Environment-scoped platform
│   └── ...
│
└── apps/                       # Environment-scoped applications
    └── ...
```

## Rollout Order

### Cluster ApplicationSet (rollout-phase)

```
010: CRDs (all CRDs deployed first)
020: Operators (all operators deployed after CRDs)
030: Post-operator config (if any)
```

### Environment ApplicationSet (rollout-phase)

```
010: Namespace creation (foundation-namespaces)
130: Secret infrastructure (vault-webhook-certs)
140: Configs (coredns-config, managed-storage)
150: State stores (vault-core)
199: Foundation smoke tests
250: Data workloads (postgres-cluster, kafka-cluster, redis-failover)
350: Platform workloads (envoy-gateway, keycloak, prometheus)
650: Application workloads (www-ameide, inference, agents)
```

## Why This Architecture?

### Problem with Per-Environment Operators

Kubernetes operators create **cluster-scoped resources** with hardcoded names:

| Resource Type | Example Name | Conflict? |
|--------------|--------------|-----------|
| MutatingWebhookConfiguration | `cnpg-mutating-webhook-configuration` | Yes - only one can exist |
| ValidatingWebhookConfiguration | `cnpg-validating-webhook-configuration` | Yes - only one can exist |
| ClusterRole | `cloudnative-pg` | Yes - ArgoCD ownership conflict |
| ClusterRoleBinding | `cloudnative-pg` | Yes - ArgoCD ownership conflict |

If deployed per environment, these resources conflict with each other.

### Benefits of Cluster-Scoped Operators

1. **No resource conflicts** - Webhooks and ClusterRoles exist once
2. **Resource efficiency** - One operator pod instead of three
3. **Simpler RBAC** - One ClusterRole per operator
4. **Standard pattern** - How Kubernetes operators are designed to work
5. **Independent scaling** - Environment namespaces can scale to zero without affecting operators

## Network Isolation

### NetworkPolicy for Environment Namespaces

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-environment
  namespace: ameide-dev
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Allow from same namespace
    - from:
        - namespaceSelector:
            matchLabels:
              ameide.io/environment: dev
    # Allow from argocd (GitOps)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: argocd
    # Allow from operator namespaces
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: cnpg-system
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: strimzi-system
    # ... other operators
```

## Failure Scenarios

### Scenario 1: Operator Namespace Down

**Impact:**
- Operator cannot reconcile CRs in any environment
- New CRs won't be processed
- Existing workloads continue running (pods persist)

**Not Impacted:**
- Running databases, caches, etc. (stateful workloads)
- Application traffic

**Recovery:**
- ArgoCD auto-heals operator deployment

### Scenario 2: Environment Namespace Down

**Impact:**
- That environment's workloads unavailable
- Applications in that environment down

**Not Impacted:**
- Other environments (completely isolated)
- Operators (in separate namespaces)
- ArgoCD (independent)

### Scenario 3: CRDs Deleted

**Impact:**
- All operators fail to reconcile
- CR resources become orphaned

**Recovery:**
1. ArgoCD syncs `cluster-crds-*` applications
2. CRDs restored
3. Operators resume reconciliation

## Implementation Checklist

- [x] ArgoCD has isolated cert-manager instance
- [x] ArgoCD has isolated envoy-gateway controller for cluster gateway
- [x] ArgoCD TLS independent of environment namespaces
- [x] Cluster ApplicationSet for operators and CRDs
- [x] Environment ApplicationSet for workloads only
- [x] Operators deployed to dedicated `-system` namespaces
- [x] CRDs deployed once (cluster-scoped)
- [x] Helm charts with bundled CRDs use `skipCrds: true`
- [x] Component.yaml files specify correct namespace
- [x] foundation-namespaces only creates environment namespace
- [x] NetworkPolicies for cross-namespace isolation
- [x] Documentation for isolation architecture (this doc)
- [ ] Monitoring dashboards per namespace
- [ ] Runbook for namespace failure recovery

## Implementation Commits

- `refactor: move operators to cluster-scoped namespaces`
- `refactor: move CRDs to cluster ApplicationSet`
- `feat: add cluster ApplicationSet for operators and CRDs`
