# 440 – Storage Architecture Improvements

> **Related documents:**
> - [443-tenancy-models.md](443-tenancy-models.md) – **Tenancy model overview (single source of truth)**
> - [441-networking.md](441-networking.md) – Networking improvements
> - [442-environment-isolation.md](442-environment-isolation.md) – Environment isolation
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) – Environment naming conventions
> - [240-cluster-rightsizing.md](240-cluster-rightsizing.md) – Cluster resource planning

## Implementation Status (2025-12-04)

| Component | Status | Notes |
|-----------|--------|-------|
| **Premium storage class** | ✅ Done | PostgreSQL, Redis, ClickHouse, Kafka updated to `managed-csi-premium` |
| **Kafka persistent storage** | ✅ Done | Template updated with configurable persistent-claim storage |
| **CNPG backups Bicep** | ✅ Done | `storageAccount.bicep`, `backupIdentity.bicep` modules created |
| **CNPG backups Terraform** | ✅ Done | Backup modules created in `modules/azure-identity/` |
| **IaC backup output parity** | ✅ Done | `backup_storage_account_name`, `backup_identity_client_id` in both Bicep & Terraform |
| **CNPG backups deployment** | ⏳ Pending | Run Bicep/Terraform with `enableBackupStorage=true`, then enable in CNPG values |
| **Node affinity (data tier)** | ⏳ Blocked | Waiting for node pools from 442 |

> **Consistency testing**: Run `./infra/scripts/test-iac-consistency.sh schema` to verify Bicep and Terraform output parity.

## Problem Statement

Current storage configuration has several concerns that need to be addressed for production readiness:

1. **Kafka ephemeral storage**: Data loss on pod restart
2. **No backup strategy**: No CNPG scheduled backups or Velero configured
3. **Single storage tier**: No differentiation between performance tiers (premium vs standard)
4. **No data-tier node affinity**: Data workloads not pinned to appropriate node pools

## Current State (Verified 2025-12-04)

### AKS Cluster Storage

The cluster runs on **Azure Kubernetes Service** with native Azure Disk CSI driver:

```bash
# Actual cluster configuration
az aks show --name ameide --resource-group Ameide
# Location: westeurope
# Kubernetes: 1.32.7
# Node SKU: Standard_D8as_v6 (8 vCPU, 32Gi RAM)
```

### StorageClass Configuration

AKS provides built-in storage classes with Azure Disk CSI:

| StorageClass | Provisioner | SKU | Expansion | Status |
|--------------|-------------|-----|-----------|--------|
| `managed-csi` | disk.csi.azure.com | StandardSSD_LRS | ✅ Yes | Default |
| `managed-csi-premium` | disk.csi.azure.com | Premium_LRS | ✅ Yes | Available |
| `azurefile-csi` | file.csi.azure.com | Standard_LRS | ✅ Yes | Available |

> **Note**: The local `storageclass-managed-csi.yaml` file in the repo is a **fallback for k3d local development** only. On AKS, the Azure-managed classes take precedence.

### Storage by Component

| Component | Storage Class | Dev | Staging | Production | Issues |
|-----------|--------------|-----|---------|------------|--------|
| PostgreSQL CNPG | managed-csi-premium | 10Gi | 10Gi | 200Gi | Backup config ready, needs Azure SA |
| Redis Failover | managed-csi-premium | 8Gi | 8Gi | 8Gi | ✅ |
| Kafka (Strimzi) | managed-csi-premium | 20Gi | 20Gi | 20Gi | ✅ Persistent storage configured |
| ClickHouse | managed-csi-premium | 20Gi | 20Gi | 50Gi | ✅ |
| MinIO | managed-csi | disabled | 50Gi | 100Gi | ✅ (standard OK for bulk) |
| Tempo | managed-csi | 10Gi | 10Gi | 50Gi | ✅ (standard OK for bulk) |
| Loki | S3 (MinIO) | - | - | - | Depends on MinIO |
| Prometheus | managed-csi | 50Gi | 50Gi | 100Gi | ✅ (standard OK for bulk) |

### Storage Tier Recommendations

| Workload Type | Recommended StorageClass | Rationale |
|---------------|-------------------------|-----------|
| PostgreSQL, ClickHouse | `managed-csi-premium` | IOPS-sensitive, low latency |
| Kafka, Redis | `managed-csi-premium` | Write-heavy, throughput |
| MinIO, Tempo, Loki | `managed-csi` (standard) | Bulk storage, cost-sensitive |
| Prometheus | `managed-csi` | Retention-based, moderate IOPS |

### Tenancy-aware Data Isolation

Storage strategy differs by tenancy model (see [443-tenancy-models.md](443-tenancy-models.md)):

| Tenancy Model | Database Strategy | Object Storage Strategy | Cost/Complexity |
|---------------|-------------------|-------------------------|-----------------|
| Shared SaaS | Single CNPG cluster, multi-tenant schema | Shared MinIO/Azure Blob with tenant prefix | Low |
| Namespace-per-tenant | Shared CNPG cluster, separate databases per tenant | Shared MinIO/Azure Blob, `/<tenant-id>/…` prefix | Medium |
| Namespace-per-tenant (premium) | Dedicated CNPG cluster per namespace | Shared MinIO/Azure Blob, `/<tenant-id>/…` prefix | High |
| Private cloud | Dedicated CNPG cluster per tenant cluster | Dedicated Storage Account per tenant | Per-tenant |

> **Default recommendation**: Shared CNPG cluster with separate databases per tenant. Row-level security optional.

**Naming conventions for tenant isolation:**

- Every PVC name SHOULD include environment and tenant (where applicable):
  - `pgdata-<env>-<tenant-id>-<service>` (e.g., `pgdata-prod-acme-platform`)
- CNPG database names MUST include tenant id for namespace-per-tenant:
  - `ameide_<tenant-id>` (e.g., `ameide_acme`)
- MinIO bucket prefixes MUST include tenant id for namespace-per-tenant and private cloud:
  - `ameide/<env>/<tenant-id>/postgres/…`
  - `ameide/<env>/<tenant-id>/minio/…`

**Example: Namespace-per-tenant PostgreSQL**

```yaml
# sources/values/tenants/acme/data/platform-postgres-clusters.yaml
cluster:
  name: postgres-tenant-acme
  storage:
    storageClass: managed-csi-premium
    size: 50Gi
  backup:
    barmanObjectStore:
      destinationPath: "s3://ameide-backups/production/acme/postgres/"
```

### Key Files

- `sources/charts/foundation/common/raw-manifests/files/storageclass-managed-csi.yaml`
- `sources/charts/foundation/operators-config/kafka_cluster/templates/kafka.yaml`
- `sources/values/_shared/data/platform-postgres-clusters.yaml`
- `sources/values/_shared/data/data-redis-failover.yaml`
- `sources/values/_shared/data/data-clickhouse.yaml`

---

## Proposed Changes

### 1. Use Premium Storage for Data-Tier Workloads

**Priority**: HIGH

AKS already provides `managed-csi-premium`. Update data workloads to use it:

**File**: `sources/values/_shared/data/platform-postgres-clusters.yaml`

```yaml
cluster:
  storage:
    storageClass: managed-csi-premium  # Changed from managed-csi
    size: 10Gi  # Dev default, overridden per-env
```

**File**: `sources/values/env/production/data/platform-postgres-clusters.yaml`

```yaml
cluster:
  storage:
    storageClass: managed-csi-premium
    size: 200Gi
```

### 2. Add Data-Tier Node Affinity

**Priority**: MEDIUM

When node pools are implemented (see [442-environment-isolation.md](442-environment-isolation.md)), pin data workloads to the appropriate environment pool:

```yaml
# For StatefulSets (PostgreSQL, Kafka, Redis)
spec:
  template:
    spec:
      nodeSelector:
        ameide.io/pool: prod  # From globals.yaml
      tolerations:
        - key: "ameide.io/environment"
          value: "production"
          effect: "NoSchedule"
```

> **Note**: This depends on node pool strategy defined in 442. Initially, all workloads share the same pool.

**Tenant node affinity:**

When node pools from 442 are deployed, all data-tier StatefulSets SHOULD use `nodeSelector` and `tolerations` that target the environment pool (`prod`, `staging`, `dev`), regardless of tenant.

- **Shared SaaS**: Data workloads run on environment pool (e.g., `prod` pool)
- **Namespace-per-tenant**: Tenant data workloads (`tenant-acme-prod`) run on `prod` pool alongside shared SaaS
- **Private cloud**: Tenant cluster has its own 4-pool strategy

Tenants are isolated via namespaces and PVC/database naming, not dedicated data pools by default. Large tenants MAY receive dedicated data pools as a premium option (extension of the 4-pool strategy).

### 3. Enable Kafka Persistent Storage

**Priority**: HIGH

Update Kafka to use persistent storage:

**File**: `sources/charts/foundation/operators-config/kafka_cluster/templates/kafka.yaml`

```yaml
# Change from:
storage:
  type: ephemeral

# To:
storage:
  type: persistent-claim
  size: 20Gi
  class: managed-csi-premium
  deleteClaim: false
```

**File**: `sources/values/_shared/data/data-kafka-cluster.yaml`

```yaml
kafka:
  storage:
    type: persistent-claim
    size: 20Gi
    class: managed-csi-premium
    deleteClaim: false
```

### 4. Implement CNPG Backup Strategy

**Priority**: HIGH

Enable scheduled backups for PostgreSQL:

**File**: `sources/values/_shared/data/platform-postgres-clusters.yaml`

```yaml
cluster:
  backup:
    barmanObjectStore:
      destinationPath: "s3://ameide-backups/postgres/"
      endpointURL: "https://ameidesa.blob.core.windows.net"
      s3Credentials:
        accessKeyId:
          name: azure-backup-credentials
          key: accessKeyId
        secretAccessKey:
          name: azure-backup-credentials
          key: secretAccessKey
      wal:
        compression: gzip
        maxParallel: 2
    retentionPolicy: "7d"

  # Scheduled backups
  scheduledBackups:
    - name: daily-backup
      schedule: "0 2 * * *"  # 2 AM daily
      backupOwnerReference: self
      method: barmanObjectStore
```

### 5. Consider Velero for Cluster Backup

**Priority**: LOW (Future)

For full cluster disaster recovery:

```yaml
# velero-values.yaml
configuration:
  provider: azure
  backupStorageLocation:
    bucket: velero-backups
    config:
      resourceGroup: Ameide
      storageAccount: ameidesa
  volumeSnapshotLocation:
    config:
      resourceGroup: Ameide
schedules:
  daily:
    schedule: "0 3 * * *"
    template:
      ttl: "168h"  # 7 days
```

---

## Migration Considerations

### StorageClass Change

1. **New PVCs only**: Existing PVCs cannot change StorageClass
2. **Data migration**: May need to backup/restore for existing volumes
3. **Test in dev**: Validate Azure Disk performance before production

### Kafka Migration

1. **Data loss acceptable**: Current ephemeral means no data anyway
2. **Size calculation**: Monitor topic sizes before setting PVC size
3. **Replication factor**: Consider increasing replicas with persistent storage

---

## Implementation Steps

### Phase 1: Storage Class Assignment
1. [x] Update PostgreSQL to use `managed-csi-premium` in shared values → **2025-12-04**
2. [x] Update Kafka template for configurable persistent storage → **2025-12-04**
3. [x] Update Redis to use `managed-csi-premium` for persistence → **2025-12-04**
4. [x] Update ClickHouse to use `managed-csi-premium` (fixed typo from `managed-premium`) → **2025-12-04**
5. [ ] Test storage provisioning in dev environment

### Phase 2: Backup Strategy
6. [x] Create Bicep module for Azure Storage Account (`modules/storageAccount.bicep`) → **2025-12-04**
7. [x] Create Bicep module for backup identity with workload identity (`modules/backupIdentity.bicep`) → **2025-12-04**
8. [x] Add backup modules to `main.bicep` orchestration → **2025-12-04**
9. [x] Update `infra/scripts/sync-globals.sh` to sync backup outputs → **2025-12-04**
10. [x] Add backup fields to `globals.yaml` for all environments → **2025-12-04**
11. [x] Add CNPG barmanObjectStore config template (disabled, ready to enable) → **2025-12-04**
12. [ ] Deploy Bicep with `enableBackupStorage=true` to create storage account
13. [ ] Run `infra/scripts/sync-globals.sh <env>` to populate backup fields in globals.yaml
14. [ ] Enable CNPG scheduled backups (daily at 2 AM)
15. [ ] Test CNPG backup and restore procedure

### Phase 3: Node Affinity (after 442 node pools)

> **Prerequisite**: Node pools and nodeSelector/tolerations now configured in 442 (2025-12-04).
> Data workloads can use the same pattern once node pools are deployed.

9. [ ] Add `nodeSelector` for data-tier workloads
10. [ ] Add tolerations for data node pool taints
11. [ ] Verify StatefulSets schedule on correct nodes

### Phase 4: Future Improvements
12. [ ] Evaluate Velero for full cluster DR
13. [ ] Consider Azure Files for shared storage needs

---

## Validation Checklist

### Storage Class Configuration (Phase 1)
- [ ] `managed-csi-premium` PVCs provision successfully for PostgreSQL
- [ ] `managed-csi-premium` PVCs provision successfully for Redis
- [ ] `managed-csi-premium` PVCs provision successfully for Kafka
- [ ] `managed-csi-premium` PVCs provision successfully for ClickHouse
- [ ] Volume expansion works (test resize on dev)
- [ ] Kafka pods restart without data loss
- [ ] Storage IOPS meets database requirements

### Backup Strategy (Phase 2)
- [ ] Azure Storage Account created for backups
- [ ] CNPG backups execute on schedule
- [ ] CNPG restore tested successfully

### Node Affinity (Phase 3)
- [ ] Data workloads schedule on correct node pool (after 442 deployment)
