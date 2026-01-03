# 531 – Local CNPG Placement vs Control-Plane Taints

**Created**: 2026-01-03
**Updated**: 2026-01-03

> **Status – New:** Track and fix local k3d scheduling + PV placement so control-plane stability policies (tainted server node) don’t break operator-managed stateful workloads (CNPG/Postgres).

## Summary

Local k3d bootstraps taint the control-plane node (`k3d-ameide-server-0`) as `NoSchedule` to reduce apiserver stalls. In some runs, CNPG’s Postgres PVC is provisioned with node affinity pinned to the control-plane node (local-path `WaitForFirstConsumer`), leaving the Postgres pod `Pending` once the taint is applied. This cascades into Keycloak failing to start and ArgoCD Dex crashlooping (OIDC issuer unreachable).

This backlog item makes the long-term behavior deterministic: keep the control-plane tainted, and ensure CNPG pods + their PVs land on an agent node.

## Context / References

- Control-plane taints for local stability: `infra/scripts/tf-local.sh` (`taint_control_plane_nodes()`, log line “reduce local apiserver stalls”).
- Bootstrap note: Dex can be unhealthy during early bootstrap, but it should converge once Keycloak is healthy: `backlog/444-terraform.md` (“don’t wait on Dex”).
- Incident inventory: `backlog/450-argocd-service-issues-inventory.md` (local Dex/Keycloak/bootstrap variants).
- Local stability / apiserver pressure incident class: `backlog/519-gitops-fleet-policy-hardening.md`.

## Problem Statement

Desired local policy:
- Keep k3d control-plane node tainted `NoSchedule` so workloads prefer agents and the apiserver stays responsive.

Failure mode:
- A CNPG Postgres instance schedules onto `k3d-ameide-server-0` before placement rules exist.
- local-path (`WaitForFirstConsumer`) binds the PVC to that node.
- Later, tainting makes restarts unschedulable (PV node affinity ≠ schedulable nodes).
- Keycloak becomes unhealthy (DB not reachable), which makes Dex unhealthy (issuer unreachable) and blocks ArgoCD SSO.

## Root Cause

For StorageClasses with `volumeBindingMode: WaitForFirstConsumer`, PV node selection is based on where the first consumer pod schedules. If CNPG schedules Postgres on the control-plane node early, the PV becomes pinned there. After tainting, the pod cannot reschedule elsewhere, but the PV affinity remains.

## Ideal Fix (GitOps, deterministic)

1. **Local-only CNPG placement policy**
   - Add local values so CNPG Pods avoid nodes labeled `node-role.kubernetes.io/control-plane` and `node-role.kubernetes.io/master`.
   - Use `spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution` so first scheduling (and PV binding) lands on an agent node.

2. **Keep control-plane taints**
   - Maintain the `NoSchedule` taints for local stability.

3. **Repair path for already-pinned PVs**
   - On clusters already affected, delete/recreate the local cluster (or delete the Postgres PVC/PV) so PVs reprovision under the corrected placement rules.

## Temporary Workaround (Stopgap)

Un-taint the control-plane node so pinned PVs can schedule:

```bash
kubectl --context ameide-local taint node k3d-ameide-server-0 node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-
```

This should be treated as a stopgap only.

## Acceptance Criteria

- With control-plane taints enabled, CNPG Postgres pods schedule on an agent node.
- The Postgres PV shows node affinity for an agent node (not `k3d-ameide-server-0`).
- Keycloak reaches `Running` with ready endpoints; Dex stops crashlooping and can reach the issuer.
- A fresh `deploy.sh local` converges without manual taint removal or PVC deletion.

