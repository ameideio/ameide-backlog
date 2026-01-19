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

- **AKS**
  - GitOps does not create any `StorageClass` objects.
  - Cluster-scoped namespaces required by storage/platform subsystems are managed once per cluster via ArgoCD.
  - The per-environment `foundation-managed-storage` component is allowed to be empty on AKS (`allowEmpty: true`).
- **Local**
  - Local env defines the local-path StorageClasses used by local deployments.
- CI guardrail fails if `foundation-managed-storage` renders any `StorageClass` for AKS:
  - `scripts/ci/check-aks-storageclass-collision.sh`
  - wired into `.github/workflows/helm-test.yaml`

## Detection signal (how we know “this is AKS”)

The repo uses the cluster type as an explicit signal in both rendering and GitOps:

- ArgoCD uses `config/clusters/azure.yaml` (`clusterType: azure`) vs `config/clusters/local.yaml` (`clusterType: local`)
- Value layering includes `sources/values/cluster/{clusterType}/globals.yaml`
- Charts should treat `global.ameide.cluster.type` as the authoritative indicator (`azure` vs `local`)

## Remediation playbook (if a collision ever happens)

1. Remove GitOps ownership of the colliding `StorageClass` (delete/stop applying the manifest in Git).
2. Let AKS reconcile its built-in classes back to the Azure CSI provisioners.
3. Verify: `kubectl get storageclass managed-csi -o jsonpath='{.provisioner}'` returns `disk.csi.azure.com`.
4. Confirm the default class annotation is correct (`storageclass.kubernetes.io/is-default-class`).
5. Re-run CI and confirm ArgoCD is `Synced` with no StorageClass drift.

## Follow-ups (recommended)

**Required**
- Rename local StorageClasses away from AKS-reserved names (see `700`).

**Recommended**
- Add a render-time guardrail for any other chart that could introduce `StorageClass` objects on AKS (end state: “no StorageClass kind renders when clusterType=azure”).

## Notes (ArgoCD drift prevention)

Cluster-scoped resources MUST NOT be managed by multiple per-environment applications.

If a resource must exist only once (for example a shared namespace), manage it via a cluster-scoped component/app to avoid:
- `SharedResourceWarning`
- persistent `OutOfSync` due to ownership conflicts
