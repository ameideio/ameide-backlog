---
title: "684 – AKS node pool strategy (Option B: system/platform/apps + workspaces)"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-16
suite: "aks"
source_of_truth: false
---

# 684 – AKS node pool strategy (Option B: system/platform/apps + workspaces)

## Purpose

Define an AKS node pool layout aligned to workload classes, plus dedicated execution pools for developer workspaces/tasks:

- `ameide-system`: shared cluster control-plane workloads
- `ameide-platform`: data plane (databases, brokers, observability state)
- `ameide-apps`: stateless microservices
- `ameide-workspaces-che`: Che DevWorkspaces (autoscale 0→N)
- `ameide-workspaces-coder`: Coder workspaces/tasks (autoscale 0→N)

This document is the **target-state specification** and the integration plan across:

- Terraform (AKS node pools)
- ArgoCD ApplicationSets (values layering + scheduling)
- Helm values (nodeSelector/tolerations + storage class selection)
- CI-only cloud mutation policy (Terraform apply/destroy runs in CI)

## Terminology note (Option B naming collision)

`backlog/442-environment-isolation.md` already uses **“Option B”** to mean **per-environment pools (system/dev/staging/prod)**.

This backlog defines a different **Option B**: **workload-class pools (system/platform/apps)**.

If we adopt this approach, we should treat `442` as historical context and update it (or add an explicit “Option C” there) so “Option B” is unambiguous.

## Current state (as of 2026-01-15)

- **Cluster inventory snapshot:** `backlog/682-aks-capacity-reservation-snapshot.md`
- **Terraform AKS reality (single effective pool):**
  - The default/system pool is pinned to **5 nodes** (`infra/terraform/azure/main.tf`).
  - That pool is labeled `ameide.io/pool=general` (`infra/terraform/modules/azure-aks/main.tf`).
  - ArgoCD schedules dev/staging/prod workloads using `nodeProfile: general-pool` (`config/clusters/azure.yaml` → `config/node-profiles/general-pool.yaml`).

Net effect: “system” nodes are acting as the shared general worker pool.

## Target state

### Node pools

| Pool | Purpose | VM SKU | Scaling | Disk intent |
|------|---------|--------|---------|------------|
| `system` | ArgoCD, controllers, CRD operators, cluster infra | `Standard_D4*` | fixed 2 | Premium LRS |
| `platform` | Stateful/data workloads + platform control-plane components (e.g. Che control-plane/vcluster) | `Standard_D8*` | fixed 3 | Premium LRS |
| `apps` | Stateless microservices | `Standard_B4ms` | fixed 3 | Standard |
| `workspaces-che` | Che DevWorkspaces execution plane | `Standard_D4*` | autoscale min=0 max=`N_che` | Premium SSD (workspace volumes) |
| `workspaces-coder` | Coder workspaces/tasks execution plane | `Standard_D4*` | autoscale min=0 max=`N_coder` | Premium SSD (workspace volumes) |

Sizing note:
- `N_che` and `N_coder` should be derived from peak concurrent workspace demand (interactive + task mode), not from baseline platform capacity.

### Scheduling contract (labels/taints)

All pools MUST set a stable label for selection:

- `ameide.io/pool: system|platform|apps|workspaces-che|workspaces-coder`

And MUST prevent accidental placement on non-app pools:

- `system`: taint `CriticalAddonsOnly=true:NoSchedule` (cluster infra must tolerate)
- `platform`: taint `ameide.io/pool=platform:NoSchedule` (platform/data workloads must tolerate)
- `workspaces-che`: taint `ameide.io/pool=workspaces-che:NoSchedule` (Che workspaces must tolerate)
- `workspaces-coder`: taint `ameide.io/pool=workspaces-coder:NoSchedule` (Coder workspaces must tolerate)
- `apps`: no taint (default landing zone)

Rationale: this keeps “regular workloads” out of `system` and `platform` by default, while keeping `apps` simple.

## Workload mapping

### system pool

Cluster-scoped infrastructure per `backlog/446-namespace-isolation.md`:

- ArgoCD (and ArgoCD-adjacent controllers)
- cert-manager (cluster install)
- external-secrets controller
- operator namespaces (`*-system`), CRD installers, admission webhooks
- cluster gateway control-plane components

### platform pool

Environment-scoped *platform/data plane* workloads:

- CNPG Postgres (`backlog/683-cnpg.md`)
- Kafka/Redis/ClickHouse and other stateful data components
- Observability state (Prometheus storage, Loki/Tempo backends where applicable)
- Platform control-plane components that are “always on” but not cluster-scoped (e.g. Che vcluster/control-plane from `backlog/680-che.md`)

### apps pool

Stateless microservices and horizontally scalable workloads, including “tenant workloads”.

### workspaces-che pool

Che workspace pods (DevWorkspace workloads). This is the execution plane; it is intentionally separate from the Che control-plane/vcluster components.

**Storage intent:** workspace PVCs should default to `managed-csi-premium` on AKS unless explicitly overridden for cost-sensitive workloads.

### workspaces-coder pool

Coder workspace/task pods. This is the execution plane; it is intentionally separate from any platform control-plane components.

**Storage intent:** workspace PVCs should default to `managed-csi-premium` on AKS unless explicitly overridden for cost-sensitive workloads.

## Required Git changes (design)

### 0) CI-only policy (cloud Terraform is CI-owned)

For shared cloud clusters, Terraform mutation is CI-only:

- **CI runs** `terraform apply`/`destroy` and publishes runtime facts.
- **Local machines** should run only `terraform validate` and review plans, and must not run `terraform apply` against cloud workspaces.

Cross-references:
- `AGENTS.md` (single writer: Git → CI Terraform → ArgoCD)
- `backlog/444-terraform-v2.md` (explicit “Only CI applies” contract)

### 1) Terraform: create pools + enforce taints/labels

Update the Terraform AKS module to:

- Stop treating the default/system pool as `ameide.io/pool=general`.
- Create user pools:
  - `platform` (D8 class)
  - `apps` (B4ms class)
  - `workspaces-che` (D4 class, autoscale 0→N)
  - `workspaces-coder` (D4 class, autoscale 0→N)
- Apply the label/taint contract above.

Related code locations:

- `infra/terraform/modules/azure-aks/*` (node pool resources)
- `infra/terraform/azure/*` (opinionated defaults for the “ameide” cluster)

**Disk intent:** confirm whether AKS/Terraform can set OS disk SKU per pool (Premium vs Standard). If not, treat “Premium” as the **PV storage class** contract for platform/data/workspace volumes (see next section).

### 2) Storage tiering: platform data uses premium PVs

GitOps should enforce premium storage where it matters:

- Use Azure CSI `managed-csi-premium` for IOPS-sensitive PVCs.
- Keep `managed-csi` (standard) for bulk/object/metrics where acceptable.

Cross-reference: `backlog/440-storage-concerns.md` (storage tiers + “data-tier node affinity” dependency).

### 3) ArgoCD scheduling: component-level placement (recommended)

Today, `nodeProfile` is chosen per **environment** (`config/clusters/azure.yaml`) and applied uniformly by `argocd/applicationsets/ameide.yaml`.

To implement *workload-class pools*, we need **component-level** placement. Recommended approach:

- Add an optional `nodeProfile` (or `pool`) field to each `environments/_shared/components/**/component.yaml`.
- Update `argocd/applicationsets/ameide.yaml` to prefer the component-level node profile when set, otherwise fall back to the environment’s default.

Why: per-chart “special-casing” is fragile and repeats the work already done in the node-profile system (and `backlog/447-third-party-chart-tolerations.md` shows how costly it is when placement is scattered across many value paths).

### 4) Explicit scheduling for `excludeGlobals: true` components

Components with `excludeGlobals: true` do not inherit the global/nodeProfile layering and must be scheduled explicitly via their own values.

Che-related components are in this category:

- `environments/_shared/components/platform/control-plane/che*/component.yaml`
- `environments/_shared/components/platform/control-plane/vcluster-che/component.yaml`

Cross-reference: `backlog/680-che.md`.

Workspace schedulers (Che/Coder) must also be explicit about node placement, since workspace pods are not always controlled by our standard chart patterns:

- Che DevWorkspace templates must set nodeSelector/tolerations for `ameide.io/pool=workspaces-che`.
- Coder workspace templates must set nodeSelector/tolerations for `ameide.io/pool=workspaces-coder`.

## Migration plan (high level)

1. **Add new pools** in Terraform (do not remove existing scheduling yet).
2. **Introduce component-level placement** in ArgoCD ApplicationSet generation.
3. **Move platform/data workloads first** (platform pool taint ensures only intentional workloads land there).
4. **Constrain system pool** (CriticalAddonsOnly tolerations + optional nodeSelectors for cluster controllers).
5. **Move Che/Coder workspace execution** onto their dedicated pools (autoscale 0→N).
6. **Finalize apps pool** as the default landing zone for stateless workloads.
7. Capture a new post-change snapshot (update `backlog/682-aks-capacity-reservation-snapshot.md` or add a new snapshot).

## Risks / gotchas

- **Terminology drift**: `442` “Option B” ≠ this doc’s “Option B”; resolve before implementation.
- **Scheduling surface area**: third-party charts and `excludeGlobals` components need explicit placement (`backlog/447-third-party-chart-tolerations.md`).
- **Storage class drift**: enforce premium PVs for platform data; do not confuse AKS-managed storage classes with local fallbacks (`backlog/440-storage-concerns.md`).
- **Environment shutdown**: this 3-pool design does **not** provide “scale dev/staging node pools to 0” savings like the per-environment pool design in `442`.
- **Workspace autoscale**: workspace pools can scale to 0; ensure no critical platform pods depend on them.

## Acceptance criteria

- AKS has pools (`system`, `platform`, `apps`, `workspaces-che`, `workspaces-coder`) with stable labels.
- `system` and `platform` pools are protected by taints; no “regular workloads” land there accidentally.
- Workspace pools are protected by taints; only their respective workspace pods schedule there.
- CNPG/Kafka/Redis/ClickHouse are placed on `platform` nodes and use appropriate PV storage classes.
- Che control-plane/vcluster components have explicit placement and remain reachable (`backlog/680-che.md`).
- ArgoCD Applications remain renderable without ad-hoc per-chart placement hacks.
- Cloud Terraform mutation follows CI-only policy (`backlog/444-terraform-v2.md`); no manual cloud `terraform apply`.

## Cross-references (existing docs to keep aligned)

- `backlog/682-aks-capacity-reservation-snapshot.md` – current cluster inventory snapshot
- `backlog/442-environment-isolation.md` – per-environment pool strategy (historical alternative; naming collision)
- `backlog/444-terraform-v2.md` – CI-only cloud Terraform apply/destroy policy
- `backlog/446-namespace-isolation.md` – cluster-scoped vs environment-scoped split (what should live on `system`)
- `backlog/447-third-party-chart-tolerations.md` – tolerations/nodeSelector surface area and pitfalls
- `backlog/441-networking.md` – tier labels + NetworkPolicy assumptions (workload placement impacts policy)
- `backlog/440-storage-concerns.md` – storage tiers + premium PV guidance (platform/data)
- `backlog/683-cnpg.md` – CNPG operational constraints (Azure Disk RWO, update strategy)
- `backlog/680-che.md` – Che control-plane/vcluster constraints (shared component, routing model)
- `backlog/465-applicationset-architecture.md` – ApplicationSet design constraints (where to encode placement)
