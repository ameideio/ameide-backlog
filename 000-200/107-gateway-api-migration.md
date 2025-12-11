# Gateway API Migration Strategy

## Executive Summary

Migrate from Ingress v1 to Gateway API for cleaner, more maintainable routing configuration that better aligns with our domain strategy and microservices architecture.

## Current State (✅ MIGRATED - August 2025, Enhanced January 2025)

### Problems with Ingress v1 (RESOLVED)
1. ~~Template Complexity~~ → ✅ Clean HTTPRoute templates
2. ~~Non-Standard Values~~ → ✅ Standard Gateway API resources
3. ~~Maintenance Burden~~ → ✅ Simple, declarative configs
4. ~~Poor Separation~~ → ✅ Gateway (infra) vs HTTPRoute (app)

### What We Have Now
- ✅ Envoy Gateway v1.5.0 (replaced NGINX Ingress)
- ✅ Unified Envoy for all traffic (HTTP, HTTPS, gRPC-Web)
- ✅ cert-manager integrated with Gateway API
- ✅ Domain strategy with .test for local, .io for production
- ✅ **HTTPS-only architecture** with HTTP→HTTPS redirects
- ✅ **Split endpoints for OAuth** - External HTTPS, internal HTTP
- ✅ **X-Forwarded headers** properly configured for proxies

## Gateway API Benefits

### Clean Architecture
```yaml
# Before: Ingress v1 (mixed concerns)
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  rules:
    - host: platform.ameide.io
      http:
        paths:
          - path: /
            backend:  # Routing + backend mixed
              service:
                name: www-ameide-canvas
                port:
                  number: 3001

# After: Gateway API (separated concerns)
---
# Infrastructure team owns Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ameide
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      port: 80
      protocol: HTTP
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        certificateRefs:
          - name: ameide-wildcard-tls
---
# App team owns HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: platform
spec:
  parentRefs:
    - name: ameide
  hostnames:
    - platform.ameide.io
  rules:
    - backendRefs:
        - name: www-ameide-canvas
          port: 3001
```

### Perfect Domain Strategy Alignment

Each service gets its own HTTPRoute (aligned with port spec):

**Local Development (.test domains on ports 8080/8443):**
- `platform.dev.ameide.io` → HTTPRoute → www-ameide-canvas (port 3001) ✅
- `dev.ameide.io` → HTTPRoute → www-ameide (port 3000) ✅
- `portal.dev.ameide.io` → HTTPRoute → www-ameide-portal (port 3002) ✅
- `portal-canvas.dev.ameide.io` → HTTPRoute → www-ameide-portal-canvas (port 3003) ✅
- `api.dev.ameide.io` → HTTPRoute → envoy (port 8000 for gRPC-Web) ✅
- `auth.dev.ameide.io` → HTTPRoute → keycloak (port 4000) ⏳
- `metrics.dev.ameide.io` → HTTPRoute → grafana (port 9001) ⏳

**Production (.io domains on standard ports):**
- Same services on `.io` domains with standard HTTPS (443)

**Port Alignment**: All services follow `backlog/100-port-numbering-forward.md`

### Native gRPC Support

```yaml
apiVersion: gateway.networking.k8s.io/v1  # GA since Gateway API v1.1.0
kind: GRPCRoute
metadata:
  name: api
spec:
  parentRefs:
    - name: ameide
  hostnames: [api.ameide.io]
  rules:
    - backendRefs:
        - name: envoy
          port: 8000
```

## Implementation Plan

### Direct Adoption Approach (Selected)
**Decision**: Skip Ingress v1 fixes, go directly to Gateway API

### ✅ Phase 1: Infrastructure Setup (COMPLETE)
**Goal**: Deploy Gateway API infrastructure

1. **Envoy Gateway Configuration** ✅
   - Configured in helmfile.yaml using official OCI chart v1.5.0
   - Provides unified Envoy control plane for all traffic types

2. **Gateway Resources Created** ✅
   - Gateway chart at `charts/platform/gateway/` with:
     - GatewayClass for Envoy controller
     - Gateway with HTTP, HTTPS, and gRPC listeners
     - HTTP to HTTPS redirect via HTTPRoute
     - Namespace labeling for access control

3. **Cert-Manager Integration** ✅
   - Gateway annotated with `cert-manager.io/cluster-issuer`
   - Automatic certificate provisioning in Gateway namespace
   - Environment-specific issuers (self-signed, staging, prod)

4. **Policy Framework** ✅
   - BackendTLSPolicy templates for mTLS to services
   - BackendTrafficPolicy for retries, timeouts, circuit breakers
   - Environment-specific configurations (dev, staging, prod)

5. **Cross-Namespace Support** ✅
   - ReferenceGrant templates for cross-namespace access
   - Selector-based namespace access with `gateway-access: allowed` label

### ✅ Phase 2: Service Migration (COMPLETE - August 11, 2025)
**Goal**: Convert all services to use HTTPRoute

1. **HTTPRoute Template Created** ✅
   - Created flexible template at `www-ameide-canvas/templates/httproute.yaml`
   - Supports single or multiple hostnames
   - Optional custom rules and filters
   - Route-level timeouts (Gateway API v1.1+)
   - Cross-namespace gateway references

2. **Services Migrated** ✅:
   - ✅ www-ameide-canvas → platform.dev.ameide.io, platform.ameide.io
   - ✅ www-ameide → dev.ameide.io, www.dev.ameide.io
   - ✅ www-ameide-portal → portal.dev.ameide.io
   - ✅ www-ameide-portal-canvas → portal-canvas.dev.ameide.io
   - ✅ API routes → api.dev.ameide.io, api.ameide.io (via HTTPRoute with CORS)
   - ⏳ keycloak → auth.dev.ameide.io (pending final migration)
   - ⏳ grafana → metrics.dev.ameide.io (pending final migration)

2. **Create GatewayClass**
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: GatewayClass
   metadata:
     name: envoy
   spec:
     controllerName: gateway.envoyproxy.io/gatewayclass-controller
   ```

3. **Deploy Gateway**
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: Gateway
   metadata:
     name: ameide
     namespace: ameide
   spec:
     gatewayClassName: envoy
     listeners:
       - name: http
         port: 80
         protocol: HTTP
         allowedRoutes:
           namespaces:
             from: Same
       - name: https
         port: 443
         protocol: HTTPS
         tls:
           mode: Terminate
           certificateRefs:
             - name: ameide-wildcard-tls
         allowedRoutes:
           namespaces:
             from: Same
   ```

4. **Create HTTPRoutes for Each Service**
   - Start with non-critical service
   - Test thoroughly
   - Roll out to remaining services

### ✅ Phase 3: Migration (COMPLETE - August 2025)
**Goal**: Complete cutover to Gateway API

1. **DNS/Load Balancer Updated** ✅
   - Gateway service exposed on ports 8080/8443 for local
   - k3d loadbalancer configured for port mappings

2. **Ingress Resources Deprecated** ✅
   - No Ingress objects remain
   - All routing via HTTPRoute/GRPCRoute

3. **NGINX Ingress Controller Removed** ✅
   - Replaced entirely by Envoy Gateway
   - All resources cleaned up

## TLS Certificate Ownership (✅ IMPLEMENTED)

**Single Source of Truth: The Gateway owns all TLS certificates.**

- **Wildcard certificate** (`*.dev.ameide.io` for local, `*.ameide.io` for prod) ✅
- **cert-manager** provisions/renews automatically ✅
- **Self-signed ClusterIssuer** for local development ✅
- **HTTPRoutes/GRPCRoutes** are TLS-agnostic - they only specify hostnames ✅
- **Services** speak plain HTTP internally (no TLS configuration needed) ✅

This eliminates:
- Certificate distribution complexity
- Cross-namespace secret references
- Service-level TLS configuration
- Certificate renewal coordination

## Helm Chart Updates

### Current (Broken)
```yaml
# values.yaml - Non-standard structure
ingress:
  hosts:
    - host: platform.ameide.io
      paths:
        - path: /
          pathType: Prefix
          # No backend specification!
```

### With Gateway API
```yaml
# values.yaml - Simple, clear structure
httproute:
  enabled: true
  gateway: ameide
  hostname: platform.ameide.io
  service:
    name: www-ameide-canvas  # Optional, can default
    port: 3001
```

### Template Simplification
```yaml
# templates/httproute.yaml - No complex logic needed
{{- if .Values.httproute.enabled }}
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: {{ include "chart.fullname" . }}
spec:
  parentRefs:
    - name: {{ .Values.httproute.gateway }}
  hostnames:
    - {{ .Values.httproute.hostname }}
  rules:
    - backendRefs:
        - name: {{ .Values.httproute.service.name | default (include "chart.fullname" .) }}
          port: {{ .Values.httproute.service.port | default .Values.service.port }}
{{- end }}
```

## Unified Envoy Stack (✅ ACHIEVED)

### Previous: Two Separate Systems
1. ~~NGINX Ingress → Web traffic~~
2. ~~Standalone Envoy Proxy → gRPC-Web translation~~

### Current: Single Envoy Gateway ✅
```
Envoy Gateway v1.5.0
  ├── HTTP/HTTPS → Web services ✅
  ├── gRPC → Native gRPC services ✅
  └── gRPC-Web → Browser clients ✅
```

Achieved Benefits:
- ✅ Single control plane
- ✅ Consistent configuration
- ✅ Better observability
- ✅ Reduced complexity

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Gateway API maturity | Use stable v1 features only |
| Team familiarity | Training and documentation |
| Migration disruption | Parallel run with gradual cutover |
| Rollback complexity | Backup Gateway configuration before changes |

## Success Criteria

- ✅ All services accessible via Gateway API
- ✅ No Ingress v1 resources remaining
- ✅ Simplified Helm templates with HTTPRoute
- ✅ Unified Envoy stack (v1.5.0)
- ✅ Documentation updated
- ⏳ Team training on Gateway API (ongoing)

## Timeline (✅ COMPLETE)

- ✅ **Phase 1**: Gateway API infrastructure deployed (August 2025)
- ✅ **Phase 2**: Services migrated to HTTPRoute (August 2025)
- ✅ **Phase 3**: Verification and documentation (August 2025)

## Implementation Summary

**Successfully adopted Gateway API with Envoy Gateway v1.5.0**

Achieved:
- ✅ No Ingress v1 resources remain
- ✅ Clean, modern architecture implemented
- ✅ Unified Envoy stack for all traffic types
- ✅ Future-proof with Kubernetes Gateway API v1.0+
- ✅ Environment-specific configurations (.test for local, .io for prod)
- ✅ cert-manager integration with self-signed certificates

## Remaining Gaps

### Minor Pending Items
1. **Keycloak HTTPRoute**: Currently accessible but needs formal HTTPRoute definition
2. **Grafana HTTPRoute**: Metrics dashboard needs HTTPRoute for metrics.dev.ameide.io
3. **Production TLS**: Configure Let's Encrypt issuers for staging/production

### Future Enhancements
1. **GRPCRoute**: Consider native GRPCRoute for API services (currently using HTTPRoute)
2. **Traffic Policies**: Implement rate limiting, circuit breakers via BackendTrafficPolicy
3. **Observability**: Add Gateway metrics to Prometheus/Grafana

## References

- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway](https://gateway.envoyproxy.io/)
- [Gateway API vs Ingress](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/)
- [Cert-manager Gateway API Support](https://cert-manager.io/docs/usage/gateway/)
