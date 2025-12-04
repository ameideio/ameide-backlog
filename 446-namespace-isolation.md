# 446: Namespace Isolation Architecture

## Status: Implemented

## Summary

This document describes the complete namespace isolation architecture for the Ameide platform, covering cluster-scoped services, environment namespaces, and their independence from each other.

## Related Documents

- [442-environment-isolation.md](442-environment-isolation.md) - Environment dev/staging/prod isolation (node pools, shutdown)
- [445-argocd-namespace-isolation.md](445-argocd-namespace-isolation.md) - ArgoCD namespace isolation details
- [443-tenancy-models.md](443-tenancy-models.md) - Multi-tenant architecture
- [441-networking.md](441-networking.md) - Network policies

## Namespace Categories

The cluster has three categories of namespaces with different isolation requirements:

### 1. Cluster-Scoped Namespaces (Always Running)

These namespaces contain infrastructure that must be operational regardless of environment state:

| Namespace | Purpose | Dependencies | Can Scale to Zero? |
|-----------|---------|--------------|-------------------|
| `argocd` | GitOps controller, TLS, cert-manager | None (self-contained) | No |
| `kube-system` | Kubernetes system components | None | No |

### 2. Environment Namespaces (Scalable)

These namespaces contain ALL application workloads AND operators that can be independently scaled:

| Namespace | Purpose | Dependencies | Can Scale to Zero? |
|-----------|---------|--------------|-------------------|
| `ameide-dev` | Development environment (apps + operators) | ArgoCD | Yes |
| `ameide-staging` | Staging environment (apps + operators) | ArgoCD | Yes |
| `ameide-prod` | Production environment (apps + operators) | ArgoCD | Yes (not recommended) |

**Key Principle**: All operators (cert-manager, CNPG, Strimzi, Redis, etc.) deploy INTO the environment namespace, NOT into separate cluster-scoped namespaces. This ensures complete environment isolation.

### 3. Tenant Namespaces (Future)

For namespace-per-tenant deployments:

| Namespace | Purpose | Dependencies | Can Scale to Zero? |
|-----------|---------|--------------|-------------------|
| `tenant-<id>-prod` | Tenant production | ArgoCD, shared infra | Per-tenant decision |

## Isolation Principles

### Principle 1: No Cross-Namespace Dependencies for Core Functions

Each namespace must be self-sufficient for its core functions:

```
✅ CORRECT: ArgoCD has its own cert-manager
   argocd namespace:
   ├── argocd-server
   ├── cert-manager (isolated instance)
   └── ClusterIssuer + Certificate

❌ WRONG: ArgoCD depends on dev namespace cert-manager
   argocd namespace:
   └── argocd-server (needs TLS)
           ↓
   ameide-dev namespace:
   └── cert-manager (if dev is down, ArgoCD breaks)
```

### Principle 2: Operators Deploy INTO Environment Namespaces

**All operators deploy INTO the environment namespace**, not into separate cluster-scoped namespaces:

```
✅ CORRECT: Operators in environment namespace
   ameide-dev namespace:
   ├── foundation-cert-manager (operator)
   ├── foundation-cloudnative-pg (operator)
   ├── foundation-strimzi-operator (operator)
   ├── foundation-redis-operator (operator)
   ├── foundation-external-secrets (operator)
   ├── dev-data-postgres-cluster (workload)
   ├── dev-data-kafka-cluster (workload)
   └── dev-apps-* (applications)

❌ WRONG: Operators in separate namespaces
   cnpg-system namespace:     ← Shared, breaks isolation
   └── cloudnative-pg operator

   strimzi-system namespace:  ← Shared, breaks isolation
   └── strimzi operator

   ameide-dev namespace:
   └── applications only
```

### Principle 3: Only Truly Cluster-Scoped Resources Are Shared

| Resource Type | Correct Location | Wrong Location |
|---------------|------------------|----------------|
| CRDs | Cluster-scoped | Any namespace |
| ClusterRoles | Cluster-scoped | Any namespace |
| ClusterIssuers | Cluster-scoped (but managed by namespace cert-managers) | N/A |
| StorageClasses | Cluster-scoped | Any namespace |
| ArgoCD | `argocd` namespace | Environment namespace |
| All operators | Environment namespace (`ameide-*`) | Separate `-system` namespaces |

### Principle 4: Environment Independence

Each environment must operate completely independently:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Shared Cluster                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [argocd]          Independent of all environments                   │
│      │             Contains: ArgoCD + cert-manager                   │
│      │                                                               │
│      │ Manages (GitOps)                                             │
│      │                                                               │
│      ├──────────────────┬──────────────────┬──────────────────┐     │
│      ▼                  ▼                  ▼                  │     │
│  [ameide-dev]      [ameide-staging]   [ameide-prod]          │     │
│      │                  │                  │                  │     │
│  EVERYTHING:        EVERYTHING:        EVERYTHING:           │     │
│  - cert-manager     - cert-manager     - cert-manager        │     │
│  - cnpg operator    - cnpg operator    - cnpg operator       │     │
│  - strimzi operator - strimzi operator - strimzi operator    │     │
│  - redis operator   - redis operator   - redis operator      │     │
│  - external-secrets - external-secrets - external-secrets    │     │
│  - vault            - vault            - vault               │     │
│  - postgres         - postgres         - postgres            │     │
│  - redis            - redis            - redis               │     │
│  - kafka            - kafka            - kafka               │     │
│  - apps...          - apps...          - apps...             │     │
│                                                               │     │
│  Can scale to 0 ✓   Can scale to 0 ✓   Always running        │     │
│                                                               │     │
└─────────────────────────────────────────────────────────────────────┘
```

## Resource Duplication Strategy

### What Gets Duplicated Per Environment Namespace

| Resource | Duplicated? | Reason |
|----------|-------------|--------|
| cert-manager operator | Yes | Independent TLS management |
| external-secrets operator | Yes | Namespace-scoped secret sync |
| cloudnative-pg operator | Yes | Independent PostgreSQL management |
| strimzi-kafka operator | Yes | Independent Kafka management |
| redis operator | Yes | Independent Redis management |
| clickhouse operator | Yes | Independent ClickHouse management |
| vault | Yes | Per-environment secret management |
| PostgreSQL clusters | Yes | Data isolation |
| Redis clusters | Yes | Cache isolation |
| Kafka clusters | Yes | Event isolation |
| Application deployments | Yes | Environment-specific versions |
| Prometheus | Yes | Environment-specific metrics |
| Loki | Yes | Environment-specific logs |

### What Remains Cluster-Scoped

| Resource | Location | Reason |
|----------|----------|--------|
| ArgoCD | `argocd` namespace | Single GitOps controller |
| ArgoCD cert-manager | `argocd` namespace | ArgoCD TLS independence |
| CRDs | Cluster-scoped | API definitions shared by all |
| ClusterRoles | Cluster-scoped | RBAC definitions |
| ClusterIssuers | Cluster-scoped | But managed by per-namespace cert-managers |
| StorageClasses | Cluster-scoped | Storage provisioners |

## ApplicationSet Configuration

### How Namespace Inheritance Works

The ApplicationSet matrix generator provides the namespace to each component:

```yaml
# argocd/applicationsets/ameide.yaml
generators:
  - matrix:
      generators:
        - list:
            elements:
              - env: dev
                namespace: ameide-dev      # ← Environment namespace
              - env: staging
                namespace: ameide-staging
              - env: production
                namespace: ameide-prod
        - git:
            files:
              - path: environments/_shared/components/**/component.yaml

template:
  spec:
    destination:
      namespace: '{{ .namespace }}'  # ← Inherited from list generator
```

### Component Configuration

Components do NOT specify a namespace - they inherit from the ApplicationSet:

```yaml
# environments/_shared/components/foundation/operators/cloudnative-pg/component.yaml
# CNPG operator - deploys to environment namespace (ameide-dev, ameide-staging, ameide-prod)
name: foundation-cloudnative-pg
project: ameide
# namespace: inherited from ApplicationSet ({{ .namespace }})
domain: foundation
dependencyPhase: "controllers"
componentType: "operator"
rolloutPhase: "220"
```

## Cert-Manager Architecture

### Multiple Cert-Manager Instances

The cluster runs **multiple cert-manager instances**, each managing certificates for its namespace:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Cert-Manager Instances                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  argocd namespace                                                   │
│  ├── argocd-cert-manager (v1.16.2)                                 │
│  ├── ClusterIssuer: letsencrypt-argocd                             │
│  └── Certificate: argocd-server → argocd-ameide-io-tls             │
│      DNS Zone: ameide.io                                            │
│                                                                      │
│  ameide-dev namespace                                               │
│  ├── foundation-cert-manager (v1.16.2)                             │
│  ├── ClusterIssuer: letsencrypt-dev-dns01                          │
│  └── Certificate: ameide-wildcard-dev → ameide-wildcard-tls        │
│      DNS Zone: dev.ameide.io                                        │
│                                                                      │
│  ameide-staging namespace                                           │
│  ├── foundation-cert-manager (v1.16.2)                             │
│  ├── ClusterIssuer: letsencrypt-staging-dns01                      │
│  └── Certificate: ameide-wildcard-staging → ameide-wildcard-tls    │
│      DNS Zone: staging.ameide.io                                    │
│                                                                      │
│  ameide-prod namespace                                              │
│  ├── foundation-cert-manager (v1.16.2)                             │
│  ├── ClusterIssuer: letsencrypt-prod-dns01                         │
│  └── Certificate: ameide-wildcard-prod → ameide-wildcard-tls       │
│      DNS Zone: ameide.io                                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Why Multiple Cert-Managers?

1. **Isolation**: Environment cert-manager failure doesn't affect ArgoCD or other environments
2. **Independent lifecycle**: Each can be upgraded/configured independently
3. **Cleaner failures**: Issues are contained to one namespace
4. **Workload Identity scoping**: Each cert-manager uses its own managed identity

## Network Isolation

### Cross-Namespace NetworkPolicy

Environments are network-isolated from each other:

```yaml
# Created by foundation-namespaces for each environment
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-environment
  namespace: ameide-dev  # (or ameide-staging, ameide-prod)
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Allow from same environment namespace
    - from:
        - namespaceSelector:
            matchLabels:
              ameide.io/environment: dev
    # Allow from argocd (for GitOps sync operations)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: argocd
    # Allow from kube-system (DNS, metrics-server)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
```

### Allowed Cross-Namespace Traffic

| Source | Destination | Purpose | Allowed? |
|--------|-------------|---------|----------|
| argocd | ameide-dev | GitOps sync | Yes |
| argocd | ameide-staging | GitOps sync | Yes |
| argocd | ameide-prod | GitOps sync | Yes |
| kube-system | ameide-* | DNS, metrics | Yes |
| ameide-dev | ameide-staging | Cross-env access | No |
| ameide-dev | ameide-prod | Cross-env access | No |
| ameide-staging | ameide-prod | Cross-env access | No |
| External | ameide-dev (via Envoy) | User traffic | Yes |

## Deployment Order

### Bootstrap Sequence

```
1. Terraform creates infrastructure
   └── AKS, KeyVault, DNS zones, managed identities

2. sync-globals.sh cluster
   └── Populates sources/values/cluster/globals.yaml

3. bootstrap.sh --install-argo
   └── Installs ArgoCD in argocd namespace

4. bootstrap.sh --apply-root-apps
   └── Applies argocd/bootstrap-app.yaml
       └── ArgoCD syncs argocd-config Application
           ├── argocd-cert-manager (sync-wave: -1)
           ├── argocd-tls (sync-wave: 0)
           ├── Projects
           └── ApplicationSets

5. ArgoCD ApplicationSets generate environment apps
   └── All apps deploy to environment namespace (ameide-dev, etc.)
   └── Operators: dev-foundation-cert-manager, dev-foundation-cloudnative-pg, ...
   └── Data: dev-data-postgres-cluster, dev-data-kafka-cluster, ...
   └── Platform: dev-platform-envoy-gateway, dev-platform-keycloak, ...
   └── Apps: dev-www-ameide, dev-inference, ...
```

### Sync Wave Order Within Environments

Each environment follows the same sync wave order (via rolloutPhase):

```
010: Namespace (foundation-namespaces)
110: CRDs (foundation-crds-*)
120: Operators (foundation-cert-manager, foundation-external-secrets, ...)
130: Secrets infrastructure (vault-webhook-certs)
140: Configs (coredns-config, managed-storage)
150: State stores (vault-core)
199: Smoke tests (operators-smoke)
220: Data operators (cloudnative-pg, strimzi, redis, clickhouse)
250: Data runtimes (postgres-clusters, kafka-cluster, redis-failover)
350: Platform runtimes (envoy-gateway, keycloak, prometheus, grafana)
650: Application runtimes (www-ameide, inference, agents, ...)
```

## Failure Scenarios

### Scenario 1: Dev Namespace Down

**Impact:**
- Dev applications unavailable
- Dev cert-manager not running
- Dev operators not running (CNPG, Strimzi, etc.)
- Dev certificates may expire (if down > 30 days)

**Not Impacted:**
- ArgoCD (has own cert-manager)
- Staging environment (has its own operators)
- Production environment (has its own operators)

### Scenario 2: ArgoCD Namespace Down

**Impact:**
- No GitOps reconciliation
- No new deployments
- No self-healing

**Not Impacted:**
- Running workloads in all environments (continue running)
- Existing certificates (continue working)
- Data (persisted)

**Recovery:**
```bash
./bootstrap/bootstrap.sh --install-argo --apply-root-apps
```

### Scenario 3: cert-manager CRDs Deleted

**Impact:**
- All cert-manager instances fail (argocd + all environments)
- Certificate resources become orphaned
- TLS certificates stop renewing

**Recovery:**
1. ArgoCD resync restores CRDs (foundation-crds-cert-manager)
2. cert-manager instances restart
3. Certificates resume renewal

## Monitoring

### Key Metrics Per Namespace

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| `cert_manager_certificate_ready_status` | cert-manager | `!= 1` for 15m |
| `argocd_app_health_status` | ArgoCD | `!= Healthy` for 10m |
| Namespace pod count | kube-state-metrics | `== 0` when state is "on" |

### Dashboard Organization

```
Grafana Dashboards:
├── Cluster Overview
│   ├── ArgoCD health
│   ├── Node pool status
│   └── Cross-namespace traffic
├── ArgoCD Namespace
│   ├── cert-manager health
│   └── Application sync status
├── Dev Environment
│   ├── All operators health
│   ├── cert-manager health
│   ├── Application health
│   └── Resource usage
├── Staging Environment
│   └── ...
└── Production Environment
    └── ...
```

## Implementation Checklist

- [x] ArgoCD has isolated cert-manager instance
- [x] ArgoCD TLS independent of environment namespaces
- [x] Cluster globals separated from environment globals
- [x] sync-globals.sh supports cluster target
- [x] All operators deploy to environment namespace (not cluster-scoped)
- [x] Component.yaml files inherit namespace from ApplicationSet
- [x] foundation-namespaces only creates environment namespace
- [x] NetworkPolicies deployed for cross-namespace isolation
- [x] Documentation for isolation architecture (this doc)
- [ ] Monitoring dashboards per namespace
- [ ] Runbook for namespace failure recovery

## Implementation Commits

- `refactor: implement 446 namespace isolation - operators per environment`
- `fix: remove platform-storage namespace creation (cluster-scoped StorageClass)`
