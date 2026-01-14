Goal

  - Reduce Azure monthly spend by rotating AKS node pools from D-series to B-series and by standardizing persistent storage away from premium classes where feasible.

Context

  - Current spend drivers include AKS node pool VMs (D8as/D4as) and premium storage for some stateful components (e.g. Kafka, GitLab).
  - GitOps/Single-writer rule: do not mutate clusters manually; all changes must flow through Git + CI (`terraform-azure-apply`) and then ArgoCD reconciliation.

Scope

  - AKS node pool SKU changes (rotation required).
  - Storage class usage changes in GitOps values (note: changing StorageClass does not migrate existing PVs; see “Migration”).

Non-goals

  - Replatforming to a new cluster.
  - Data-lossy “delete and recreate” of stateful services without an explicit migration plan.

Inventory (current state)

  - Terraform defaults (Azure):
      - system pool: `Standard_D4as_v6` (2 nodes)
      - general pool: `Standard_D8as_v6` (min 3 / max 10)
    See `infra/terraform/modules/azure-aks/variables.tf`.
  - Storage classes used by charts/values:
      - `managed-csi` (standard) widely used
      - `managed-csi-premium` still used by:
          - Kafka: `sources/values/_shared/data/data-kafka-cluster.yaml`
          - ClickHouse: `sources/values/_shared/data/data-clickhouse.yaml` (env/dev overrides to standard)
          - Redis failover: `sources/values/_shared/data/data-redis-failover.yaml` (env/dev overrides to standard)
          - Postgres clusters: `sources/values/_shared/data/platform-postgres-clusters.yaml` (env overrides exist)
          - GitLab: `sources/values/env/production/platform/platform-gitlab.yaml`

Proposal

  1) Node pools: rotate to B-series
    - Candidate SKUs (region permitting): `Standard_B4ms` / `Standard_B8ms` (final choice depends on steady CPU usage and memory requirements).
    - Keep system pool conservative if needed (kube-system addons can be sensitive to CPU credit exhaustion); alternatively keep system pool D-series and only move “general” workloads to B-series.
    - Ensure safe rotation:
        - General/spot pools already support rotation via `temporary_name_for_rotation`.
        - Confirm `default_node_pool` supports `temporary_name_for_rotation`; if not present, add it so system pool can rotate without cluster replacement.

  2) Storage: move to standard where feasible
    - Change “default” storage class usage for non-critical data services to `managed-csi`.
    - Minimize blast radius by doing this per-environment:
        - Dev/staging: standard
        - Production: standard for low-throughput components; keep premium only where needed (Kafka/GitLab/Postgres based on SLOs).

Migration / Implementation Notes

  - Node pool rotation:
      - Implement via Terraform so AKS drains/rotates nodes and ArgoCD reschedules workloads.
      - Validate readiness: DaemonSets, PDBs, and any node-affinity constraints still allow scheduling onto B-series pools.
  - Storage class changes:
      - Changing `storageClassName` only affects new PVCs.
      - For existing stateful workloads, define a migration per component:
          - Kafka (Strimzi): safest is cluster expansion with new volumes, then decommission old brokers (or planned downtime + restore from backup if available).
          - GitLab: follow chart’s documented backup/restore (or snapshot migration) before changing storage class.
          - Postgres (CNPG): add new cluster with desired storage class, logical replication/pg_dump restore, then cut over.
      - Do not “kubectl patch PVC” in cloud; migrations should be driven by chart/CR changes in Git with explicit runbooks.

Acceptance Criteria

  - AKS node pools run on the selected B-series SKUs (or an explicitly documented mix with D-series system pool).
  - Old D-series pools are removed after rotation completes.
  - Dev/staging have no new PVCs provisioned with `managed-csi-premium`.
  - Production storage class decisions are documented per component (standard vs premium) with rationale.
  - Post-change: no persistent scheduling failures; no systemic CPU starvation (watch for CPU credit exhaustion on B-series).

Rollback

  - Revert Terraform VM size changes and re-run `terraform-azure-apply` to rotate back to D-series.
  - For storage, rollback requires moving workloads back (cannot “flip” an existing PV’s class); treat as a migration reversal with the same rigor as forward migration.
