---
title: "701 – Terraform adopt reality: AKS drift realignment (no-op plan)"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-19
suite: "terraform"
source_of_truth: false
---

# 701 – Terraform adopt reality: AKS drift realignment (no-op plan)

## Problem

Terraform configuration drifted from the live AKS cluster after break-glass actions (manual pool changes, upgrades, etc.).
As a result, Terraform plan/apply is not “day-2 boring” and may propose disruptive rotations/replacements (or fail with
“already exists” conflicts when resources were created outside Terraform).

This backlog item defines the “adopt reality” path: align IaC to the live cluster until `terraform plan` is **no-op**.

Constraints:
- Cloud mutation must follow the GitOps single-writer path (Git → CI Terraform → ArgoCD). No manual cloud `terraform apply`.
- Read-only inspection is OK (`az ... show/list`, `kubectl get/describe/logs`).

## Evidence (current reality)

- AKS pool inventory and settings are captured in `backlog/685-aks-pod-placement-snapshot.md`.
- Current Terraform definitions live in:
  - `infra/terraform/modules/azure-aks/main.tf`
  - `infra/terraform/azure/variables.tf`
  - `.github/workflows/terraform-azure-apply.yaml` (imports/adoption)

## Contract (target state)

1) CI `Terraform Azure Plan` is **no-op** against the shared cluster.
2) Node pool definitions in Terraform match the live cluster for all pools Terraform manages:
   - **names**, **vm sizes**, **autoscaling settings**, **maxPods**, **labels/taints**.
3) The AKS upgrade model is explicit and Terraform does not churn on version drift:
   - Either Terraform does not set `kubernetes_version` (preferred for “rapid”/“patch” channels), or it pins intentionally.
4) CI “apply” does not fail due to pre-existing resources (import/adopt is deterministic).

## Work items

### 1) Align Terraform to live nodepools (adopt reality)

- Treat **live nodepool names** as canonical (do not “rename” via Terraform).
- Align immutable pool properties (vm size, maxPods) to reality to avoid forced rotations.
- If we want a different pool shape later, create a **new** pool and migrate workloads deliberately (separate backlog/wave).

### 2) Make the upgrade model explicit

Pick one model and encode it:

- **Model A (“boring day-2”):** auto-upgrade = `patch`, Terraform does not fight patch drift.
- **Model B (“rapid”):** auto-upgrade = `rapid`, Terraform must not pin `kubernetes_version` (or must ignore it) to avoid churn.

### 3) CI adoption/import completeness

Ensure `.github/workflows/terraform-azure-apply.yaml` imports all Terraform-managed nodepools which may be created in break-glass:
- apps/applications pool
- platform pool
- arc-runners pool
- workspaces pools (che/coder)

## Acceptance criteria

- `Terraform Azure Plan` artifact shows `Plan: 0 to add, 0 to change, 0 to destroy` after adoption.
- `Terraform Azure Apply + Verify` is safe to rerun and remains converged (no forced rotations without an explicit change).
- “Normal apply” does not perform imports/adoption; break-glass recovery uses the explicit adopt workflow.
