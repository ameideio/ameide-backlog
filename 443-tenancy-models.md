# 443 – Tenancy Model Overview

**Created**: 2025-12-04
**Updated**: 2025-12-04

> **Related documents:**
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) – Environment naming, namespace conventions, GitOps structure
> - [439-deploy-infrastructure.md](439-deploy-infrastructure.md) – Infrastructure deployment and bootstrap
> - [440-storage-concerns.md](440-storage-concerns.md) – Storage architecture
> - [441-networking.md](441-networking.md) – Networking and isolation
> - [442-environment-isolation.md](442-environment-isolation.md) – Environment isolation and node pools

## Overview

AMEIDE supports three tenancy models (SKUs) that can be deployed independently or in combination. This document serves as the **single source of truth** for tenancy concepts referenced across all architecture documents.

## Two Orthogonal Axes

All deployments are characterized by two independent axes:

```
                    Tenant Axis
                        │
    ┌───────────────────┼───────────────────┐
    │                   │                   │
    │   Shared SaaS     │  Namespace-per-   │
    │   (multi-tenant   │  tenant           │
    │    app-level)     │  (namespace       │
    │                   │   isolation)      │
    │                   │                   │
────┼───────────────────┼───────────────────┼──── Environment Axis
    │                   │                   │     (dev/staging/prod)
    │   Private Cloud   │                   │
    │   (dedicated      │                   │
    │    cluster)       │                   │
    │                   │                   │
    └───────────────────┴───────────────────┘
```

- **Environment axis**: `dev`, `staging`, `production` – Controls SDLC stage
- **Tenant axis**: `shared`, `namespace`, `private` – Controls isolation level

## SKU Matrix

| SKU | Cluster Pattern | Namespace Pattern | Isolation Level | Target Customer |
|-----|-----------------|-------------------|-----------------|-----------------|
| **Shared SaaS** | Single `ameide` cluster | `ameide-prod` | Application-level (row-level) | SMB, self-serve |
| **Namespace-per-tenant** | Single `ameide` cluster per region | `tenant-<slug>-prod` | Kubernetes namespace | Mid-market |
| **Private Cloud** | Dedicated `ameide-<slug>` cluster | `ameide-dev/staging/prod` | Full cluster | Enterprise, regulated |

### Shared SaaS

```
┌─────────────────────────────────────────────────────────────────┐
│  AKS Cluster: ameide                                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Namespace: ameide-prod                                       ││
│  │                                                              ││
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐                        ││
│  │  │ Tenant A│ │ Tenant B│ │ Tenant C│  ← App-level isolation ││
│  │  │ (rows)  │ │ (rows)  │ │ (rows)  │                        ││
│  │  └─────────┘ └─────────┘ └─────────┘                        ││
│  │                                                              ││
│  │  Shared: PostgreSQL, Redis, Kafka, MinIO                    ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

- **Domains**: `*.ameide.io`
- **Database**: Multi-tenant schema with `tenant_id` column
- **Billing**: Usage-based (API calls, storage)
- **Deployment**: Single ApplicationSet element

### Namespace-per-Tenant

```
┌─────────────────────────────────────────────────────────────────┐
│  AKS Cluster: ameide                                             │
│  ┌───────────────────┐ ┌───────────────────┐ ┌─────────────────┐│
│  │ tenant-acme-prod  │ │ tenant-contoso-prod│ │ ameide-prod    ││
│  │                   │ │                   │ │ (internal)      ││
│  │  ┌─────────────┐  │ │  ┌─────────────┐  │ │                 ││
│  │  │ Platform    │  │ │  │ Platform    │  │ │                 ││
│  │  │ Graph       │  │ │  │ Graph       │  │ │                 ││
│  │  │ Inference   │  │ │  │ Inference   │  │ │                 ││
│  │  └─────────────┘  │ │  └─────────────┘  │ │                 ││
│  │                   │ │                   │ │                 ││
│  │  Separate DB      │ │  Separate DB      │ │                 ││
│  └───────────────────┘ └───────────────────┘ └─────────────────┘│
│                                                                  │
│  Shared: CNPG Operator, Redis Operator, Kafka Operator          │
└─────────────────────────────────────────────────────────────────┘
```

- **Domains**: `*.<slug>.ameide.io` or custom domain
- **Database**: Separate database per tenant (shared CNPG cluster)
- **Billing**: Per-seat or flat fee
- **Deployment**: Additional ApplicationSet elements per tenant

### Private Cloud

```
┌─────────────────────────────────────────────────────────────────┐
│  Customer Azure Subscription                                     │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  AKS Cluster: ameide-acme                                    ││
│  │  ┌─────────────────────────────────────────────────────────┐││
│  │  │ Namespace: ameide-prod                                   │││
│  │  │                                                          │││
│  │  │  Full AMEIDE stack, dedicated resources                  │││
│  │  │  Customer-controlled data residency                      │││
│  │  └─────────────────────────────────────────────────────────┘││
│  │                                                              ││
│  │  Dedicated: PostgreSQL, Redis, Kafka, Storage Account       ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

- **Domains**: `*.ameide-<slug>.io` or `*.customer-domain.tld`
- **Database**: Dedicated CNPG cluster
- **Billing**: Azure Managed Application (monthly fee)
- **Deployment**: Separate Bicep deployment per tenant
- **See**: [148-azure-managed-offering.md](148-azure-managed-offering.md) for Marketplace packaging

---

## Label Reference

All workloads MUST be labeled consistently to support isolation policies.

### Required Labels

| Label | Values | Scope | Description |
|-------|--------|-------|-------------|
| `ameide.io/environment` | `dev`, `staging`, `production`, `cluster` | Namespace, Pod | SDLC stage |
| `ameide.io/tenant-kind` | `shared`, `namespace`, `private` | Namespace, Pod | Tenancy model |
| `ameide.io/tenant-id` | `<tenant-slug>` | Namespace, Pod | Tenant identifier (omit for shared) |
| `ameide.io/tier` | `frontend`, `backend`, `data`, `platform`, `foundation` | Pod | Service tier for NetworkPolicy |

### Label Examples

**Shared SaaS namespace (`ameide-prod`):**
```yaml
metadata:
  name: ameide-prod
  labels:
    ameide.io/environment: production
    ameide.io/tenant-kind: shared
    # No tenant-id for shared
```

**Namespace-per-tenant (`tenant-acme-prod`):**
```yaml
metadata:
  name: tenant-acme-prod
  labels:
    ameide.io/environment: production
    ameide.io/tenant-kind: namespace
    ameide.io/tenant-id: acme
```

**Private cloud namespace (`ameide-prod` in dedicated cluster):**
```yaml
metadata:
  name: ameide-prod
  labels:
    ameide.io/environment: production
    ameide.io/tenant-kind: private
    ameide.io/tenant-id: acme
```

**Pod labels (backend service):**
```yaml
metadata:
  labels:
    ameide.io/environment: production
    ameide.io/tenant-kind: namespace
    ameide.io/tenant-id: acme
    ameide.io/tier: backend
```

---

## Namespace Naming Conventions

| Tenancy Model | Namespace Pattern | Examples |
|---------------|-------------------|----------|
| Shared SaaS | `ameide-<env>` | `ameide-dev`, `ameide-staging`, `ameide-prod` |
| Namespace-per-tenant | `tenant-<slug>-<env>` | `tenant-acme-prod`, `tenant-contoso-staging` |
| Private cloud | `ameide-<env>` (in tenant cluster) | `ameide-prod` in cluster `ameide-acme` |

### Internal vs Tenant Namespaces

```
Shared cluster (ameide):
├── ameide-dev          # Internal development
├── ameide-staging      # Internal staging
├── ameide-prod         # Shared SaaS production
├── tenant-acme-prod    # Namespace tenant: Acme
├── tenant-contoso-prod # Namespace tenant: Contoso
├── argocd              # Cluster-scoped (environment: cluster)
└── cert-manager        # Cluster-scoped (environment: cluster)
```

---

## Domain Routing Patterns

| Tenancy Model | Domain Pattern | HTTPRoute Target |
|---------------|----------------|------------------|
| Shared SaaS | `*.ameide.io` | `ameide-prod` namespace |
| Namespace-per-tenant | `*.<tenant>.ameide.io` | `tenant-<tenant>-prod` namespace |
| Namespace-per-tenant | Custom domain (CNAME) | `tenant-<tenant>-prod` namespace |
| Private cloud | `*.ameide-<tenant>.io` | `ameide-prod` in tenant cluster |
| Private cloud | Custom domain | `ameide-prod` in tenant cluster |

---

## Infrastructure by SKU

### Bicep Deployment Modes

| SKU | Bicep Usage | Subscription |
|-----|-------------|--------------|
| Shared SaaS | `deploy-managed-app.sh staging/production` | AMEIDE subscription |
| Namespace-per-tenant | No new Bicep; GitOps only | AMEIDE subscription |
| Private cloud | `deploy-managed-app.sh` with tenant params | Customer subscription |

See [439-deploy-infrastructure.md](439-deploy-infrastructure.md) for details.

### Compute Resources

| SKU | Node Pools | Node Affinity |
|-----|------------|---------------|
| Shared SaaS | system, dev, staging, prod | Workloads use environment pool |
| Namespace-per-tenant | Same shared pools | `tenant-*-prod` → `prod` pool |
| Private cloud | Dedicated 4-pool strategy | Same pattern in tenant cluster |

See [442-environment-isolation.md](442-environment-isolation.md) for node pool strategy.

### Storage Resources

| SKU | Database | Object Storage | Backup |
|-----|----------|----------------|--------|
| Shared SaaS | Shared CNPG, multi-tenant schema | Shared MinIO, tenant prefix | Shared backup location |
| Namespace-per-tenant | Shared CNPG cluster, separate DB | Shared MinIO, `/<tenant>/` prefix | Per-tenant backup path |
| Private cloud | Dedicated CNPG cluster | Dedicated Storage Account | Customer-controlled |

See [440-storage-concerns.md](440-storage-concerns.md) for storage architecture.

### Network Isolation

| SKU | Isolation Mechanism | NetworkPolicy |
|-----|---------------------|---------------|
| Shared SaaS | Application-level (tenant_id) | Cross-env deny only |
| Namespace-per-tenant | Kubernetes namespace | Cross-env + cross-tenant deny |
| Private cloud | Cluster boundary | Standard cross-env deny |

See [441-networking.md](441-networking.md) for NetworkPolicy templates.

---

## GitOps Configuration

### ApplicationSet Strategy

**Shared SaaS** uses the standard environment list:
```yaml
generators:
  - list:
      elements:
        - env: production
          tenantKind: shared
          namespace: ameide-prod
```

**Namespace-per-tenant** extends the list:
```yaml
generators:
  - list:
      elements:
        # Shared SaaS
        - env: production
          tenantKind: shared
          namespace: ameide-prod
        # Namespace tenants
        - env: production
          tenantKind: namespace
          tenantId: acme
          namespace: tenant-acme-prod
        - env: production
          tenantKind: namespace
          tenantId: contoso
          namespace: tenant-contoso-prod
```

**Private cloud** uses separate GitOps repos or branches per tenant.

### Values Structure

```
sources/values/
├── _shared/                    # Shared across all environments
├── dev/                        # Internal dev
├── staging/                    # Internal staging
├── production/                 # Shared SaaS production
└── tenants/                    # Namespace-per-tenant overrides
    ├── acme/
    │   ├── globals.yaml        # Tenant-specific globals
    │   └── apps/               # Tenant-specific app values
    └── contoso/
        └── globals.yaml
```

> **Scaling note**: For >20 tenants, consider replacing the `list` generator with a
> `git` generator that reads tenant definitions from `sources/values/tenants/*/tenant.yaml` files.

See [434-unified-environment-naming.md](434-unified-environment-naming.md) for GitOps structure.

---

## Decision Tree: Which SKU?

```
Start
  │
  ├─ Does customer require dedicated infrastructure?
  │   │
  │   ├─ Yes → Does customer require their own Azure subscription?
  │   │         │
  │   │         ├─ Yes → Private Cloud (Managed App)
  │   │         │
  │   │         └─ No → Private Cloud (in AMEIDE subscription)
  │   │
  │   └─ No → Does customer require namespace isolation?
  │            │
  │            ├─ Yes → Namespace-per-tenant
  │            │
  │            └─ No → Shared SaaS
```

### SKU Selection Criteria

| Requirement | Shared SaaS | Namespace-per-tenant | Private Cloud |
|-------------|-------------|----------------------|---------------|
| Data residency in customer subscription | - | - | **Required** |
| SOC 2 Type II dedicated audit | - | Possible | **Required** |
| Custom domain | Possible | **Standard** | **Standard** |
| Dedicated compute | - | Optional (premium) | **Standard** |
| White-label branding | Possible | **Standard** | **Standard** |
| Self-managed upgrades | - | - | **Optional** |
| Price point | $ | $$ | $$$ |

---

## Implementation Status

| Component | Status | Notes |
|-----------|--------|-------|
| Label definitions | ✅ Documented | This document |
| Tier labels in app values | ✅ Implemented | All app values have `tier`, `domain`, `exposure` labels |
| Tier label helpers in templates | ✅ Implemented | `_helpers.tpl` includes `tierLabels` helper for `ameide.io/tier` and `ameide.io/environment` |
| Dynamic namespace in templates | ✅ Implemented | All shared values use `{{ .Release.Namespace }}` instead of hardcoded `ameide` |
| Shared SaaS | ✅ Implemented | Current production model |
| Cross-env NetworkPolicy | ✅ Implemented | `deny-cross-environment` in `foundation-namespaces.yaml` |
| Tier-based NetworkPolicy | ✅ Implemented | `tier-based-ingress` in `foundation-namespaces.yaml` |
| Cross-tenant NetworkPolicy | ✅ Implemented | `deny-cross-tenant` ready in `foundation-namespaces.yaml` |
| Rate limiting | ✅ Implemented | `rate-limit-policy.yaml` with per-env values |
| Private cloud Bicep | ✅ Implemented | deploy-managed-app.sh supports tenant params |
| Namespace-per-tenant GitOps | ⏳ Pending | ApplicationSet extension needed per tenant |
| Per-tenant values structure | ✅ Implemented | `sources/values/tenants/_template/` with README |

### Recent Implementation (2025-12-04)

**Namespace Parameterization Complete:**
- Removed all hardcoded `namespace: ameide` from shared values files
- ExternalSecret manifests now use `{{ .Release.Namespace }}`
- CNPG postgres cluster uses dynamic namespace (removed `namespaceOverride`)
- All services use short DNS names (Kubernetes resolves within namespace)

**Tier Labels Implemented:**
- All app values files (`sources/values/_shared/apps/*.yaml`) include:
  - `tier: backend|frontend|data|platform`
  - `domain: apps|data|platform|foundation`
  - `exposure: internal|public`
- Chart `_helpers.tpl` files include `tierLabels` helper that outputs:
  - `ameide.io/tier: {{ .Values.tier }}`
  - `ameide.io/environment: {{ .Values.environment }}`

**Files Modified:**
- 12 app chart `_helpers.tpl` and `deployment.yaml` files
- 29 shared values files under `sources/values/_shared/`
- CNPG, Kafka, Clickhouse operator config templates

See commits: `ca030c2`, `05f8f00`, `075d05e`

---

## Backlog Items

### Immediate

- [x] **TENANT-1**: Add tier labels to app Helm charts (`tier`, `domain`, `exposure` in values; `ameide.io/tier`, `ameide.io/environment` via `_helpers.tpl`) → **2025-12-04**
- [x] **TENANT-1b**: Add tenant labels (`ameide.io/tenant-kind`, `ameide.io/tenant-id`) to namespace values template → **2025-12-04**
- [x] **TENANT-2**: Create `sources/values/tenants/` directory structure → **2025-12-04**
- [x] **TENANT-3**: Add cross-tenant NetworkPolicy (`deny-cross-tenant`) to foundation-namespaces.yaml → **2025-12-04**
- [x] **TENANT-4**: Document tenant-aware ApplicationSet generator example in 434 → **2025-12-04**

### Future

- [ ] **TENANT-5**: Tenant onboarding automation (CLI or API)
- [ ] **TENANT-6**: Per-tenant resource quotas
- [ ] **TENANT-7**: Tenant billing integration
- [ ] **TENANT-8**: Premium tenant node pools (dedicated compute option)
- [ ] **TENANT-9**: Per-tenant shutdown capability (extend environmentState)

---

## References

- [434-unified-environment-naming.md](434-unified-environment-naming.md) – Environment naming and GitOps
- [439-deploy-infrastructure.md](439-deploy-infrastructure.md) – Bicep deployment
- [440-storage-concerns.md](440-storage-concerns.md) – Storage architecture
- [441-networking.md](441-networking.md) – Network isolation
- [442-environment-isolation.md](442-environment-isolation.md) – Compute isolation
- [148-azure-managed-offering.md](148-azure-managed-offering.md) – Azure Managed Application
