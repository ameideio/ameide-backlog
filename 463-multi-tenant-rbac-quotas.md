# 463 – Multi-Tenant RBAC & Resource Quotas

> **Related documents:**
> - [441-networking.md](441-networking.md) – Networking improvements (NetworkPolicy isolation)
> - [443-tenancy-models.md](443-tenancy-models.md) – Tenancy model overview
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Namespace isolation patterns

## Problem Statement

Multi-tenancy currently relies on NetworkPolicy for traffic isolation, but lacks:

1. **RBAC**: No tenant-scoped roles defined; tenants could potentially modify platform labels
2. **ResourceQuota**: No resource limits per tenant namespace
3. **LimitRange**: No default resource requests/limits for tenant workloads

Per [Kubernetes Multi-tenancy guidance](https://kubernetes.io/docs/concepts/security/multi-tenancy/), complete tenant isolation requires:
- Namespaces + **NetworkPolicy** (traffic) ✅ Done in 441
- **RBAC** roles (control plane access) ❌ Not implemented
- **ResourceQuota** (resource consumption) ❌ Not implemented

## Requirements

### 1. Tenant RBAC Roles

Define namespace-scoped roles for tenant workloads:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-workload
  namespace: tenant-{{ .Values.tenantId }}-{{ .Values.environment }}
rules:
  # Workloads can read their own pods, services, configmaps
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
  # Cannot modify namespace labels
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get"]  # read-only
```

### 2. Label Protection

Prevent tenants from modifying platform labels (`ameide.io/*`):

**Option A**: ValidatingAdmissionPolicy (Kubernetes 1.30+)
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: protect-platform-labels
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        resources: ["namespaces"]
        operations: ["UPDATE"]
  validations:
    - expression: |
        !has(oldObject.metadata.labels) ||
        oldObject.metadata.labels.filter(k, k.startsWith('ameide.io/')).all(k,
          object.metadata.labels[k] == oldObject.metadata.labels[k])
      message: "Cannot modify ameide.io/* labels"
```

**Option B**: Kyverno ClusterPolicy
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: protect-platform-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: block-label-changes
      match:
        resources:
          kinds: ["Namespace"]
      validate:
        message: "Cannot modify ameide.io/* labels"
        deny:
          conditions:
            - key: "{{ request.object.metadata.labels | keys(@) | filter(@, contains(@, 'ameide.io')) }}"
              operator: NotEquals
              value: "{{ request.oldObject.metadata.labels | keys(@) | filter(@, contains(@, 'ameide.io')) }}"
```

### 3. ResourceQuota per Tenant

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-{{ .Values.tenantId }}-{{ .Values.environment }}
spec:
  hard:
    requests.cpu: "{{ .Values.quota.cpu | default "4" }}"
    requests.memory: "{{ .Values.quota.memory | default "8Gi" }}"
    limits.cpu: "{{ .Values.quota.cpuLimit | default "8" }}"
    limits.memory: "{{ .Values.quota.memoryLimit | default "16Gi" }}"
    pods: "{{ .Values.quota.pods | default "20" }}"
    services: "{{ .Values.quota.services | default "10" }}"
    persistentvolumeclaims: "{{ .Values.quota.pvcs | default "5" }}"
```

### 4. LimitRange Defaults

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limits
  namespace: tenant-{{ .Values.tenantId }}-{{ .Values.environment }}
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "4Gi"
```

## Implementation Plan

### Phase 1: Foundation (Quota + LimitRange)
- [ ] Add ResourceQuota template to tenant namespace chart
- [ ] Add LimitRange template to tenant namespace chart
- [ ] Define quota tiers (small/medium/large) in values
- [ ] Deploy to dev tenant namespaces first

### Phase 2: RBAC
- [ ] Define tenant-workload Role
- [ ] Define tenant-admin Role (for self-service)
- [ ] Create RoleBindings for tenant service accounts
- [ ] Test with actual tenant workloads

### Phase 3: Label Protection
- [ ] Choose admission controller (VAP vs Kyverno)
- [ ] Implement label protection policy
- [ ] Test that tenants cannot modify `ameide.io/*` labels
- [ ] Document escape hatch for platform team

## Tenant Quota Tiers

| Tier | CPU (req/limit) | Memory (req/limit) | Pods | PVCs |
|------|-----------------|---------------------|------|------|
| **small** | 2/4 | 4Gi/8Gi | 10 | 2 |
| **medium** | 4/8 | 8Gi/16Gi | 20 | 5 |
| **large** | 8/16 | 16Gi/32Gi | 50 | 10 |
| **enterprise** | custom | custom | custom | custom |

## References

- [Kubernetes Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [AKS Multi-tenant Best Practices](https://learn.microsoft.com/en-us/azure/aks/best-practices-multi-tenancy)
- [Kubernetes ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)
