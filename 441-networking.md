# 441 – Networking Improvements

> **Related documents:**
> - [443-tenancy-models.md](443-tenancy-models.md) – **Tenancy model overview (single source of truth)**
> - [440-storage-concerns.md](440-storage-concerns.md) – Storage improvements
> - [442-environment-isolation.md](442-environment-isolation.md) – Environment isolation
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) – Environment naming conventions
> - [240-cluster-rightsizing.md](240-cluster-rightsizing.md) – Cluster resource planning
> - [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) – Dual ApplicationSet architecture
> - [436-envoy-gateway-observability.md](436-envoy-gateway-observability.md) – EnvoyProxy telemetry configuration
> - [450-envoy-gateway-per-environment-architecture.md](450-envoy-gateway-per-environment-architecture.md) – Per-environment gateway architecture
> - [457-azure-lb-redirect-gateway-consolidation.md](457-azure-lb-redirect-gateway-consolidation.md) – HTTP redirect consolidation
> - [459-httproute-ownership.md](459-httproute-ownership.md) – HTTPRoute ownership ("apps own routes" pattern)

## Implementation Status (2025-12-04)

| Component | Status | Notes |
|-----------|--------|-------|
| **Namespace labels** | ✅ Done | `ameide.io/environment` + tenant labels added to `foundation-namespaces.yaml` |
| **Tier labels (values)** | ✅ Done | `tier` field added to all app/data/platform values |
| **Tier labels (pods)** | ✅ Done | `tierLabels` helper added to all 12 app charts |
| **Cross-env NetworkPolicy** | ✅ Done | `deny-cross-environment` policy in `foundation-namespaces.yaml` |
| **nodeSelector/tolerations** | ✅ Done | Added to all app deployment templates |
| **Tier-based NetworkPolicy** | ✅ Done | `tier-based-ingress` policy in `foundation-namespaces.yaml` |
| **Cross-tenant NetworkPolicy** | ✅ Done | `deny-cross-tenant` policy in `foundation-namespaces.yaml` (conditional) |
| **Rate limiting template** | ✅ Done | `rate-limit-policy.yaml` with auth & global policies |
| **Rate limiting values** | ✅ Done | Values added to `_shared/platform/platform-gateway.yaml` |

## Problem Statement

Current networking configuration lacks several production-ready features:

1. **No cross-environment isolation**: Traffic can flow between dev/staging/prod namespaces
2. **No tier-based traffic flow**: No enforcement of frontend→backend→data flow
3. **No rate limiting**: Gateway accepts unlimited requests
4. **Missing namespace labels**: Required for NetworkPolicy selectors
5. **Component-only policies**: Existing policies (minio, keycloak, kafka) don't enforce architecture

## Current State (Verified 2025-12-04)

### Gateway Architecture

- **Gateway Controller**: Envoy Gateway v1.6.0 (runs in `argocd` namespace)
- **API Version**: Kubernetes Gateway API
- **TLS**: cert-manager with wildcard certificates per environment
- **Load Balancer**: Azure Standard LB with static public IPs per environment
- **HTTP Redirect**: Consolidated into main gateway HTTP listener (see [457](457-azure-lb-redirect-gateway-consolidation.md))

### EnvoyProxy Per-Environment Architecture

Each environment has its own EnvoyProxy resource with a dedicated static IP from Terraform:

| Environment | EnvoyProxy | Namespace | Static IP | Azure Resource |
|-------------|------------|-----------|-----------|----------------|
| dev | ameide-proxy-config | ameide-dev | 40.68.113.216 | ameide-dev-envoy-pip |
| staging | ameide-proxy-config | ameide-staging | 108.142.228.7 | ameide-staging-envoy-pip |
| production | ameide-proxy-config | ameide-prod | 4.180.130.190 | ameide-prod-envoy-pip |
| argocd (cluster) | cluster-proxy-config | argocd | 20.160.216.7 | ameide-argocd-pip |

**DNS Records**: Explicit subdomain A records (www, platform, api, grafana, etc.) are Terraform-managed alongside wildcard (`*`) and apex (`@`) records. This ensures DNS always points to the correct Envoy IP per environment. See [444-terraform.md](444-terraform.md) TF-22.

Configuration files:
- Per-env static IPs: `sources/values/{dev,staging,production}/platform/platform-gateway.yaml`
- Cluster gateway: `sources/charts/cluster/gateway/values.yaml`
- EnvoyProxy template: `sources/charts/apps/gateway/templates/envoyproxy-telemetry.yaml`

### Tenant Routing Patterns

Domain routing differs by tenancy model (see [443-tenancy-models.md](443-tenancy-models.md)):

| Tenancy Model | Domain Pattern | HTTPRoute Target Namespace |
|---------------|----------------|----------------------------|
| Shared SaaS | `*.ameide.io` | `ameide-prod` |
| Namespace-per-tenant | `*.<tenant>.ameide.io` | `tenant-<tenant>-prod` |
| Namespace-per-tenant | Custom domain (CNAME) | `tenant-<tenant>-prod` |
| Private cloud | `*.ameide-<tenant>.io` | `ameide-prod` (in tenant cluster) |
| Private cloud | Custom domain | `ameide-prod` (in tenant cluster) |

**HTTPRoute example for namespace-per-tenant:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: tenant-acme-platform
  namespace: tenant-acme-prod
spec:
  parentRefs:
    - name: ameide
      namespace: envoy-gateway-system
  hostnames:
    - "platform.acme.ameide.io"
    - "api.acme.ameide.io"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: platform
          namespace: tenant-acme-prod
          port: 8080
```

All tenancy models reuse the same Gateway + HTTPRoute patterns; only hostnames and target namespaces change.

> **HTTPRoute ownership**: Per [459-httproute-ownership.md](459-httproute-ownership.md), apps own their HTTPRoutes (rendered from their Helm charts), while the gateway chart owns infrastructure-level routes (apex redirects, CORS policies). This follows Gateway API best practices for the "app developer" vs "cluster operator" personas.

### Existing Gateway Policies

**File**: `sources/charts/apps/gateway/templates/`

| Policy Type | Purpose | Status |
|-------------|---------|--------|
| BackendTrafficPolicy | Timeouts, retries | ✅ Configured |
| ClientTrafficPolicy | X-Forwarded-For | ✅ Configured |
| SecurityPolicy | CORS | ✅ Configured |
| EnvoyPatchPolicy | TLS settings | ✅ Configured |
| Rate Limiting | Request throttling | ✅ Configured (`rate-limit-policy.yaml`) |

### Existing Network Policies

```bash
kubectl get networkpolicies -A
```

| Namespace | Policy | Scope | Gap |
|-----------|--------|-------|-----|
| ameide-dev | `data-minio` | MinIO pods | Component-only |
| ameide-dev | `keycloak-network-policy` | Keycloak pods | Component-only |
| ameide-staging | `kafka-network-policy-kafka` | Kafka pods | Component-only |
| argocd | `argocd-*-network-policy` | ArgoCD components | ✅ Good |

**Implemented policies (2025-12-04):**
- ✅ Cross-environment isolation (`deny-cross-environment` in `foundation-namespaces.yaml`)
- ✅ Tier-based flow enforcement (`tier-based-ingress` in `foundation-namespaces.yaml`)
- ⏳ Default deny ingress per namespace (not yet implemented - consider carefully as it's restrictive)

### Namespace Labels (Implemented 2025-12-04)

Labels configured in `sources/values/_shared/foundation/foundation-namespaces.yaml`:

| Namespace | `ameide.io/environment` | Notes |
|-----------|------------------------|-------|
| ameide-dev | `dev` | Via `{{ .Values.environment }}` |
| ameide-staging | `staging` | Via `{{ .Values.environment }}` |
| ameide-prod | `production` | Via `{{ .Values.environment }}` |
| argocd | `cluster` | Explicit label |
| cert-manager | `cluster` | Explicit label |
| cnpg-system | `cluster` | Explicit label |
| strimzi-system | `cluster` | Explicit label |

> **Note**: Pod tier labels (`ameide.io/tier`) are applied via `tierLabels` helper in deployment templates,
> not namespace labels. Tier values come from `tier:` field in app values files.

### Tenant Labels (Required for Multi-Tenancy)

Namespaces and pods MUST also carry tenant labels for cross-tenant isolation (see [443-tenancy-models.md](443-tenancy-models.md)):

| Label | Values | Description |
|-------|--------|-------------|
| `ameide.io/tenant-kind` | `shared`, `namespace`, `private` | Tenancy model |
| `ameide.io/tenant-id` | `<tenant-slug>` | Tenant identifier (omit for shared) |

**Examples:**

Shared SaaS namespace:
```yaml
# ameide-prod
metadata:
  labels:
    ameide.io/environment: production
    ameide.io/tenant-kind: shared
    # No tenant-id for shared
```

Namespace-per-tenant:
```yaml
# tenant-acme-prod
metadata:
  labels:
    ameide.io/environment: production
    ameide.io/tenant-kind: namespace
    ameide.io/tenant-id: acme
```

### Key Files

- `sources/charts/apps/gateway/templates/backendtrafficpolicy.yaml`
- `sources/charts/apps/gateway/templates/clienttrafficpolicy.yaml`
- `sources/charts/apps/gateway/templates/securitypolicy-cors.yaml`
- `sources/charts/apps/gateway/templates/envoypatch-*.yaml`
- `sources/values/*/platform/platform-gateway.yaml`

---

## Proposed Changes

### 0. Add Namespace Labels (Prerequisite)

**Priority**: HIGH (Blocker for all NetworkPolicies)

Add labels to namespaces via the namespace Helm chart:

**File**: `sources/charts/foundation/namespaces/templates/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace }}
  labels:
    ameide.io/environment: {{ .Values.environment }}
    ameide.io/managed-by: gitops
    kubernetes.io/metadata.name: {{ .Values.namespace }}
```

**Required labels per namespace:**

| Namespace | `ameide.io/environment` | Notes |
|-----------|------------------------|-------|
| ameide-dev | `dev` | Development workloads |
| ameide-staging | `staging` | Staging workloads |
| ameide-prod | `production` | Production workloads |
| argocd | `cluster` | Cluster-scoped, allowed everywhere |

---

### 1. Cross-Environment Isolation

**Priority**: HIGH

Block all traffic between environment namespaces:

**File**: `sources/charts/foundation/common/templates/networkpolicy-cross-env-deny.yaml`

```yaml
{{- range $env := list "dev" "staging" "production" }}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-env-ingress
  namespace: ameide-{{ $env }}
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Allow from same environment namespace
    - from:
        - namespaceSelector:
            matchLabels:
              ameide.io/environment: {{ $env }}
    # Allow from cluster-scoped namespaces (argocd, cert-manager)
    - from:
        - namespaceSelector:
            matchLabels:
              ameide.io/environment: cluster
    # Allow from kube-system (metrics, DNS)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
{{- end }}
```

**Traffic matrix after policy:**

| Source → Dest | ameide-dev | ameide-staging | ameide-prod |
|---------------|-----------|----------------|-------------|
| ameide-dev | ✅ | ❌ | ❌ |
| ameide-staging | ❌ | ✅ | ❌ |
| ameide-prod | ❌ | ❌ | ✅ |
| argocd | ✅ | ✅ | ✅ |
| kube-system | ✅ | ✅ | ✅ |

---

### 1.5 Cross-Tenant Isolation (Namespace-per-tenant) ✅

**Status**: Implemented in `foundation-namespaces.yaml` with conditional rendering

For namespace-per-tenant workloads (`tenant-*` namespaces), block traffic between different tenants:

**File**: `sources/values/_shared/foundation/foundation-namespaces.yaml` (conditional manifest)

```yaml
{{- if eq .Values.tenantKind "namespace" }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-tenant-ingress
  namespace: {{ .Values.namespace }}  # e.g. tenant-acme-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Same tenant only
    - from:
        - namespaceSelector:
            matchLabels:
              ameide.io/tenant-id: {{ .Values.tenantId }}
    # Cluster services (argocd, cert-manager, kube-system)
    - from:
        - namespaceSelector:
            matchLabels:
              ameide.io/environment: cluster
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
{{- end }}
```

> **Shared SaaS note**: Namespaces with `tenant-kind=shared` (e.g., `ameide-prod`) use
> application-level multi-tenancy instead of namespace isolation. This policy only
> applies to `tenant-*` namespaces.

**Cross-tenant traffic matrix:**

| Source → Dest | tenant-acme-prod | tenant-contoso-prod | ameide-prod |
|---------------|------------------|---------------------|-------------|
| tenant-acme-prod | ✅ | ❌ | ❌ |
| tenant-contoso-prod | ❌ | ✅ | ❌ |
| ameide-prod | ❌ | ❌ | ✅ |
| argocd | ✅ | ✅ | ✅ |
| kube-system | ✅ | ✅ | ✅ |

---

### 2. Tier-Based Traffic Flow

**Priority**: MEDIUM

Enforce the architectural flow: `external → frontend → backend → data`

**Service tier classification** (from `component.yaml` domains):

| Tier | Services | Exposure |
|------|----------|----------|
| **frontend** | www-ameide, www-ameide-platform, plausible | External (HTTPRoute) |
| **backend** | graph, inference, platform, agents, workflows, threads | Mixed |
| **data** | postgres, redis, kafka, temporal, minio, clickhouse | Internal only |
| **platform** | keycloak, grafana, prometheus, envoy-gateway | Mixed |

**File**: `sources/charts/foundation/common/templates/networkpolicy-tier-flow.yaml`

```yaml
# Data tier: only accepts traffic from backend tier
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: data-tier-ingress
  namespace: {{ .Values.namespace }}
spec:
  podSelector:
    matchLabels:
      ameide.io/tier: data
  policyTypes:
    - Ingress
  ingress:
    # Allow from backend tier
    - from:
        - podSelector:
            matchLabels:
              ameide.io/tier: backend
    # Allow from platform tier (observability)
    - from:
        - podSelector:
            matchLabels:
              ameide.io/tier: platform
    # Allow from same tier (replication)
    - from:
        - podSelector:
            matchLabels:
              ameide.io/tier: data
```

> **Prerequisite**: Pod labels must include `ameide.io/tier`. See [442-environment-isolation.md](442-environment-isolation.md) for label propagation.

---

### 3. Add Rate Limiting at Gateway

**Priority**: MEDIUM

Create a BackendTrafficPolicy for rate limiting:

**File**: `sources/charts/apps/gateway/templates/backendtrafficpolicy-ratelimit.yaml`

```yaml
{{- if .Values.rateLimit.enabled }}
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: {{ include "gateway.fullname" . }}-rate-limit
  namespace: {{ .Release.Namespace }}
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: {{ .Values.gateway.name | default "ameide" }}
  rateLimit:
    type: Global
    global:
      rules:
        - limit:
            requests: {{ .Values.rateLimit.requestsPerMinute | default 1000 }}
            unit: Minute
          clientSelectors:
            - headers:
                - name: X-API-Key
                  type: Distinct
{{- end }}
```

**Values configuration**:

```yaml
# sources/values/_shared/platform/platform-gateway.yaml
rateLimit:
  enabled: true
  requestsPerMinute: 1000

# sources/values/production/platform/platform-gateway.yaml
rateLimit:
  enabled: true
  requestsPerMinute: 5000  # Higher for production
```

### 2. Implement Network Policies

**Priority**: MEDIUM

#### 2.1 Default Deny Policy

**File**: `sources/charts/foundation/common/raw-manifests/files/networkpolicy-default-deny.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: {{ .Values.namespace }}
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

#### 2.2 Data Layer Isolation

**File**: `sources/charts/foundation/common/raw-manifests/files/networkpolicy-postgres.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-ingress
  namespace: {{ .Values.namespace }}
spec:
  podSelector:
    matchLabels:
      cnpg.io/cluster: postgres-ameide
  policyTypes:
    - Ingress
  ingress:
    # Allow from same namespace only
    - from:
        - podSelector: {}
    # Allow from Prometheus for monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ .Values.namespace }}
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - protocol: TCP
          port: 9187  # postgres exporter
```

#### 2.3 Redis Isolation

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-ingress
  namespace: {{ .Values.namespace }}
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: redis
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 6379
        - protocol: TCP
          port: 26379  # Sentinel
```

#### 2.4 Kafka Isolation

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kafka-ingress
  namespace: {{ .Values.namespace }}
spec:
  podSelector:
    matchLabels:
      strimzi.io/cluster: kafka
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 9092
        - protocol: TCP
          port: 9093
```

### 3. Add Health Check Standardization

**Priority**: LOW

Ensure all HTTPRoutes include health checks:

**File**: Update each HTTPRoute template

```yaml
# Add health endpoint to each route
- matches:
    - path:
        type: Exact
        value: /healthz
  backendRefs:
    - name: {{ .service.name }}
      port: {{ .service.port }}
```

**Or create a dedicated health route**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: health-check-route
spec:
  parentRefs:
    - name: ameide
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /healthz
      backendRefs:
        - name: envoy-gateway
          port: 8080
```

### 4. Configure Access Logging Sampling

**Priority**: LOW

Update EnvoyPatchPolicy for log sampling:

**File**: `sources/charts/apps/gateway/templates/envoypatch-accesslog.yaml`

```yaml
{{- if .Values.accessLog.sampling.enabled }}
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyPatchPolicy
metadata:
  name: {{ include "gateway.fullname" . }}-accesslog-sampling
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: {{ .Values.gateway.name | default "ameide" }}
  type: JSONPatch
  jsonPatches:
    - type: "type.googleapis.com/envoy.config.listener.v3.Listener"
      name: "{{ .Values.gateway.name }}/https"
      operation:
        op: add
        path: "/filter_chains/0/filters/0/typed_config/access_log/0/filter"
        value:
          runtime_filter:
            runtime_key: "access_log.sampling"
            percent_sampled:
              numerator: {{ .Values.accessLog.sampling.percent }}
              denominator: HUNDRED
{{- end }}
```

**Values**:

```yaml
# Production only - sample 10% of access logs
accessLog:
  sampling:
    enabled: true
    percent: 10
```

---

## Environment-Specific Configuration

| Setting | Dev | Staging | Production |
|---------|-----|---------|------------|
| Rate Limit | 100/min | 500/min | 5000/min |
| Network Policies | Permissive | Enforced | Enforced |
| Access Log Sampling | 100% | 50% | 10% |
| Health Check Interval | 30s | 15s | 10s |

---

## Implementation Steps

### Phase 0: Prerequisites (Blocker) ✅
1. [x] Add `ameide.io/environment` label to namespace Helm chart → **2025-12-04**
2. [x] Add `ameide.io/environment: cluster` to argocd, cert-manager namespaces → **2025-12-04**
3. [ ] Deploy namespace label changes via ArgoCD
4. [ ] Verify labels with `kubectl get ns --show-labels`

> **Implementation note**: Namespace labels added to `sources/values/_shared/foundation/foundation-namespaces.yaml`.
> Workload namespaces use `{{ .Values.environment }}` templating.
> Cluster-scoped namespaces labeled with `ameide.io/environment: cluster`.

### Phase 1: Cross-Environment Isolation ✅
5. [x] Create `deny-cross-env-ingress` NetworkPolicy template → **2025-12-04**
6. [ ] Deploy to dev namespace first, test ArgoCD sync still works
7. [ ] Verify dev pods cannot reach staging/prod
8. [ ] Roll out to staging and prod namespaces

> **Implementation note**: `deny-cross-environment` NetworkPolicy added to `foundation-namespaces.yaml`.
> Policy allows same-environment, cluster namespace, and kube-system traffic.

### Phase 2: Tier-Based Flow ✅
9. [x] Add `tier` field to app values files → **2025-12-04** (see 442)
10. [x] Update Helm charts to include `ameide.io/tier` label on pods → **2025-12-04**
11. [x] Create `tier-based-ingress` NetworkPolicy → **2025-12-04**
12. [ ] Test backend services can still reach data layer
13. [ ] Test frontend cannot directly reach data layer

> **Implementation note**: `tierLabels` helper added to all 12 app chart `_helpers.tpl` files.
> `tier-based-ingress` NetworkPolicy added to `foundation-namespaces.yaml`.
> Data tier only accepts from backend/platform/data tiers + cluster namespace.

### Phase 3: Gateway Policies ✅
14. [x] Create rate limiting BackendTrafficPolicy template → **2025-12-04**
15. [x] Add rate limit values per environment → **2025-12-04**
16. [ ] Add access log sampling for production

> **Implementation note**: `rate-limit-policy.yaml` template exists with auth & global policies.
> Rate limit values added to `_shared/platform/platform-gateway.yaml` and per-environment overrides.
> - Dev: 10 req/min auth, global disabled
> - Staging: 15 req/min auth, global disabled (was 2000 req/sec)
> - Production: 20 req/min auth, global disabled (was 5000 req/sec)

#### ⚠️ Incident: Global Rate Limiting Caused HTTP 500 (2025-12-07)

**Symptom**: https://www.ameide.io/ returned HTTP 500 for all requests.

**Root Cause**: The `rate-limit-global` BackendTrafficPolicy was targeting the Gateway with `type: Global` rate limiting, but Envoy Gateway wasn't configured with a rate limit backend (Redis). This caused the policy to be marked `Invalid`:

```
message: 'RateLimit: enable Ratelimit in the EnvoyGateway config to configure global rateLimit.'
reason: Invalid
status: "False"
type: Accepted
```

When an Invalid BackendTrafficPolicy is attached to the Gateway, Envoy Gateway generates `direct_response: {"status": 500}` for all routes instead of proper cluster routing. All 31 production routes were affected.

**Fix** (commit `58a29c1`):
- Disabled global rate limiting in staging and production:
  - `sources/values/staging/platform/platform-gateway.yaml`: `rateLimit.global.enabled: false`
  - `sources/values/production/platform/platform-gateway.yaml`: `rateLimit.global.enabled: false`

**To re-enable global rate limiting**, you must first configure Envoy Gateway with a rate limit service:

```yaml
# EnvoyGateway config (not currently deployed)
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyGateway
metadata:
  name: eg
  namespace: envoy-gateway-system
spec:
  rateLimit:
    backend:
      type: Redis
      redis:
        url: redis://redis-ratelimit:6379
```

**Lessons Learned**:
1. Global rate limiting requires additional infrastructure (Redis) that wasn't documented as a prerequisite
2. Invalid policies attached to Gateway cause catastrophic failures (HTTP 500 for all routes)
3. Local rate limiting (`rate-limit-auth`) works without Redis and remains enabled

### Phase 4: Component Policies (Future)
17. [ ] Review existing minio/keycloak/kafka policies
18. [ ] Ensure they align with tier-based model
19. [ ] Document network flow requirements

---

## Validation Checklist

### Cross-Environment Isolation
- [ ] ameide-dev pods cannot reach ameide-staging
- [ ] ameide-dev pods cannot reach ameide-prod
- [ ] ameide-staging pods cannot reach ameide-prod
- [ ] ArgoCD can still sync all namespaces
- [ ] Prometheus can still scrape all namespaces

### Tier-Based Flow
- [ ] Frontend tier receives external traffic
- [ ] Backend tier receives traffic from frontend
- [ ] Data tier only receives traffic from backend/platform
- [ ] Data tier cannot be reached directly from external

### Gateway Policies
- [ ] Rate limiting blocks requests above threshold
- [ ] Rate limiting allows bursts with backoff
- [ ] Access logs sampled (not 100%) in production
- [ ] Health check endpoints respond correctly
