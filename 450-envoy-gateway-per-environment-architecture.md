# 450: Envoy Gateway Per-Environment Architecture

**Status**: Implemented
**Created**: 2025-12-04
**Updated**: 2025-12-16

## Summary

This document describes the per-environment Envoy Gateway architecture where:
- **One Envoy Gateway controller** runs per cluster (cluster-shared control plane).
- Each environment owns its **namespaced Gateway/Routes** and a **namespaced EnvoyProxy** that defines data-plane infrastructure knobs (LB/static IP/annotations/tolerations) and telemetry.
- Per-environment infrastructure variability is expressed **per `Gateway`** via `Gateway.spec.infrastructure.parametersRef` (namespaced), instead of multiplying cluster-scoped `GatewayClass` objects.

## Related Documents

| Backlog | Topic | Status |
|---------|-------|--------|
| [457-azure-lb-redirect-gateway-consolidation.md](457-azure-lb-redirect-gateway-consolidation.md) | HTTP redirect consolidation, Azure LB fix | Current |
| [386-envoy-gateway-cert-tls-docs.md](386-envoy-gateway-cert-tls-docs.md) | TLS termination, cert-manager PKI | Partially superseded |
| [436-envoy-gateway-observability.md](436-envoy-gateway-observability.md) | Telemetry, access logs, Prometheus | Partially superseded |
| [446-namespace-isolation.md](446-namespace-isolation.md) | Namespace architecture | Current |
| [438-cert-manager-dns01-azure-workload-identity.md](438-cert-manager-dns01-azure-workload-identity.md) | DNS-01 certificate issuance | Current |
| [712-traffic-manager-first-redesign.md](712-traffic-manager-first-redesign.md) | Edge-first traffic manager (public DNS/TLS owned outside the cluster) | Proposed |
| [417-envoy-route-tracking.md](417-envoy-route-tracking.md) | HTTPRoute/GRPCRoute inventory | Current |
| [459-httproute-ownership.md](459-httproute-ownership.md) | HTTPRoute ownership migration (apps own routes) | Current |
| [old/115-keycloak-ssl-update.md](old/115-keycloak-ssl-update.md) | Legacy TLS termination docs | Archived |
| [old/107-gateway-api-migration.md](old/107-gateway-api-migration.md) | Gateway API migration | Archived |

## Why Per-Environment GatewayClasses

The previous architecture used a single shared `GatewayClass` named `envoy` for all environments. This caused issues because:

1. **GatewayClass is cluster-scoped** - Only one can exist with a given name
2. **parametersRef limitation** - GatewayClass can only reference ONE EnvoyProxy resource
3. **Race condition** - Whichever environment synced last would overwrite the GatewayClass parametersRef
4. **Wrong IPs assigned** - All environments got the last-synced environment's static IP configuration

**Updated solution (vendor/spec-aligned):**
- Keep `GatewayClass` ownership cluster-scoped (one shared class per controller).
- Put environment-specific infra differences on the **namespaced `Gateway`** via `spec.infrastructure.parametersRef`, pointing to the env’s `EnvoyProxy`.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              Kubernetes Cluster                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  CLUSTER-SCOPED: GatewayClass (shared)                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │   GatewayClass: envoy                                                         │  │
│  │   (no per-env parametersRef; per-Gateway overrides via infrastructureRef)     │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐         │
│  │   ameide-dev    │ ameide-staging  │  ameide-prod    │     argocd      │         │
│  │   namespace     │   namespace     │   namespace     │   namespace     │         │
│  ├─────────────────┼─────────────────┼─────────────────┼─────────────────┤         │
│  │ EnvoyProxy:     │ EnvoyProxy:     │ EnvoyProxy:     │ EnvoyProxy:     │         │
│  │ ameide-proxy-   │ ameide-proxy-   │ ameide-proxy-   │ cluster-proxy-  │         │
│  │ config          │ config          │ config          │ config          │         │
│  │                 │                 │                 │                 │         │
│  │ IP: 40.68.113.  │ IP: 108.142.    │ IP: 4.180.130.  │ IP: 20.160.216. │         │
│  │      216        │      228.7      │      190        │      7          │         │
│  │                 │                 │                 │                 │         │
│  │ Tolerations:    │ Tolerations:    │ Tolerations:    │ Tolerations:    │         │
│  │ ameide.io/      │ ameide.io/      │ ameide.io/      │ CriticalAddons  │         │
│  │ environment=dev │ environment=    │ environment=    │ Only            │         │
│  │                 │ staging         │ production      │                 │         │
│  ├─────────────────┼─────────────────┼─────────────────┼─────────────────┤         │
│  │ Gateway: ameide │ Gateway: ameide │ Gateway: ameide │ Gateway: cluster│         │
│  │ (HTTP+HTTPS)    │ (HTTP+HTTPS)    │ (HTTP+HTTPS)    │ (HTTP+HTTPS)    │         │
│  └─────────────────┴─────────────────┴─────────────────┴─────────────────┘         │
│                                                                                      │
│  Envoy Pods: All run in argocd namespace (managed by Envoy Gateway controller)      │
│  LoadBalancer Services: All in argocd namespace with per-environment static IPs     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Resource Mapping (updated)

| Environment | GatewayClass | EnvoyProxy (infra/telemetry) | Gateway Namespace | Static IP | Domain |
|-------------|--------------|------------------------------|-------------------|-----------|--------|
| Dev | `envoy` | `ameide-dev/ameide-proxy-config` | `ameide-dev` | 40.68.113.216 | `*.dev.ameide.io` |
| Staging | `envoy` | `ameide-staging/ameide-proxy-config` | `ameide-staging` | 108.142.228.7 | `*.staging.ameide.io` |
| Production | `envoy` | `ameide-prod/ameide-proxy-config` | `ameide-prod` | 4.180.130.190 | `*.ameide.io` |
| Cluster (ArgoCD) | `envoy-cluster` | `argocd/cluster-proxy-config` | `argocd` | 20.160.216.7 | `argocd.ameide.io` |

## TLS Termination

TLS is terminated at the Gateway level using wildcard certificates issued by cert-manager.

### Certificate Chain

```
Let's Encrypt (ACME)
       │
       ▼
ClusterIssuer: letsencrypt-prod
       │
       ▼
Certificate: ameide-wildcard-tls (per environment namespace)
       │
       ▼
Secret: ameide-wildcard-tls
       │
       ▼
Gateway listener (tls.mode: Terminate)
       │
       ▼
Plain HTTP to backend services
```

### Gateway TLS Configuration

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ameide
  namespace: ameide-dev  # per environment
spec:
  gatewayClassName: envoy-dev  # per environment
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      hostname: "*.dev.ameide.io"
      tls:
        mode: Terminate
        certificateRefs:
          - name: ameide-wildcard-tls
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: allowed
```

### HTTP to HTTPS Redirect

> **Updated 2025-12-05**: HTTP redirect is now handled by an HTTP listener on the main gateway,
> not a separate redirect gateway. See [457-azure-lb-redirect-gateway-consolidation.md](457-azure-lb-redirect-gateway-consolidation.md).

Each environment's main gateway includes an HTTP listener (port 80) for redirect:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ameide
  namespace: ameide-dev
spec:
  gatewayClassName: envoy-dev
  listeners:
    # HTTP listener for HTTP→HTTPS redirect
    - name: http
      port: 80
      protocol: HTTP
      hostname: "*.dev.ameide.io"
      allowedRoutes:
        namespaces:
          from: Same
    # HTTPS listener for TLS termination
    - name: https
      port: 443
      protocol: HTTPS
      hostname: "*.dev.ameide.io"
      tls:
        mode: Terminate
        certificateRefs:
          - name: ameide-wildcard-tls
```

An HTTPRoute on the HTTP listener issues the redirect:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ameide-redirect
  namespace: ameide-dev
spec:
  parentRefs:
    - name: ameide
      sectionName: http
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```

**Why single gateway instead of separate redirect gateway?**

Azure LoadBalancer doesn't reliably support multiple `LoadBalancer` Services sharing the same static IP.
Creating a separate redirect gateway caused Azure to reject the second service with IP conflicts.
The single-gateway pattern follows Gateway API / Envoy Gateway vendor recommendations.

## Observability

### Prometheus Metrics

Each EnvoyProxy enables Prometheus metrics scraping:

```yaml
spec:
  telemetry:
    metrics:
      prometheus: {}  # Empty object enables default metrics
```

Metrics are scraped by ServiceMonitors and available in Grafana.

### Access Logs (OpenTelemetry)

Access logs are sent to the OpenTelemetry Collector in each environment namespace:

```yaml
spec:
  telemetry:
    accessLog:
      settings:
        - format:
            type: JSON
            json:
              start_time: "%START_TIME%"
              method: "%REQ(:METHOD)%"
              path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
              protocol: "%PROTOCOL%"
              response_code: "%RESPONSE_CODE%"
              duration: "%DURATION%"
              upstream_host: "%UPSTREAM_HOST%"
              # ... additional fields
          sinks:
            - type: OpenTelemetry
              openTelemetry:
                host: otel-collector.ameide-dev.svc.cluster.local
                port: 4317
                resources:
                  k8s.cluster.name: "ameide-dev"
```

### Telemetry Flow

```
Request → Envoy Proxy
              │
              ├──► Prometheus metrics (/metrics:19001)
              │         │
              │         ▼
              │    ServiceMonitor → Prometheus → Grafana
              │
              └──► Access logs (JSON)
                        │
                        ▼
                   OpenTelemetry Collector (per-env namespace)
                        │
                        ▼
                   Loki → Grafana
```

## Implementation Files

### Chart Templates

| File | Purpose |
|------|---------|
| `sources/charts/apps/gateway/templates/gatewayclass.yaml` | GatewayClass with parametersRef to EnvoyProxy |
| `sources/charts/apps/gateway/templates/envoyproxy-telemetry.yaml` | EnvoyProxy with static IP, tolerations, telemetry |
| `sources/charts/apps/gateway/templates/gateway.yaml` | Main Gateway (HTTP+HTTPS listeners) |
| `sources/charts/apps/gateway/templates/http-redirect.yaml` | HTTP→HTTPS redirect HTTPRoute |

### Per-Environment Values

| File | Contents |
|------|----------|
| `sources/values/env/dev/platform/platform-gateway.yaml` | `gatewayClass.name: envoy-dev`, `gateway.className: envoy-dev`, static IP, tolerations |
| `sources/values/env/staging/platform/platform-gateway.yaml` | `gatewayClass.name: envoy-staging`, etc. |
| `sources/values/env/production/platform/platform-gateway.yaml` | `gatewayClass.name: envoy-prod`, etc. |

### Cluster Gateway (ArgoCD)

| File | Purpose |
|------|---------|
| `sources/charts/cluster/gateway/templates/gatewayclass.yaml` | `envoy-cluster` GatewayClass |
| `sources/charts/cluster/gateway/templates/envoyproxy.yaml` | ArgoCD EnvoyProxy with `CriticalAddonsOnly` toleration |

## Verification Commands

### Check GatewayClasses

```bash
kubectl get gatewayclass
# Expected: envoy-dev, envoy-staging, envoy-prod, envoy-cluster (all ACCEPTED: True)
```

### Check Gateways

```bash
kubectl get gateway -A
# Expected: One 'ameide' gateway per environment, PROGRAMMED: True with correct IPs
# Note: No separate 'ameide-redirect' gateways (consolidated into main gateway)
```

### Check EnvoyProxy Resources

```bash
kubectl get envoyproxy -A
# Expected: One per environment namespace
```

### Check Envoy Pods

```bash
kubectl get pods -A -l gateway.envoyproxy.io/owning-gateway-name
# Expected: 4 pods (1 per environment), all Running
# Note: Single gateway per environment means single Envoy deployment per environment
```

### Check Services with IPs

```bash
kubectl get svc -A -l gateway.envoyproxy.io/owning-gateway-name \
  -o custom-columns='NAME:.metadata.name,IP:.status.loadBalancer.ingress[*].ip'
```

### Test TLS

```bash
curl -v https://platform.dev.ameide.io 2>&1 | grep -E 'SSL|TLS|certificate'
```

## Troubleshooting

### Gateway shows PROGRAMMED: False

1. Check EnvoyProxy exists in correct namespace
2. Check `Gateway.spec.infrastructure.parametersRef` points to the correct EnvoyProxy name + namespace
3. Check Envoy pods are Running (may be Pending due to taints)

### Wrong IP assigned

1. Verify `Gateway.spec.addresses` requests the correct IP for the environment
2. Check Azure annotations are correct (resource group, health probe path)
3. If migrating from the legacy `loadBalancerIP`/annotation approach, delete the generated LoadBalancer Service once to force recreation with the requested IP

### Envoy pods Pending

1. Check tolerations match node taints
2. For dev/staging/prod: need `ameide.io/environment` toleration
3. For cluster: need `CriticalAddonsOnly` toleration

### TLS certificate errors

1. Check Certificate resource is Ready: `kubectl get certificate -A`
2. Check Secret exists: `kubectl get secret ameide-wildcard-tls -n <namespace>`
3. Check Gateway references correct secret name

## Migration Notes

### From Shared GatewayClass

If migrating from the old shared `envoy` GatewayClass:

1. The old per-environment `GatewayClass` objects (`envoy-dev`, `envoy-staging`, `envoy-prod`, `envoy-local`) may still exist; they can be deleted once all Gateways reference the shared `GatewayClass/envoy` and use per-Gateway infra refs.
2. Old Envoy services with wrong IPs should be deleted to force recreation
3. ArgoCD will recreate resources with correct configuration on next sync

### Document Updates Needed

The following documents reference the old architecture and should note this document:

- **386-envoy-gateway-cert-tls-docs.md**: Update to reflect controller in `argocd` + per-env `EnvoyProxy`
- **436-envoy-gateway-observability.md**: Update to reflect controller in `argocd` + per-env `EnvoyProxy`

## References

- [Envoy Gateway Docs: EnvoyProxy API](https://gateway.envoyproxy.io/docs/api/extension_types/#envoyproxy)
- [Gateway API: GatewayClass](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/)
- [Azure Load Balancer Annotations](https://learn.microsoft.com/en-us/azure/aks/load-balancer-standard)
