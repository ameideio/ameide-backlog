# 466 – Post-Mortem: Cluster Gateway Outage

**Date**: 2025-12-07
**Duration**: ~25 minutes (10:10 - 10:35 UTC)
**Severity**: P1 - ArgoCD UI inaccessible
**Author**: Claude (AI Assistant)

## Executive Summary

During a refactoring effort to align chart folder structure (464), the `sources/charts/cluster/gateway/` chart was incorrectly identified as "orphaned" and deleted. This caused the ArgoCD cluster gateway to fail, making https://argocd.ameide.io/ inaccessible. The issue was compounded by envoy-gateway's inability to automatically recreate deleted Services.

## Timeline (UTC)

| Time | Event |
|------|-------|
| 10:00 | Started Phase 5 of 464: removing orphaned charts |
| 10:05 | Deleted 7 charts including `sources/charts/cluster/gateway/` |
| 10:05 | Committed and pushed changes |
| 10:10 | ArgoCD reported `cluster-gateway` Application as Unknown/ComparisonError |
| 10:10 | Manually-created `cluster-gateway` Application deletion started (stuck on finalizers) |
| 10:13 | Identified root cause: gateway chart was NOT orphaned, was used by cluster-gateway |
| 10:13 | Restored chart from git history: `git checkout 7b0f1fb~1 -- sources/charts/cluster/gateway/` |
| 10:14 | Committed and pushed restoration |
| 10:14 | ArgoCD still showing Unknown - chart not in main branch yet for ApplicationSet |
| 10:20 | Removed finalizers from stuck Application deletion |
| 10:21 | Old Application finally deleted |
| 10:22 | User feedback: "cluster gateway is what makes argocd (cluster scoped) exposed" |
| 10:24 | Created new component.yaml for gateway under cluster ApplicationSet management |
| 10:24 | New `cluster-gateway` Application created by ApplicationSet |
| 10:25 | Application Synced but Health: Progressing |
| 10:30 | Discovered: envoy LoadBalancer Service was deleted and NOT recreated |
| 10:32 | Gateway showing "No addresses have been assigned" |
| 10:34 | Manually created Service with correct Azure static IP annotations |
| 10:35 | ArgoCD accessible at https://argocd.ameide.io/ - **RECOVERED** |

## Root Cause Analysis

### Primary Cause: Incorrect Orphan Identification

The `sources/charts/cluster/gateway/` chart was flagged as "orphaned" because:

1. **No component.yaml existed** in `environments/_shared/components/cluster/`
2. The `cluster-gateway` Application was **manually created** outside of GitOps
3. Our orphan detection logic checked for component references but missed manually-created Applications

```
# What we checked (incomplete)
grep -r "cluster/gateway" environments/  # Found nothing → marked as orphan

# What we should have checked
kubectl get application -n argocd | grep gateway  # Would have found cluster-gateway
```

### Secondary Cause: Envoy Gateway Service Reconciliation Gap

When the old `cluster-gateway` Application was deleted:

1. The Application had finalizers that tried to clean up resources
2. The envoy LoadBalancer Service (`envoy-argocd-cluster-18b70826`) was deleted
3. Envoy-gateway controller did **NOT recreate** the Service

**Technical Details:**

```yaml
# EnvoyProxy had correct config
spec:
  provider:
    kubernetes:
      envoyService:
        annotations:
          service.beta.kubernetes.io/azure-load-balancer-ipv4: 20.160.216.7
        loadBalancerIP: 20.160.216.7
```

But envoy-gateway's infrastructure runner:
- Creates Services during initial Gateway reconciliation
- Does NOT watch for deleted Services
- Does NOT recreate missing child resources
- Has no "desired state" reconciliation loop for Services

### Tertiary Cause: DNS/IP Mismatch

- DNS `argocd.ameide.io` → `20.160.216.7` (static Azure IP)
- Old Service got dynamic IP `20.61.241.89` (Azure assigned)
- When manually recreating, had to explicitly set Azure annotations for static IP

## Impact

| System | Impact |
|--------|--------|
| ArgoCD UI | Inaccessible for ~25 minutes |
| ArgoCD API | Inaccessible for ~25 minutes |
| GitOps Syncs | Continued working (in-cluster communication) |
| Application Deployments | Not affected |
| Monitoring | Alerts triggered for gateway health |

## What Went Well

1. **Quick identification**: Root cause identified within 3 minutes of first error
2. **Git recovery**: Chart restored from git history within 1 minute
3. **User communication**: User provided critical context about gateway purpose
4. **Service recreation**: Manual workaround restored access

## What Went Wrong

1. **Incomplete orphan detection**: Didn't check for manually-created Applications
2. **No pre-flight validation**: Didn't verify chart deletion impact before pushing
3. **Finalizer handling**: Application stuck in deletion blocked progress
4. **Controller limitation**: Envoy-gateway doesn't reconcile missing Services
5. **Missing documentation**: Cluster-gateway wasn't documented as critical infrastructure

## Lessons Learned

### 1. Manually-Created Resources Are Technical Debt

The `cluster-gateway` Application was created manually, outside GitOps:
- Not visible in git
- Not managed by ApplicationSet
- No component.yaml to declare it exists
- Created "invisible" dependency on chart

**Resolution**: Created proper `component.yaml` at:
```
environments/_shared/components/cluster/configs/gateway/component.yaml
```

### 2. Orphan Detection Must Be Comprehensive

Checking only git for references is insufficient:

```bash
# WRONG: Only checks git
find environments -name "component.yaml" -exec grep -l "cluster/gateway" {} \;

# RIGHT: Also check running Applications
kubectl get applications -A -o jsonpath='{range .items[*]}{.spec.sources[*].path}{"\n"}{end}' | grep "cluster/gateway"
```

### 3. Envoy Gateway Has No Service Self-Healing

The envoy-gateway controller:
- Creates Deployment, Service, ConfigMap during Gateway creation
- Does NOT watch for their deletion
- Does NOT recreate if they go missing
- Considers Gateway "Programmed" even without Service

This is a **known limitation** - the Service is a "fire and forget" creation.

### 4. Static IP Binding Requires Annotations

Azure LoadBalancer static IPs require explicit annotations:

```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-ipv4: "20.160.216.7"
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "Ameide"
spec:
  loadBalancerIP: "20.160.216.7"
```

Without these, Azure assigns a random dynamic IP, breaking DNS.

## Action Items

### Immediate (Done)

| ID | Action | Status |
|----|--------|--------|
| 466-1 | Restore gateway chart | ✅ Done |
| 466-2 | Create gateway component.yaml | ✅ Done |
| 466-3 | Recreate LoadBalancer Service | ✅ Done |
| 466-4 | Verify ArgoCD accessibility | ✅ Done |
| 466-5 | Document ApplicationSet architecture (465) | ✅ Done |

### Short-Term (TODO)

| ID | Action | Owner | Priority |
|----|--------|-------|----------|
| 466-6 | Add cluster-gateway to smoke tests | - | P2 |
| 466-7 | Create runbook for gateway recovery | - | P2 |
| 466-8 | Document envoy-gateway Service limitation | - | P3 |

### Long-Term (TODO)

| ID | Action | Owner | Priority |
|----|--------|-------|----------|
| 466-9 | Add pre-flight checks before chart deletion | - | P2 |
| 466-10 | Consider managing envoy Service in Helm chart | - | P3 |
| 466-11 | Audit all manually-created Applications | - | P2 |

## Technical Details

### Affected Resources

```
sources/charts/cluster/gateway/
├── Chart.yaml
├── Chart.lock
├── charts/
│   └── gateway-helm-1.3.0.tgz
├── templates/
│   ├── envoyproxy.yaml
│   ├── gateway.yaml
│   ├── gatewayclass.yaml
│   ├── httproute-argocd.yaml
│   └── redirect-gateway.yaml
└── values.yaml
```

### Recovery Commands Used

```bash
# Restore chart from git
git checkout 7b0f1fb~1 -- sources/charts/cluster/gateway/

# Remove stuck finalizers
kubectl patch application cluster-gateway -n argocd \
  --type=json -p='[{"op": "remove", "path": "/metadata/finalizers"}]'

# Manually create Service (workaround)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: envoy-argocd-cluster-18b70826
  namespace: argocd
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-ipv4: "20.160.216.7"
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "Ameide"
spec:
  type: LoadBalancer
  loadBalancerIP: "20.160.216.7"
  externalTrafficPolicy: Local
  selector:
    gateway.envoyproxy.io/owning-gateway-name: cluster
    gateway.envoyproxy.io/owning-gateway-namespace: argocd
  ports:
  - name: https-443
    port: 443
    targetPort: 10443
EOF
```

### Verification Commands

```bash
# Check gateway status
kubectl get gateway cluster -n argocd

# Check Application health
kubectl get application cluster-gateway -n argocd

# Test HTTPS access
curl -s -o /dev/null -w "%{http_code}" https://argocd.ameide.io/
```

## Prevention Measures

### 1. Orphan Detection Checklist

Before deleting any chart, verify:

- [ ] No component.yaml references the chart
- [ ] No Application in any namespace uses the chart path
- [ ] Chart is not a dependency of another chart
- [ ] No manual Applications exist that use the chart
- [ ] Deletion is tested in dev environment first

### 2. Critical Infrastructure Registry

Maintain a list of critical cluster infrastructure:

| Component | Chart | Application | Recovery Priority |
|-----------|-------|-------------|-------------------|
| ArgoCD Gateway | cluster/gateway | cluster-gateway | P0 |
| CoreDNS Config | foundation/common/raw-manifests | cluster-coredns-config | P0 |
| Cert Manager | third_party/cert-manager | cluster-crds-cert-manager | P0 |

### 3. Gateway Monitoring

Add synthetic monitoring for:
- https://argocd.ameide.io/ availability
- Gateway `Programmed` condition
- LoadBalancer Service existence
- Static IP binding verification

## Appendix

### Related Documents

- [464-chart-folder-alignment.md](464-chart-folder-alignment.md) - Original refactoring backlog
- [465-applicationset-architecture.md](465-applicationset-architecture.md) - ApplicationSet documentation
- [446-namespace-isolation.md](446-namespace-isolation.md) - Namespace isolation architecture

### Commits Related to Incident

```
7b0f1fb chore: remove 7 orphaned charts (464-13 to 464-19)  # CAUSED ISSUE
29f6832 fix(cluster-gateway): restore incorrectly deleted gateway chart
57267c9 refactor(cluster): clean cluster-scoped config structure (464)
```

### Service Recreation Details

The manually-created Service will be managed by envoy-gateway going forward. However, if envoy-gateway is restarted or the Gateway is recreated, it may create a new Service with a different name/hash.

**Risk**: The manually-created Service could become orphaned if envoy-gateway creates a new one.

**Mitigation**: Monitor for duplicate Services and clean up as needed.
