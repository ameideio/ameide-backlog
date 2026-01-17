---
title: "686 – AKS pool principles + migration plan (system/platform/apps)"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-17
suite: "aks"
source_of_truth: false
---

# 686 – AKS pool principles + migration plan (system/platform/apps)

## Context / goals

We are standardizing node pool usage to:

- reduce DNS/control-plane instability caused by churn and contention,
- keep **AKS-managed** components isolated,
- move **platform control-plane + platform operators** (including ArgoCD namespace) onto the **platform** pool,
- right-size **resource requests** so platform fits **3 nodes** (vs “needs 5” due to request over-allocation),
- decommission the legacy `ameidesys` nodepool safely,
- use **break-glass** only as a temporary bridge, then align Terraform/Bicep and Git so drift does not reappear.

Non-goals:

- Re-architecting application workloads.
- Large refactors of charts unless required for safety.

## Principles (authoritative)

### AKS terminology (system vs user pools)

AKS models node pools by **mode**:

- **System-mode** node pools: intended for critical system add-ons (for example CoreDNS, metrics-server).
- **User-mode** node pools: intended for all other workloads (including “platform” and “apps” in our internal model).

In this document:

- `system` corresponds to an AKS **system-mode** pool.
- `platform`, `apps`, `workspaces-*`, `arc-runners` correspond to AKS **user-mode** pools (even if “platform” feels control-plane-ish internally).

### Pool definitions

| Pool | Purpose | Scheduling rule of thumb |
|---|---|---|
| `system` | **AKS-managed / kube-system add-ons only** | Only AKS/kube-system related workloads land here |
| `platform` | **Platform control plane + operators** | ArgoCD namespace + Ameide operators + infra controllers |
| `apps` | Product workloads | Business/tenant workloads only |
| `workspaces-*`, `arc-runners` | Exclusive pools | Only their intended workloads; tolerate taint explicitly |

Note: `workspaces-*` and `arc-runners` are **conceptual pool groups** expressed via our `ameide.io/pool` label; they are not necessarily literal AKS agent pool names (AKS agent pool names have strict lowercase/length rules).

### Namespace ownership (desired steady state)

| Namespace | Owner | Target pool |
|---|---|---|
| `kube-system` | AKS | `system` |
| `argocd` | Platform | `platform` |
| `ameide-system` (Ameide operators) | Platform | `platform` |
| Cluster operators (`cert-manager`, `external-secrets`, `cnpg-system`, `keda-system`, `strimzi-system`, `redis-system`, etc.) | Platform | `platform` |
| `ameide-dev`, `ameide-staging`, `ameide-prod` | Apps + Platform shared services | apps workloads on `apps`, shared infra on `platform` |

If a namespace is “platform-owned” but currently has workloads pinned to `system`, that is a **policy violation** to be eliminated (by moving those workloads to `platform`, not by reclassifying them as system).

### Scheduling mechanics

Hard rules:

- `system` is **exclusive by definition** and SHOULD be tainted `CriticalAddonsOnly=true:NoSchedule` (AKS guidance) to keep non-system workloads off system nodes by default.
- Other **exclusive pools** (e.g., `workspaces-*`, `arc-runners`) SHOULD be tainted `NoSchedule` so only explicitly-tolerating workloads land there.
- `system` must not become a “dumping ground” for unpinned controllers (taint is the backstop; explicit placement is still required).
- Use **explicit nodeSelector or required node affinity** for pool placement on all cluster-level controllers.

Guidelines:

- Prefer nodeSelector `ameide.io/pool=<pool>` when supported by the chart.
- When a component can’t express nodeSelector (some operators/CRDs), use required nodeAffinity to the same label.
- Prefer configuring labels/taints at the **node pool** level (Terraform/Bicep/AKS API) so new nodes inherit them; per-node labeling is break-glass only, and in shared cloud should be done via the dedicated CI break-glass workflows (not manual `kubectl label/taint`).
- Avoid constantly mutating AKS-managed kube-system Deployments via GitOps (CoreDNS, etc.). Customize using supported AKS mechanisms only.
- Consider admission enforcement (Azure Policy / Gatekeeper / Kyverno) so “platform-owned namespaces must declare pool placement” does not regress over time.

### Resource requests (fit platform into 3 nodes)

Principle: Kubernetes schedules on **requests**, not on real-time usage.

Target: `platform` should remain schedulable with **3 nodes** at typical load **with upgrade headroom** (surge nodes + PDB/eviction constraints), by:

1) measuring requested CPU/memory totals (per pool),
2) identifying top request consumers,
3) lowering requests where safe, and/or shifting non-platform workloads off platform,
4) keeping HA: do not reduce requests in a way that forces frequent OOMKills or tight CPU throttling.

We must treat changes that touch pod templates (requests/limits/nodeSelectors) as **rollout-causing** changes and stage them carefully.

## Implementation plan (step-by-step, non-disruptive)

### Phase 0 — Safety prerequisites (before moving anything)

- Confirm Argo sync strategy is safe for controllers (rollouts, maxUnavailable=0 where required).
- Confirm each critical controller has either:
  - ≥2 replicas, or
  - an acceptable maintenance window / graceful failover story.
- Confirm CoreDNS is stable (replicas ≥2) and not being rolled by GitOps churn.
- Confirm node pool upgrade settings and headroom expectations (e.g., `maxSurge` per pool) so “fits 3 nodes” does not deadlock upgrades due to PDB constraints.
- Confirm the intended `system` pool isolation mechanism (taint) and toleration expectations for critical add-ons.

### Phase 1 — Break-glass stabilization (today; temporary)

Goal: stop fires and get consistent labeling/placement without waiting for Terraform.

Actions (documented, reversible):

- Ensure node pool/node labeling is consistent (fix any `<none>`), preferring a Git/IaC change; if stabilization can’t wait, use the dedicated CI break-glass workflows (avoid manual `kubectl label` in shared cloud).
- Apply/verify `system` pool taint `CriticalAddonsOnly=true:NoSchedule` and confirm critical add-ons tolerate it; encode this in IaC in Phase 4 (avoid manual `kubectl taint` in shared cloud).
- If a component must be moved immediately to restore availability, use a minimal Git change + fast Argo sync (preferred) or a CI break-glass workflow; avoid direct `kubectl patch/apply` in shared cloud.

Exit criteria:

- DNS flapping stops (no repeated lookup errors from Argo/external-secrets/vault).
- No critical pods Pending due to pool misplacement.

### Phase 2 — Git alignment (make Git the source of truth again)

Goal: encode the target pool policy in Git so break-glass changes are no longer needed.

Work items (in small PRs / waves):

1) `argocd` namespace components → `platform` pool
   - gateway/Envoy proxy + envoy-gateway controller
   - nifikop
   - preview-janitor
   - ArgoCD control plane (installer/terraform path)
2) Ameide operators (`ameide-system`) → `platform` pool
3) Cluster operators → `platform` pool
4) Confirm `kube-system` non-DaemonSets → `system` pool

Verification:

- Re-run the placement snapshot and ensure:
  - system namespaces are system-only (`kube-system`),
  - platform namespaces are platform-only (`argocd`, `ameide-system`, etc.),
  - no `requested_pool != actual_pool`.

### Phase 3 — Right-size requests to fit platform=3 nodes (iterative)

Goal: eliminate request-based over-allocation that forces platform pool scale-up.

Method:

1) Export per-pool allocatable vs requested CPU/memory (snapshot script).
2) Identify top N workloads by requested CPU/mem on `platform`.
3) For each candidate:
   - reduce requests by a small step,
   - verify stability (restarts, latency, error rate),
   - repeat until the platform pool fits 3 nodes with headroom for upgrades (surge + PDB constraints).

Guardrails:

- Never “solve scheduling” by moving platform-owned controllers back to system.
- Prefer reducing requests for stateless deployments first; treat StatefulSets carefully.
- Keep PDBs / rollout strategies so updates are non-disruptive.

### Phase 4 — Align Terraform/Bicep (after Git proves stable)

Goal: prevent reintroducing drift on future cluster rebuilds.

- Update infra definitions so:
  - new nodepools are created with correct labels,
  - system pool tainting (`CriticalAddonsOnly=true:NoSchedule`) is encoded at the pool level (not as a manual per-node patch),
  - ArgoCD installation sets nodeSelectors/tolerations consistent with “argocd=platform”,
  - `system` pool definition matches “AKS-related only”.

### Phase 5 — Decommission legacy `ameidesys`

Preconditions:

- New `system` nodepool is in the correct AKS mode to replace `ameidesys`.
- New `system` nodepool has sufficient nodes for reliability (prod baseline target: ≥3 nodes).
- New `system` nodepool is tainted `CriticalAddonsOnly=true:NoSchedule`.
- No critical workloads remain on `ameidesys` nodes.
- DaemonSets required by AKS run on the new system pool.

Steps:

- Drain/cordon `ameidesys` nodes, verify no rescheduling regressions.
- Remove `ameidesys` nodepool via Terraform (preferred) once stable.

## Verification checklist (every wave)

- `kubectl get pods -A --field-selector=status.phase=Pending`
- Verify no non-system workloads land on the `system` pool (taint + placement expectations hold).
- ArgoCD: `kubectl -n argocd get pods -o wide` + repo-server/application-controller logs for DNS errors
- CoreDNS: replicas, restarts, and rollout history (avoid GitOps churn)
- Snapshot: regenerate `backlog/685-aks-pod-placement-snapshot.md`
- Capacity: platform requested vs allocatable CPU/mem shows headroom on 3 nodes

## Vendor references (for reviewers)

- System pools and `CriticalAddonsOnly` guidance: https://learn.microsoft.com/en-us/azure/aks/use-system-pools
- Scheduler best practices (taints/tolerations, affinity): https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-advanced-scheduler
- Pool/node labels: https://learn.microsoft.com/en-us/azure/aks/use-labels
- CoreDNS customization: https://learn.microsoft.com/en-us/azure/aks/coredns-custom
- Reliability, upgrades, and surge guidance: https://learn.microsoft.com/en-us/azure/aks/best-practices-app-cluster-reliability
