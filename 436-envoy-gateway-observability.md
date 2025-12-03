# Backlog 436: Envoy Gateway Observability

**Status**: Implemented
**Updated**: 2025-12-03

## Overview

This document specifies the observability configuration for Envoy Gateway, including Prometheus metrics and OpenTelemetry access logs. The implementation uses an `EnvoyProxy` resource referenced by the `GatewayClass` via `parametersRef`.

## Related Backlogs

| Backlog | Topic | Relationship |
|---------|-------|--------------|
| [434-unified-environment-naming.md](434-unified-environment-naming.md) | Environment naming & gateway architecture | Defines static IP flow: Bicep → globals.yaml → EnvoyProxy template |
| [386-envoy-gateway-cert-tls-docs.md](386-envoy-gateway-cert-tls-docs.md) | TLS & cert-manager PKI | Control-plane certificates for xDS |
| [417-envoy-route-tracking.md](417-envoy-route-tracking.md) | Route inventory | All routes emit access logs via this telemetry configuration |
| [438-cert-manager-dns01-azure-workload-identity.md](438-cert-manager-dns01-azure-workload-identity.md) | DNS-01 certificates | Certificate issuance for gateway TLS |

## Background

The `ClientTrafficPolicy` resource does not support `spec.telemetry` in Envoy Gateway v1.6.0. The invalid template was removed in commit `f2feec8`. Telemetry must be configured via an `EnvoyProxy` resource with `spec.telemetry`, referenced from the `GatewayClass`.

## Architecture

```
GatewayClass (envoy)
    |
    +-- parametersRef --> EnvoyProxy (ameide-proxy-config)
                              |
                              +-- spec.provider.kubernetes.envoyService
                              |       |
                              |       +-- loadBalancerIP: <from globals.yaml>
                              |       +-- annotations: azure-load-balancer-resource-group
                              |
                              +-- spec.telemetry
                                    |
                                    +-- metrics.prometheus: {}
                                    |
                                    +-- accessLog.settings
                                          |
                                          +-- format: JSON
                                          +-- sinks: OpenTelemetry -> otel-collector:4317
```

> **Note (2025-12-03):** EnvoyProxy is deployed to `ameide` namespace (not `envoy-gateway-system`)
> because the Envoy Gateway controller runs there. Static IP configuration added via
> `spec.provider.kubernetes.envoyService.loadBalancerIP`. See [434](434-unified-environment-naming.md#appendix-certificate--gateway-architecture).

## Implementation

### Files

| File | Purpose |
|------|---------|
| `sources/charts/apps/gateway/templates/envoyproxy-telemetry.yaml` | EnvoyProxy resource with telemetry config |
| `sources/charts/apps/gateway/templates/gatewayclass.yaml` | GatewayClass with `parametersRef` |
| `sources/charts/apps/gateway/values.yaml` | Base telemetry configuration |
| `sources/values/dev/platform/platform-gateway.yaml` | Dev cluster name override |
| `sources/values/staging/platform/platform-gateway.yaml` | Staging cluster name override |
| `sources/values/production/platform/platform-gateway.yaml` | Production cluster name override |

### EnvoyProxy Resource

Deployed to `ameide` namespace (where Envoy Gateway controller runs):

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: ameide-proxy-config
  namespace: ameide
spec:
  # Static IP configuration - see 434 for Bicep → globals.yaml flow
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        loadBalancerIP: "52.136.217.181"  # From globals.yaml:azure.envoyPublicIpAddress
        annotations:
          service.beta.kubernetes.io/azure-load-balancer-resource-group: "Ameide"
  telemetry:
    metrics:
      prometheus: {}  # Note: schema uses empty object, not "enabled: true"
    accessLog:
      settings:
        - format:
            type: JSON
            json:
              start_time: "%START_TIME%"
              method: "%REQ(:METHOD)%"
              # ... (full format in template)
          sinks:
            - type: OpenTelemetry
              openTelemetry:
                host: otel-collector.ameide.svc.cluster.local
                port: 4317
                resources:
                  k8s.cluster.name: "ameide-{env}"
```

### GatewayClass Reference

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  description: "Envoy Gateway for AMEIDE platform"
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: ameide-proxy-config
    namespace: ameide  # Updated: was envoy-gateway-system
```

### Values Configuration

Base values in `sources/charts/apps/gateway/values.yaml`:

```yaml
telemetry:
  enabled: true
  name: ameide-proxy-config
  namespace: ameide  # Updated: controller runs in ameide namespace
  metrics:
    prometheus: {}
  accessLog:
    openTelemetry:
      host: otel-collector.ameide.svc.cluster.local
      port: 4317
      resources:
        k8s.cluster.name: "ameide"

# Static IP configuration (per-environment)
infrastructure:
  useStaticIP: true  # Template reads azure.envoyPublicIpAddress from globals
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "Ameide"
```

### Environment-Specific Cluster Names

| Environment | Cluster Name | File |
|-------------|--------------|------|
| Dev | `ameide-dev` | `sources/values/dev/platform/platform-gateway.yaml` |
| Staging | `ameide-staging` | `sources/values/staging/platform/platform-gateway.yaml` |
| Production | `ameide-production` | `sources/values/production/platform/platform-gateway.yaml` |

## Telemetry Flow

```
HTTP/gRPC Request
       |
       v
  Envoy Proxy (data plane)
       |
       +---> Prometheus metrics endpoint (/metrics)
       |         |
       |         v
       |    ServiceMonitor (platform-envoy-gateway)
       |         |
       |         v
       |    Prometheus --> Grafana
       |
       +---> Access logs (JSON)
                 |
                 v
            OpenTelemetry Collector
            (otel-collector.ameide.svc.cluster.local:4317)
                 |
                 v
            Loki --> Grafana
```

## Access Log Format

JSON format using Envoy Gateway defaults:

```json
{
  "start_time": "%START_TIME%",
  "method": "%REQ(:METHOD)%",
  "x-envoy-origin-path": "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%",
  "protocol": "%PROTOCOL%",
  "response_code": "%RESPONSE_CODE%",
  "response_flags": "%RESPONSE_FLAGS%",
  "response_code_details": "%RESPONSE_CODE_DETAILS%",
  "connection_termination_details": "%CONNECTION_TERMINATION_DETAILS%",
  "upstream_transport_failure_reason": "%UPSTREAM_TRANSPORT_FAILURE_REASON%",
  "bytes_received": "%BYTES_RECEIVED%",
  "bytes_sent": "%BYTES_SENT%",
  "duration": "%DURATION%",
  "x-envoy-upstream-service-time": "%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%",
  "x-forwarded-for": "%REQ(X-FORWARDED-FOR)%",
  "user-agent": "%REQ(USER-AGENT)%",
  "x-request-id": "%REQ(X-REQUEST-ID)%",
  ":authority": "%REQ(:AUTHORITY)%",
  "upstream_host": "%UPSTREAM_HOST%",
  "upstream_cluster": "%UPSTREAM_CLUSTER%",
  "upstream_local_address": "%UPSTREAM_LOCAL_ADDRESS%",
  "downstream_local_address": "%DOWNSTREAM_LOCAL_ADDRESS%",
  "downstream_remote_address": "%DOWNSTREAM_REMOTE_ADDRESS%",
  "requested_server_name": "%REQUESTED_SERVER_NAME%",
  "route_name": "%ROUTE_NAME%"
}
```

## Existing ServiceMonitors

Prometheus scraping configured via ServiceMonitors (file: `sources/charts/apps/gateway/templates/servicemonitors.yaml`):

| ServiceMonitor | Target | Port | Interval |
|----------------|--------|------|----------|
| `platform-envoy-gateway` | Envoy Gateway control plane | metrics | 30s |
| `platform-envoy-ratelimit` | Rate limiter | metrics | 30s |

## Verification

### Check EnvoyProxy resource

```bash
kubectl get envoyproxy -n ameide
kubectl describe envoyproxy ameide-proxy-config -n ameide

# Verify static IP is configured
kubectl get envoyproxy ameide-proxy-config -n ameide -o jsonpath='{.spec.provider}'
```

### Check GatewayClass references EnvoyProxy

```bash
kubectl get gatewayclass envoy -o yaml | grep -A5 parametersRef
```

### Verify access logs in Loki

```bash
curl -s "http://$LOKI_IP:3100/loki/api/v1/query_range" \
  --data-urlencode "query={exporter=\"OTLP\"}" | jq '.data.result[0].values'
```

### Verify Prometheus metrics

```bash
kubectl port-forward -n ameide svc/envoy-gateway 19001:19001
curl http://localhost:19001/metrics
```

## Future Enhancements

The following features are available but not currently configured:

- **CEL filtering**: Filter logs based on headers/request attributes
- **Separate Listener logs**: Different format/sink for connection-level failures
- **Route metadata enrichment**: Add HTTPRoute/GRPCRoute metadata to logs
- **gRPC ALS sink**: Alternative to OpenTelemetry for access log streaming

## References

- [Envoy Gateway Observability Docs](https://gateway.envoyproxy.io/docs/tasks/observability/proxy-accesslog/)
- [EnvoyProxy API Reference](https://gateway.envoyproxy.io/docs/api/extension_types/#envoyproxy)
- [Envoy Gateway Metrics](https://gateway.envoyproxy.io/docs/tasks/observability/proxy-metrics/)
