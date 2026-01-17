---
title: "687 – AKS pool policy: system=AKS-only, platform=control-plane (+ decommission ameidesys)"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-17
suite: "aks"
source_of_truth: false
---

# 687 – AKS pool policy: system=AKS-only, platform=control-plane (+ decommission ameidesys)

This backlog item is the “concrete implementation plan” for the principles in `backlog/686-aks-pool-principles-migration-plan.md`.

It is intentionally written to support:

- **break-glass first** (temporary, documented),
- **Git as source of truth next**,
- **Terraform alignment last**,
- while **not disrupting any service**.

## Desired end state (authoritative)

### Pool definitions (what each pool is allowed to run)

| Pool | Owner | What is allowed to run there |
|---|---|---|
| `system` | AKS | **AKS-managed** components only (primarily `kube-system` non-DaemonSets) |
| `platform` | Platform/GitOps | Platform control-plane: `argocd` + Ameide operators + cluster controllers/operators + shared platform services |
| `apps` | Apps | Product workloads only |
| `workspaces-*`, `arc-runners`, `buildkit` | Platform | Exclusive pools (only intended workloads + required DaemonSets) |

Important nuance:

- **DaemonSets are expected to run on all pools** (CNI/CSI/node exporters/etc). “System=AKS-only” is mostly about **non-DaemonSet** controllers and services.

### Namespace ownership (what goes where)

| Namespace | Target pool |
|---|---|
| `kube-system` (non-DaemonSets) | `system` |
| `argocd` | `platform` |
| `ameide-system` (Ameide operators) | `platform` |
| Cluster operators (`cert-manager`, `external-secrets`, `cnpg-system`, `keda-system`, `redis-system`, `strimzi-system`, `clickhouse-system`, `devworkspace-controller`, `keycloak-system`, etc.) | `platform` |

## Non-disruption guardrails (hard requirements)

Any change that modifies a pod template (nodeSelector/tolerations/affinity/resources) can trigger a rollout.

Guardrails:

1) Prefer **replicas ≥2** and **PDBs** for any externally-facing / critical control-plane component before moving pools.
2) For singleton controllers (some operators), expect only transient reconciliation delay, not service downtime — but still stage changes.
3) Do not make “system” a safe harbor: never “fix scheduling” by moving platform-owned controllers back to `system`.
4) After each wave, verify:
   - no unexpected Pending pods,
   - DNS stability (CoreDNS not flapping),
   - ArgoCD health (no recurrent `lookup … no such host` errors).

## Break-glass (temporary; this session only)

Principle:

- Break-glass actions should be **minimal**, **reversible**, and **captured as follow-up Git/Terraform work**.

Allowed break-glass actions (only if needed to restore availability):

1) Fix missing node labels (`ameide.io/pool`) so reporting and scheduling constraints behave consistently.
2) Add/remove taints on a dedicated pool to stop accidental scheduling while Git changes propagate.

Follow-up requirement:

- Any break-glass change must be translated into Terraform (nodepool labels/taints) and Git (workload scheduling constraints) once the cluster is stable.

## Implementation plan (phased PR waves)

### Wave 1 — Platform owns `argocd` namespace (including Envoy proxies)

Goal: move all ArgoCD namespace workloads (Argo components + cluster gateway + Envoy proxy fleet) to `platform`.

Git changes:

- Ensure ArgoCD chart runs on `platform`:
  - `sources/values/common/argocd.yaml` (`global.nodeSelector.ameide.io/pool=platform`)
- Ensure Envoy proxy fleets created by gateways do not tolerate “system-only” constraints:
  - `sources/charts/cluster/gateway/templates/envoyproxy.yaml` (remove hardcoded `CriticalAddonsOnly` toleration)
  - `sources/values/_shared/platform/platform-gateway.yaml` (pin `infrastructure.nodeSelector.ameide.io/pool=platform`)

Acceptance:

- `argocd` namespace: all non-DaemonSet pods land on `platform`.
- Snapshot: `Platform namespaces not on platform pool` no longer lists `argocd`.

### Wave 2 — Platform owns Ameide operators + cluster operators

Goal: move platform-owned controllers/operators off `system` → `platform` (they are not AKS-managed).

Typical Git touchpoints:

- Ameide operators: `sources/values/_shared/cluster/ameide-operators.yaml`
- Cluster operators currently pinned to `system`:
  - `sources/values/_shared/cluster/cert-manager.yaml`
  - `sources/values/_shared/cluster/external-secrets.yaml`
  - `sources/values/_shared/cluster/cloudnative-pg.yaml`
  - `sources/values/_shared/cluster/keda.yaml`
  - `sources/values/_shared/cluster/redis-operator.yaml`
  - `sources/values/_shared/cluster/strimzi-operator.yaml`
  - `sources/values/_shared/cluster/clickhouse-operator.yaml`

Safety notes:

- Prefer moving one operator stack at a time (e.g., ExternalSecrets first, then cert-manager, etc.).
- For any operator with webhooks, ensure rollout does not drop all replicas at once.

Acceptance:

- Snapshot: system pool contains only `kube-system` non-DaemonSets (plus expected DaemonSets).
- Snapshot: platform namespaces list stays entirely on `platform`.

### Wave 3 — Stabilize DNS by reducing churn on AKS-managed components

Goal: stop GitOps-induced churn on CoreDNS and other AKS-managed kube-system add-ons.

Policy:

- Prefer AKS-managed add-ons to remain **provider-managed**; GitOps should not continuously reconcile their Deployments.
- If we need scheduling “nudges”, use non-disruptive mechanisms (ignoreDifferences / config overrides) and avoid repeated rollouts.

Acceptance:

- CoreDNS replicas stable (≥2), restarts not trending upward.
- ArgoCD logs no longer show repeating DNS lookup failures.

### Wave 4 — Right-size platform requests to fit 3 nodes (iterative)

Goal: make `platform` schedulable on **3 nodes** by reducing request-based over-allocation (not by adding nodes).

Method:

1) Generate snapshot:
   - `scripts/aks/export-pod-placement.sh --out backlog/685-aks-pod-placement-snapshot.md --context ameide`
2) Use the snapshot sections:
   - `Pool allocation (requests vs allocatable)`
   - `Platform right-sizing helpers (top requests)`
3) For each high-request component:
   - reduce requests in small steps,
   - verify no OOMKills / instability,
   - repeat until the 3-node target has headroom.

Acceptance:

- Platform pool `req cpu (%)` and `req mem (%)` show healthy headroom on 3 nodes (target to be defined; start with ≥20%).
- No recurring Pending pods due to `Insufficient cpu/memory`.

### Wave 5 — (Optional) Dedicated buildkit nodepool with min=0

Goal: keep buildkit off the shared platform pool and allow scaling to 0 when idle.

Feasibility (AKS):

- Yes, if:
  - buildkit pods are pinned to a dedicated nodepool label (e.g., `ameide.io/pool=buildkit`),
  - the nodepool autoscaler is enabled with `minCount=0`,
  - the nodepool is tainted `NoSchedule` and only buildkit tolerates it.

Risks to check:

- If buildkit uses persistent volumes, shrinking nodes can leave PV zone/attachment constraints that block rescheduling.
- “min=0” means builds must be able to tolerate cold start (node provisioning time).

Acceptance:

- buildkit pod(s) schedule only on the buildkit pool.
- buildkit pool stays at 0 nodes when idle and scales up when buildkit is Pending.

### Wave 6 — Decommission `ameidesys` safely (Terraform-aligned)

Goal: remove the legacy nodepool after everything is stable on the new pools.

Preconditions:

- No platform-owned workloads remain on `ameidesys`.
- `system` pool is healthy and runs the AKS-only workload set.

Steps:

- Drain/cordon `ameidesys` nodes (non-disruptive rollout policies in place).
- Remove nodepool via Terraform (preferred) once verified.

## Verification checklist (every wave)

1) Snapshot (read-only):
   - `scripts/aks/export-pod-placement.sh --context ameide`
2) Pending pods:
   - `kubectl get pods -A --field-selector=status.phase=Pending -o wide`
3) ArgoCD stability:
   - `kubectl -n argocd get pods -o wide`
   - `kubectl -n argocd logs deploy/argocd-application-controller --tail=200`
4) DNS stability:
   - `kubectl -n kube-system get deploy coredns -o wide`
   - `kubectl -n kube-system logs deploy/coredns --tail=200`

