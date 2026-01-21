# 454 – CoreDNS Config Cluster-Scoped Migration

**Status**: Implemented
**Created**: 2025-12-05
**Related**: [446-namespace-isolation.md](446-namespace-isolation.md), [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), [452-observability-namespace-isolation.md](452-observability-namespace-isolation.md)

---

## Update (2026-01-21): rewrites are cluster-scoped, but not the long-term managed-cluster DNS contract

This backlog is still correct about **ownership and scope**: CoreDNS customization in AKS is cluster-scoped and must be owned by a single ArgoCD application.

Platform direction has changed:

- In **managed clusters**, CoreDNS wildcard rewrites are **legacy/exception tooling**, not the “make DNS correct” strategy for canonical hostnames.
- The long-term platform DNS contract is **split-horizon DNS at the DNS layer** (Terraform-managed public + private DNS), keeping `*.{env}.ameide.io` canonical for both users and pods.

See: `backlog/716-platform-dns-split-horizon-envoy.md`.

## Why CoreDNS Config is Cluster-Scoped (Not Per-Environment)

### Vendor Documentation

AKS docs explicitly state:
- CoreDNS is a **managed cluster add-on** for cluster-wide DNS resolution
- Customizations are done via a **single `coredns-custom` ConfigMap in `kube-system`**
- You **cannot** have multiple CoreDNS instances or configs per namespace

Reference: [Microsoft Learn - Customize CoreDNS for AKS](https://learn.microsoft.com/en-us/azure/aks/coredns-custom)

### ArgoCD Best Practices

ArgoCD maintainers are clear:
- `SharedResourceWarning` appears when multiple apps manage the same resource - this is a **misconfiguration**
- Best practice: **a given Kubernetes resource must be owned by exactly one ArgoCD Application**

Reference: [ArgoCD Resource Tracking](https://argo-cd.readthedocs.io/en/latest/user-guide/resource_tracking/)

### Scope Clarification

The principle "cluster scoped only the operators; envs own their services/configmaps" is correct, but CoreDNS is an edge case:

| Scope | What It Contains |
|-------|------------------|
| **Cluster** | Operators, CRDs, and **cluster-level system configs** (CoreDNS, etc.) |
| **Environment** | Namespaces, services, deployments, secrets & configmaps **belonging to a specific environment** |

CoreDNS is a **cluster-level system component** (like other operators/add-ons). Its configuration (`coredns-custom` in `kube-system`) is therefore also cluster-level, even though it is technically a ConfigMap.

**Environment apps still own their Envoy services** (`envoy.ameide-dev`, `envoy.ameide-staging`, `envoy.ameide-prod`). The cluster app only manages the DNS rewrites that point to those services.

---

## Problem Statement

The `coredns-custom` ConfigMap in `kube-system` was being managed by **three separate ArgoCD applications** (one per environment):

- `dev-foundation-coredns-config`
- `staging-foundation-coredns-config`
- `production-foundation-coredns-config`

This caused:
1. **ArgoCD OutOfSync conflicts**: All three apps tried to manage the same ConfigMap
2. **Ownership races**: Whichever environment synced last "owned" the ConfigMap
3. **Incomplete DNS rewrites**: Only one environment's rewrites were active at a time

### Symptoms

```bash
kubectl get applications -n argocd | grep coredns
# dev-foundation-coredns-config         OutOfSync     Healthy
# staging-foundation-coredns-config     Synced        Healthy
# production-foundation-coredns-config  OutOfSync     Healthy
```

ArgoCD conditions showed:
```
SharedResourceWarning: ConfigMap/coredns-custom is part of applications
argocd/production-foundation-coredns-config and dev-foundation-coredns-config
```

---

## Design Principle

> **What belongs to shared goes to cluster, what belongs to environments goes to environments.**

The `coredns-custom` ConfigMap is a **cluster-scoped resource** (in `kube-system`) that affects DNS resolution for ALL environments. It belongs to cluster scope.

---

## Solution

Consolidate into a **single cluster-scoped application** that manages all environment rewrites:

### Component Location

```
environments/_shared/components/cluster/configs/coredns-config/component.yaml
```

### Values File

```yaml
# sources/values/cluster/coredns-config.yaml
manifests:
  - |
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: coredns-custom
      namespace: kube-system
    data:
      ameide.override: |
        # Development environment
        rewrite name regex (.+\.)?dev\.ameide\.io envoy.ameide-dev.svc.cluster.local
        rewrite name regex (.+\.)?tilt\.ameide\.io envoy.ameide-dev.svc.cluster.local
        # Staging environment
        rewrite name regex (.+\.)?staging\.ameide\.io envoy.ameide-staging.svc.cluster.local
        # Production environment (must be last)
        rewrite name regex (.+\.)?ameide\.io envoy.ameide-prod.svc.cluster.local
```

### Rollout Phase

Phase **030** (post-operator config) in the cluster ApplicationSet.

---

## Files Changed

| Action | Path |
|--------|------|
| Created | `environments/_shared/components/cluster/configs/coredns-config/component.yaml` |
| Created | `sources/values/cluster/coredns-config.yaml` |
| Deleted | `environments/_shared/components/foundation/coredns-config/` |

---

## Post-Sync Cleanup

After ArgoCD syncs, the old per-environment applications will be deleted automatically by the ApplicationSet controller. The orphaned tracking annotations on the ConfigMap will be replaced.

---

## Verification

### Check single cluster-scoped app

```bash
kubectl get applications -n argocd | grep coredns
# cluster-coredns-config    Synced    Healthy
```

### Check ConfigMap has all rewrites

```bash
kubectl get configmap coredns-custom -n kube-system -o yaml
# Should show all three environment rewrites
```

### Check DNS resolution works

```bash
# If CoreDNS rewrites are enabled, validate that the configured rewrites resolve.
# Note: in managed clusters the long-term direction is to remove reliance on rewrites
# for canonical `*.{env}.ameide.io` correctness; see backlog/716.
nslookup dev.ameide.io
nslookup staging.ameide.io
nslookup ameide.io
```

---

## Rewrite Order

The order of rewrites matters because CoreDNS processes them top-to-bottom:

1. `*.dev.ameide.io` → dev envoy (most specific subdomain first)
2. `*.tilt.ameide.io` → dev envoy (development tooling)
3. `*.staging.ameide.io` → staging envoy
4. `*.ameide.io` → production envoy (catch-all, must be last)

---

## Related Changes

This completes the namespace isolation effort:

- **445**: ArgoCD namespace isolation
- **446**: Environment namespace isolation (`ameide-dev`, `ameide-staging`, `ameide-prod`)
- **447**: Dual ApplicationSet model (cluster vs environment scope)
- **447**: Third-party chart tolerations
- **448**: Cert-manager workload identity
- **452**: Observability namespace isolation
- **454**: CoreDNS cluster-scoped migration (this document)
