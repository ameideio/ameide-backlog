---
title: "696 – AKS StorageClass ownership + guardrails"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-19
suite: "aks"
source_of_truth: false
---

# 696 – AKS StorageClass ownership + guardrails

## Problem

AKS creates and reconciles built-in `StorageClass` resources (for example `managed-csi`, `managed-csi-premium`).

If GitOps applies a `StorageClass` with the same name but a different provisioner (for example local-path),
AKS can overwrite it and/or ArgoCD will continuously drift. Worse, workloads can end up using an unexpected provisioner.

## Contract (target state)

- **AKS clusters**
  - GitOps MUST NOT create/modify AKS-provided `StorageClass` objects.
  - GitOps MAY create additional uniquely named StorageClasses (only when required).
- **Local clusters (k3d/k3s)**
  - GitOps MAY create local-path StorageClasses to support local development.

## Implementation (current repo)

- `foundation-managed-storage` values are environment-scoped:
  - AKS envs manage only namespaces (no StorageClass objects).
  - Local env defines the local-path StorageClasses used by local deployments.
- CI guardrail fails if `foundation-managed-storage` renders any `StorageClass` for AKS:
  - `scripts/ci/check-aks-storageclass-collision.sh`
  - wired into `.github/workflows/helm-test.yaml`

## Follow-ups (recommended)

- Rename local StorageClasses away from AKS-reserved names (optional but safer long-term).
- Add a render-time guardrail for any other chart that could introduce `StorageClass` objects on AKS.

