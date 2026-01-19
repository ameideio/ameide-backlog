---
title: "700 – Rename local StorageClasses away from AKS defaults"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-19
suite: "storage"
source_of_truth: false
---

# 700 – Rename local StorageClasses away from AKS defaults

## Problem

Local clusters currently rely on local-path `StorageClass` objects named like AKS defaults (for example `managed-csi`).

Even with CI guardrails, keeping AKS-reserved names in Git increases long-term risk:
- future chart/components could accidentally re-introduce collisions
- review burden stays high (“is this local-only?”)
- drift incidents are harder to reason about under pressure

## Target state

- Local clusters define clearly local names:
  - `local-path` (default)
  - optionally `local-path-premium` (if we need a second tier locally)
- AKS clusters keep AKS-owned names (`managed-csi`, `managed-csi-premium`) and GitOps never defines them.

## Implementation plan

1. Add new local StorageClasses (`local-path`, optionally `local-path-premium`) using the local-path provisioner.
2. Update local values to avoid hard-coding `managed-csi*`:
   - Prefer `.Values.storageClass` (already `local-path` in `sources/values/cluster/local/globals.yaml`).
   - Replace any remaining `storageClassName: managed-csi` in local overlays with `{{ .Values.storageClass }}` or env overrides.
3. Remove the legacy local-path StorageClasses named `managed-csi*` from local GitOps.

## Acceptance criteria

- Rendering for `clusterType=local` produces `StorageClass/local-path` (and not `StorageClass/managed-csi`).
- Rendering for `clusterType=azure` produces no `StorageClass` objects.
- Local k3d boots without requiring AKS-reserved StorageClass names.

