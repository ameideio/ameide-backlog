> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 316: Security Hardening & Namespace Segmentation

**Status**: Planning
**Priority**: High
**Category**: Infrastructure, Security
**Created**: 2025-10-29
**Last Updated**: 2025-10-29

## Overview

Comprehensive security review of the AMEIDE platform's Kubernetes topology, identifying critical security gaps and providing a phased migration plan to achieve defense-in-depth architecture with namespace segmentation, network isolation, and Pod Security Standards compliance.

## Current Security Posture: MODERATE

### Key Statistics
- **Namespaces**: 14 (single application namespace `ameide` + operators)
- **Services in ameide**: 78 total, 3 LoadBalancer-exposed
- **Pods**: 70+, **59 pods** not following security best practices
- **NetworkPolicies**: Only 9 (12% coverage, primarily for operators)
- **Secrets**: 409 total (37 ExternalSecrets from Azure KeyVault)
- **No ResourceQuotas or LimitRanges** configured
- **No Pod Security Standards** labels on namespace

## Current Topology

### Single Namespace Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ SINGLE NAMESPACE DEPLOYMENT (ameide)                       │
├─────────────────────────────────────────────────────────────┤
│ ✅ Infrastructure Operators (separate namespaces):          │
│   - cnpg-system (PostgreSQL operator)                      │
│   - strimzi-system (Kafka operator)                        │
│   - cert-manager                                            │
│   - external-secrets                                        │
│   - redis-operator                                          │
│                                                             │
│ ⚠️  All Application Services (ameide namespace):            │
│   - Application: threads, platform, workflows, agents, etc.    │
│   - Data Stores: postgres, kafka, redis                    │
│   - Observability: prometheus, grafana, loki, tempo        │
│   - Auth: keycloak                                          │
│   - Gateway: envoy-gateway                                  │
│   - Analytics: plausible                                    │
│   - LLM Ops: langfuse, inference                            │
└─────────────────────────────────────────────────────────────┘
```

**Risk**: Single namespace blast radius - compromise of any pod grants network access to all services.

### External Attack Surface

**LoadBalancer Services (Public IPs)**:
```
172.20.0.4:80/443/8000  → envoy-ameide-ameide (Gateway - ALL traffic)
172.20.0.2:9000/4002    → minio (Object storage)
172.20.0.2:4001         → temporal-web (Workflow UI)
```

**Gateway Routes** (26 total):
- **7 GRPCRoutes** (internal port 8000): agents, agents_runtime, threads, platform, workflows, graph, inference
- **19 HTTPRoutes** (HTTPS port 443): UIs, APIs, observability dashboards
- **TLS Termination**: Gateway handles all TLS (wildcard cert `*.dev.ameide.io`)

### Network Isolation

**NetworkPolicies Coverage**: **12% (9 out of ~70 workloads)**

**Protected Workloads**:
- ✅ Kafka (Strimzi-managed ingress rules)
- ✅ Keycloak (port 8080, 7800 for clustering)
- ✅ OAuth2-Proxy instances (4 policies for observability UIs)
- ✅ Langfuse S3

**Unprotected Workloads** (88%):
- ❌ Application services: threads, platform, workflows, agents, inference
- ❌ Databases: PostgreSQL clusters
- ❌ Redis failover
- ❌ Envoy Gateway (data plane)
- ❌ MinIO, Temporal

## Security Findings

### 🔴 CRITICAL Severity

#### C-1: No Default-Deny NetworkPolicy
- **Finding**: Namespace lacks a default-deny NetworkPolicy, allowing all pod-to-pod communication
- **Impact**: Compromised application pod can access databases, Keycloak, Kafka, Redis directly
- **CVSS**: 8.6 (Network lateral movement, data exfiltration)
- **Mitigation**: Phase 1 - Deploy default-deny policies

#### C-2: No Application Namespace Segmentation
- **Finding**: All application services, data stores, and infrastructure in single namespace
- **Impact**: Single namespace-scoped credential compromise grants access to all resources
- **Blast Radius**: Entire platform
- **Mitigation**: Phase 2 - Multi-namespace architecture

#### C-3: Pods Running as Root
- **Finding**: 59 pods not enforcing `runAsNonRoot`, `readOnlyRootFilesystem`, or dropping capabilities
- **Impact**: Container escape → Node compromise
- **Evidence**: Node exporter pods run with `hostPID=true` and `hostNetwork=true`
- **Mitigation**: Phase 1 - Pod Security Standards enforcement

### 🟠 HIGH Severity

#### H-1: No Resource Quotas or LimitRanges
- **Finding**: No namespace-level resource constraints
- **Impact**: Noisy neighbor attacks, resource exhaustion, denial of service
- **Evidence**: 42 pods without CPU/memory limits
- **Mitigation**: Phase 1 - ResourceQuotas and LimitRanges

#### H-2: Overly Permissive RBAC
- **Finding**: 96 RoleBindings/ClusterRoleBindings, default ServiceAccount used by many pods
- **Impact**: Pods can potentially access Kubernetes API with unnecessary permissions
- **Evidence**: 28 custom ServiceAccounts, but many infrastructure pods use elevated roles
- **Mitigation**: Phase 1 - RBAC audit and least-privilege refactoring

#### H-3: No Pod Security Standards Enforcement
- **Finding**: Namespace lacks `pod-security.kubernetes.io/*` labels
- **Impact**: No admission control preventing privileged pods
- **Current**: Prometheus node-exporter runs with `hostPID` and `hostNetwork`
- **Mitigation**: Phase 1 - Apply PSS labels (restricted mode)

#### H-4: Single Gateway for All Traffic
- **Finding**: One Gateway resource handles public web, gRPC APIs, admin UIs, observability
- **Impact**: Gateway compromise = full platform access; no traffic segregation
- **Mitigation**: Phase 3 - Separate gateways for public/api/admin

### 🟡 MEDIUM Severity

#### M-1: Secrets Stored in Etcd
- **Finding**: While ExternalSecrets pulls from Azure KeyVault, secrets are still stored unencrypted in etcd
- **Mitigation**: Enable etcd encryption at rest (kube-apiserver configuration)

#### M-2: No Service Mesh for mTLS
- **Finding**: Service-to-service communication is plaintext HTTP/gRPC
- **Impact**: Network sniffing can capture internal API traffic
- **Note**: gRPC uses h2c (HTTP/2 cleartext) on port 8000
- **Mitigation**: Phase 4 - Evaluate Linkerd or Istio for mTLS

#### M-3: LoadBalancer Services Without Source IP Restrictions
- **Finding**: MinIO (9000), Temporal Web (4001) exposed without IP allowlisting
- **Impact**: Unnecessary external exposure of administrative interfaces
- **Mitigation**: Phase 3 - Gateway IP allowlisting

#### M-4: No Audit Logging for Kubernetes API
- **Finding**: No audit policy configured for API server events
- **Impact**: Limited incident response capability
- **Mitigation**: Phase 4 - Enable audit logging, ship to Loki

## Target Security Architecture

### Multi-Namespace Topology

```
┌──────────────────────────────────────────────────────────────────┐
│  PROPOSED: DEFENSE-IN-DEPTH NAMESPACE ARCHITECTURE                │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ameide-app (Application Services)                               │
│    ├─ threads, platform, workflows, graph, agents, etc.        │
│    ├─ NetworkPolicy: Default-deny + explicit allow rules        │
│    ├─ Pod Security: Restricted standard                          │
│    └─ ResourceQuota: CPU/Memory limits                           │
│                                                                   │
│  ameide-data (Data Stores)                                       │
│    ├─ postgres-*, kafka-*, redis-*                              │
│    ├─ NetworkPolicy: Allow only from ameide-app                  │
│    ├─ Pod Security: Baseline standard                            │
│    └─ No internet egress                                         │
│                                                                   │
│  ameide-ai (AI/ML Services)                                      │
│    ├─ inference, agents_runtime, langfuse                        │
│    ├─ NetworkPolicy: Allow from ameide-app + external LLM APIs  │
│    └─ ResourceQuota: GPU/CPU limits                              │
│                                                                   │
│  ameide-auth (Identity & Access)                                 │
│    ├─ keycloak, oauth2-proxy                                     │
│    ├─ NetworkPolicy: Ingress from Gateway only                   │
│    └─ Pod Security: Restricted standard                          │
│                                                                   │
│  ameide-gateway (Ingress Layer)                                  │
│    ├─ envoy-gateway, envoy data planes                           │
│    ├─ NetworkPolicy: Ingress from internet, egress to backends  │
│    └─ Separate Gateways: public vs admin vs gRPC                │
│                                                                   │
│  ameide-observability (Monitoring & Logs)                        │
│    ├─ prometheus, grafana, loki, tempo                           │
│    ├─ NetworkPolicy: Scrape from all namespaces (via SA)        │
│    └─ No access to application secrets                           │
│                                                                   │
│  ameide-system (Platform Services)                               │
│    ├─ minio, temporal, plausible                                 │
│    ├─ NetworkPolicy: Selective access from app namespaces       │
│    └─ ResourceQuota: Prevent runaway workflows                   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Gateway Segmentation

```
┌─────────────────────────────────────────────────────────┐
│  3 Separate Gateways (instead of 1)                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  gateway-public  (*.ameide.io:443)                      │
│    ├─ www-ameide, www-ameide-platform                   │
│    ├─ WAF-enabled (rate limiting, bot detection)       │
│    └─ IP allowlist: Cloudflare CDN IPs                  │
│                                                          │
│  gateway-api  (api.ameide.io:443)                       │
│    ├─ gRPC services: threads, platform, graph        │
│    ├─ OAuth2 authentication required                    │
│    └─ Rate limiting per client                          │
│                                                          │
│  gateway-admin  (admin.ameide.io:443)                   │
│    ├─ Observability UIs, Temporal, pgAdmin             │
│    ├─ IP allowlist: Corporate VPN, bastion hosts       │
│    └─ mTLS client certificates required                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

## Migration Plan

### Phase 1: Foundational Security (2-3 weeks)

**Objectives**: Harden existing single-namespace deployment before migration

#### Week 1: Pod Security Hardening
- [ ] **Day 1-2**: Audit all pod security contexts, create tracking spreadsheet
- [ ] **Day 3-4**: Update Helm charts with secure defaults:
  - `runAsNonRoot: true`
  - `readOnlyRootFilesystem: true`
  - `allowPrivilegeEscalation: false`
  - `capabilities.drop: ["ALL"]`
  - `seccompProfile.type: RuntimeDefault`
- [ ] **Day 5**: Apply Pod Security Standards labels to `ameide` namespace (mode: `warn` only)
- [ ] **Day 6-7**: Fix pods violating PSS, deploy updated charts to staging

**Example Pod Security Context**:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10000
  fsGroup: 10000
  seccompProfile:
    type: RuntimeDefault
containers:
- name: app
  securityContext:
    capabilities:
      drop: ["ALL"]
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
```

#### Week 2: Network Policies
- [ ] **Day 1-2**: Design NetworkPolicy rules for all services (ingress/egress allow lists)
- [ ] **Day 3**: Deploy default-deny NetworkPolicy (audit mode, not enforcing)
- [ ] **Day 4-5**: Analyze logs, adjust policies, deploy allow rules for legitimate traffic
- [ ] **Day 6-7**: Enable enforcement in staging, monitor for 3 days

**Example Default-Deny Policy**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: ameide
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-threads-to-platform
  namespace: ameide
spec:
  podSelector:
    matchLabels:
      app: platform
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: threads
    ports:
    - protocol: TCP
      port: 8082
```

#### Week 3: Resource Limits
- [ ] **Day 1-2**: Analyze current resource usage (Prometheus metrics), set realistic limits
- [ ] **Day 3**: Create ResourceQuota and LimitRange manifests
- [ ] **Day 4-5**: Deploy to staging, load test to validate quotas
- [ ] **Day 6-7**: Production rollout

**Example ResourceQuota**:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: ameide
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    limits.memory: 200Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "0"  # Prevent accidental exposure
```

**Deliverables**:
- ✅ 100% of pods compliant with Pod Security Standards (Baseline minimum)
- ✅ NetworkPolicies covering 100% of workloads
- ✅ ResourceQuotas preventing resource exhaustion

### Phase 2: Namespace Segmentation (3-4 weeks)

**Objectives**: Migrate from single namespace to multi-namespace architecture

#### Week 1: Planning & Helm Chart Refactoring
- [ ] **Day 1-2**: Finalize namespace architecture, create migration runbook
- [ ] **Day 3-5**: Update Helmfile to support multi-namespace deployments
- [ ] **Day 6-7**: Update application Helm charts to support dynamic namespace configuration

#### Week 2: Data Layer Migration
- [ ] **Day 1**: Create `ameide-data` namespace, apply NetworkPolicies
- [ ] **Day 2-3**: Migrate PostgreSQL clusters (pg_dump/restore, test connections)
- [ ] **Day 4-5**: Migrate Kafka (topic replication, consumer group migration)
- [ ] **Day 6-7**: Migrate Redis, validate cache behavior

#### Week 3: Application Layer Migration
- [ ] **Day 1**: Create `ameide-app`, `ameide-ai`, `ameide-auth` namespaces
- [ ] **Day 2-3**: Deploy services to new namespaces, update DNS/service references
- [ ] **Day 4-5**: Smoke test all APIs, run E2E tests
- [ ] **Day 6-7**: Cutover traffic, monitor for 48 hours

#### Week 4: Gateway & Observability Migration
- [ ] **Day 1-2**: Create `ameide-gateway` namespace, deploy separate Gateway resources
- [ ] **Day 3-4**: Migrate observability stack to `ameide-observability`
- [ ] **Day 5-7**: Update Prometheus ServiceMonitors for cross-namespace scraping

**Example Cross-Namespace NetworkPolicy**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-app-namespace
  namespace: ameide-data
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ameide-app
      podSelector:
        matchLabels:
          app: platform
    ports:
    - protocol: TCP
      port: 5432
```

**Deliverables**:
- ✅ 7 namespaces with clear separation of concerns
- ✅ Cross-namespace NetworkPolicies enforced
- ✅ Zero-downtime migration validated

### Phase 3: Gateway Security (2 weeks)

**Objectives**: Segment gateway traffic and harden external access

#### Week 1: Gateway Segmentation
- [ ] Create 3 Gateway resources: `gateway-public`, `gateway-api`, `gateway-admin`
- [ ] Update HTTPRoutes/GRPCRoutes to reference appropriate Gateways
- [ ] Configure IP allowlisting for `gateway-admin`

**Example Gateway Separation**:
```yaml
# Public Gateway (for end users)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-public
  namespace: ameide-gateway
spec:
  gatewayClassName: envoy
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: "*.ameide.io"
    tls:
      mode: Terminate
      certificateRefs:
      - name: ameide-wildcard-tls
---
# Admin Gateway (IP restricted)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-admin
  namespace: ameide-gateway
spec:
  gatewayClassName: envoy
  listeners:
  - name: https
    port: 8443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: admin
```

#### Week 2: Rate Limiting & WAF
- [ ] Configure Envoy Gateway rate limiting (BackendTrafficPolicy)
- [ ] Integrate Cloudflare or AWS WAF (or use Envoy ext_authz)
- [ ] Enable DDoS protection on LoadBalancer IPs

**Example Rate Limiting Policy**:
```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: rate-limit-api
  namespace: ameide-gateway
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: gateway-api
  rateLimit:
    type: Global
    global:
      rules:
      - clientSelectors:
        - headers:
          - name: x-user-id
            type: Distinct
        limit:
          requests: 1000
          unit: Hour
```

**Deliverables**:
- ✅ 3 separate Gateways with traffic segregation
- ✅ IP allowlisting for admin interfaces
- ✅ Rate limiting protecting APIs

### Phase 4: Advanced Security (Ongoing)

**Objectives**: Continuous security improvements

#### Service Mesh Evaluation (Optional)
- [ ] **Month 1**: POC Linkerd in dev environment, measure performance impact
- [ ] **Month 2**: Rollout mTLS to staging, validate cert rotation
- [ ] **Month 3**: Production rollout, enable authorization policies

#### Continuous Improvements
- [ ] Enable Kubernetes Audit Logging (ship to Loki)
- [ ] Implement vulnerability scanning (Trivy, Grype) in CI/CD
- [ ] Configure automated secret rotation (Azure KeyVault + ExternalSecrets)
- [ ] Deploy runtime security monitoring (Falco)
- [ ] Implement image signing and verification (Sigstore)
- [ ] Configure egress filtering (block all except allowed external APIs)

## Risk Mitigation Summary

| Risk | Current | Target | Mitigation |
|------|---------|--------|------------|
| **Blast Radius** | Single namespace = full access | 7 namespaces with isolation | Namespace segmentation + NetworkPolicies |
| **Lateral Movement** | 88% pods unprotected | 100% coverage | Default-deny + explicit allow rules |
| **Container Escape** | 59 pods not hardened | 0 privileged pods | Pod Security Standards (Restricted) |
| **Resource Exhaustion** | No limits | ResourceQuotas enforced | Namespace quotas + LimitRanges |
| **External Exposure** | 3 LoadBalancer IPs, no IP filtering | IP allowlists, WAF | Gateway segmentation, Cloudflare |
| **RBAC Over-Privilege** | 96 bindings, many elevated | Least-privilege SAs | RBAC audit + refactoring |
| **Secrets in Etcd** | Unencrypted | Encrypted at rest | etcd encryption config |
| **Service-to-Service** | Plaintext HTTP/gRPC | mTLS encrypted | Service mesh (optional) |

## Compliance Alignment

The proposed architecture addresses:

- **SOC 2 Type II**: Network segmentation (CC6.6), Access controls (CC6.2), Logging (CC7.2)
- **ISO 27001**: A.13.1 (Network security), A.9.4 (Access restriction)
- **NIST CSF**: PR.AC-5 (Network segmentation), PR.PT-3 (Least functionality)
- **CIS Kubernetes Benchmark**:
  - 5.2 (Pod Security Standards)
  - 5.3 (Network Policies)
  - 5.7 (Secrets encryption)

## Implementation Files

### New Charts/Templates to Create
- `infra/kubernetes/charts/platform/namespace-security/` (ResourceQuotas, LimitRanges, PSS)
- `infra/kubernetes/charts/platform/network-policies/` (Default-deny, app-specific policies)
- `gitops/ameide-gitops/sources/charts/apps/gateway/` (Gateway chart; configure public/admin variants via values)

### Files to Update
- `infra/kubernetes/helmfile.yaml` (Add namespace-specific releases)
- `infra/kubernetes/charts/platform/*/templates/deployment.yaml` (Add securityContext)
- `infra/kubernetes/environments/*/platform/*.yaml` (Namespace-specific values)
- `Tiltfile` (Support multi-namespace development)

## Success Metrics

### Security Metrics
- **NetworkPolicy Coverage**: 0% → 100%
- **Pod Security Compliance**: 0% → 100% (Baseline), 80% (Restricted)
- **Resource Limit Coverage**: 42 pods without limits → 0 pods
- **Namespace Blast Radius**: 1 namespace → 7 namespaces (86% reduction)
- **Mean Time to Detect (MTTD)**: N/A → <5 minutes (with audit logging)

### Performance Metrics
- **Service Latency Impact**: <5% increase (from NetworkPolicies)
- **Resource Utilization**: CPU/Memory within quota limits (no OOMKills)
- **Zero-Downtime Migration**: 100% success rate

## References

- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [NetworkPolicy Best Practices](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [NIST SP 800-190: Application Container Security Guide](https://csrc.nist.gov/publications/detail/sp/800-190/final)
- [Envoy Gateway Rate Limiting](https://gateway.envoyproxy.io/latest/user/rate-limit/)
- [Linkerd Service Mesh](https://linkerd.io/2.14/features/automatic-mtls/)

## Notes

- **Performance Impact**: NetworkPolicies add ~2-5% latency in testing; service mesh mTLS adds ~5-10%
- **Development Experience**: Multi-namespace requires updating Tilt configurations for local development
- **Backward Compatibility**: Phase 1 can be done in-place; Phase 2 requires downtime for data migration
- **Cost**: No additional infrastructure costs; potential savings from ResourceQuotas preventing overprovisioning
