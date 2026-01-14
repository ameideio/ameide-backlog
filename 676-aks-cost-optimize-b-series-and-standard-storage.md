Goal

  - Reduce Azure monthly spend by rotating AKS node pools from D-series to B-series and by standardizing persistent storage to StandardSSD (`managed-csi`).

Status

  - Done (dev-only cluster; data loss acceptable).
  - Git changes:
      - ameide-gitops PR #188: single B-series pool + standard storage values.
      - ameide-gitops PR #190 + #191: fix Azure RBAC lockout (admin group role assignment + CI TF_VAR wiring).
  - CI:
      - terraform-azure-destroy: run 20989178927 (cluster reset).
      - terraform-azure-apply: run 20991673630 (infra + GitOps seeded; failed only at Platform SSO check).
  - Verified outcomes:
      - AKS has one node pool (`system`) on `Standard_B8ms`, labelled `ameide.io/pool=general`.
      - Envoy public IPs for dev/staging/prod are all attached to the AKS LoadBalancer.
      - PVCs observed in-cluster are on `managed-csi` (no `managed-csi-premium`).

Context

  - Current spend drivers include AKS node pool VMs (D8as/D4as) and premium storage for some stateful components (e.g. Kafka, GitLab).
  - GitOps/Single-writer rule: do not mutate clusters manually; all changes must flow through Git + CI (`terraform-azure-apply`) and then ArgoCD reconciliation.

Scope

  - AKS node pool SKU changes (rotation required).
  - Storage class usage changes in GitOps values (note: changing StorageClass does not migrate existing PVs; see “Migration”).

Non-goals

  - Replatforming to a new cluster.
  - Preserving stateful data (this cluster is not production; wipe/redeploy is acceptable).

Inventory (current state)

  - Terraform defaults (Azure):
      - single pool (default/system): `Standard_B8ms` (3 nodes), workloads allowed (`only_critical_addons_enabled=false`), labelled as `ameide.io/pool=general`
      - general pool disabled
    See `infra/terraform/modules/azure-aks/variables.tf`.
  - Storage classes used by charts/values:
      - `managed-csi` (standard) widely used
      - `managed-csi-premium` removed from shared values (cluster reset required to drop existing premium PVs)

Proposal

  1) Node pools: B-series only, single pool
    - Use `Standard_B8ms` for the default/system pool and schedule all workloads there.
    - Ensure safe rotation:
        - Set `temporary_name_for_rotation` for `default_node_pool` so vm size changes do not force cluster replacement.
        - Keep the `ameide.io/pool=general` label contract for existing nodeProfiles.

  2) Storage: standard-only (dev cluster)
    - Switch all stateful charts/CRs to `managed-csi`.
    - Apply via GitOps; if premium PVs already exist, reset the cluster (data loss acceptable).

Migration / Implementation Notes

  - Node pool rotation:
      - Implement via Terraform so AKS drains/rotates nodes and ArgoCD reschedules workloads.
      - Validate readiness: DaemonSets, PDBs, and any node-affinity constraints still allow scheduling onto B-series pools.
  - Storage class changes:
      - Changing `storageClassName` only affects new PVCs.
      - For this cluster, “migration” is achieved by destroy/recreate via CI (data loss acceptable).

Acceptance Criteria

  - AKS runs on B-series only, with a single pool for all workloads.
  - No PVCs are provisioned using `managed-csi-premium`.
  - Post-change: no persistent scheduling failures; no systemic CPU starvation (watch for CPU credit exhaustion on B-series).

Rollback

  - Revert Terraform VM size changes and re-run `terraform-azure-apply` to rotate back to D-series.
  - For storage, rollback requires moving workloads back (cannot “flip” an existing PV’s class); treat as a migration reversal with the same rigor as forward migration.
