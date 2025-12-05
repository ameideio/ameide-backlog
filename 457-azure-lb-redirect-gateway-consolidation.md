# 457: Azure LoadBalancer Redirect Gateway Consolidation

**Status**: Implemented
**Created**: 2025-12-05

## Summary

This document describes the consolidation of HTTP→HTTPS redirect functionality from a separate redirect gateway into the main gateway's HTTP listener. This change was necessary to avoid Azure LoadBalancer static IP conflicts.

## Related Documents

| Backlog | Topic | Status |
|---------|-------|--------|
| [450-envoy-gateway-per-environment-architecture.md](450-envoy-gateway-per-environment-architecture.md) | Per-environment gateway architecture | Updated |
| [441-networking.md](441-networking.md) | Networking improvements | Current |
| [446-namespace-isolation.md](446-namespace-isolation.md) | Namespace isolation | Current |

## Problem Statement

Each environment had two Gateway resources:
- `ameide` - Main gateway (HTTPS on port 443)
- `ameide-redirect` - Redirect gateway (HTTP on port 80)

Both gateways were configured to use the same static Azure public IP (e.g., `40.68.113.216` for dev). When Envoy Gateway created LoadBalancer services for these gateways, Azure rejected the second service with:

```
Error syncing load balancer: failed to ensure load balancer: checkLoadBalancerResourcesConflicts:
service port http-80 is trying to consume the port 80 which is being referenced by an existing
loadBalancing rule ad11465f47fec4f97a90294850276c2b-TCP-80 with the same protocol Tcp and
frontend IP config
```

### Root Cause

Azure Kubernetes Service (AKS) cloud provider doesn't reliably support multiple `LoadBalancer` type Services sharing the same static public IP address. While Azure Load Balancer *can* technically support multiple rules on the same frontend IP with different ports, the AKS implementation has sharp edges:

- Each Service creates its own frontend IP configuration
- Azure detects this as a conflict when ports overlap or when the same IP is reused
- Known issue documented in [Azure/AKS#1621](https://github.com/Azure/AKS/issues/1621)

### Why Production Worked Initially

Production's redirect gateway initially got an IP because both services were created simultaneously during the first sync. The Azure LB was able to configure both rules in a single reconciliation. However, when services were deleted and recreated at different times (as happened during dev gateway debugging), the conflict surfaced.

## Solution

Follow the Gateway API and Envoy Gateway vendor-recommended pattern: **single Gateway with multiple listeners** for HTTP (80) and HTTPS (443).

### Before (Two Gateways)

```yaml
# Main gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ameide
spec:
  gatewayClassName: envoy-dev
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
---
# Redirect gateway (PROBLEMATIC)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ameide-redirect
spec:
  gatewayClassName: envoy-dev  # Same class = same EnvoyProxy = same static IP
  listeners:
    - name: http
      port: 80
      protocol: HTTP
```

### After (Single Gateway with HTTP Listener)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ameide
spec:
  gatewayClassName: envoy-dev
  listeners:
    - name: http       # HTTP listener for redirect
      port: 80
      protocol: HTTP
      hostname: "*.dev.ameide.io"
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      port: 443
      protocol: HTTPS
      hostname: "*.dev.ameide.io"
      tls:
        mode: Terminate
        certificateRefs:
          - name: ameide-wildcard-tls
```

This creates a single LoadBalancer service with both ports:
```
80:30585/TCP,443:31824/TCP,8000:32431/TCP,9000:32500/TCP
```

## Implementation

### Files Modified

| File | Change |
|------|--------|
| `sources/charts/apps/gateway/templates/http-redirect.yaml` | Added logic to detect HTTP listener on main gateway and point redirect HTTPRoute accordingly |
| `sources/values/dev/platform/platform-gateway.yaml` | `redirectGateway.enabled: false`, added HTTP listener to `gateway.listeners` |
| `sources/values/staging/platform/platform-gateway.yaml` | Same changes |
| `sources/values/production/platform/platform-gateway.yaml` | Same changes |

### Template Logic

The `http-redirect.yaml` template now supports both patterns:

```yaml
{{- /* Check if redirect gateway is enabled OR if main gateway has an HTTP listener */ -}}
{{- $hasHttpListener := false }}
{{- range .Values.gateway.listeners }}
  {{- if and (eq (.protocol | default "HTTP") "HTTP") (eq (.port | default 80 | int) 80) }}
    {{- $hasHttpListener = true }}
  {{- end }}
{{- end }}
{{- if or .Values.redirectGateway.enabled $hasHttpListener }}
# HTTPRoute points to appropriate gateway
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      {{- if .Values.redirectGateway.enabled }}
      name: {{ .Values.redirectGateway.name }}
      sectionName: {{ .Values.redirectGateway.listener.name | default "http" }}
      {{- else }}
      name: {{ .Values.gateway.name }}
      sectionName: http
      {{- end }}
{{- end }}
```

### Values Configuration Pattern

```yaml
# Disable separate redirect gateway
redirectGateway:
  enabled: false

# Add HTTP listener to main gateway
gateway:
  className: envoy-dev
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      hostname: "*.dev.ameide.io"
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      port: 443
      # ... rest of HTTPS config
```

## Vendor Documentation Alignment

This solution aligns with official vendor guidance:

- **Gateway API**: [HTTP Redirects and Rewrites](https://gateway-api.sigs.k8s.io/guides/http-redirect-rewrite/) shows single Gateway with both HTTP and HTTPS listeners
- **Envoy Gateway**: [HTTP Redirects](https://gateway.envoyproxy.io/docs/tasks/traffic/http-redirect/) uses the same single-gateway pattern
- **Microsoft Azure**: [AKS#1621](https://github.com/Azure/AKS/issues/1621) confirms static IP reuse issues

## Verification

```bash
# Check single gateway per environment
kubectl get gateway -A
# Expected: One 'ameide' gateway per environment, no 'ameide-redirect'

# Check LoadBalancer service has both ports
kubectl get svc -n argocd -l gateway.envoyproxy.io/owning-gateway-name=ameide
# Expected: Single service with 80, 443, 8000, 9000 ports

# Check gateway is programmed with IP
kubectl get gateway ameide -n ameide-dev -o wide
# Expected: PROGRAMMED=True, ADDRESS=40.68.113.216

# Test HTTP redirect
curl -I http://platform.dev.ameide.io
# Expected: 301 Moved Permanently, Location: https://platform.dev.ameide.io
```

## Related Incidents

### 2025-12-05: Initial Discovery

- **Symptom**: `ameide-redirect` gateway stuck with no IP, `dev-platform-gateway` showing Progressing
- **Investigation**: Found Azure LB rule conflict in service events
- **Resolution**:
  1. Updated template and values to consolidate HTTP listener
  2. Deleted orphaned redirect gateway resources
  3. Synced ArgoCD apps
- **Commits**:
  - `a51ee50` - fix(gateway): consolidate HTTP redirect into main gateway
  - `137753f` - fix(gateway): apply HTTP listener consolidation to staging and production
