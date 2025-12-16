> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 109: Local Development Domain Access

## Problem Statement

When developing AMEIDE Core platform locally using DevContainer + k3d, accessing services from the macOS host requires proper domain resolution for authentication and routing to work correctly. The platform uses domain-based routing (e.g., `auth.ameide.io` for Keycloak), but adding production domains to `/etc/hosts` would break access to the actual production site hosted on Azure.

## North Star Goals

* Click a URL, auth works, routes resolve, no "but on my machine..."
* No collisions with production DNS
* Minimal sudo (one-time only), no privileged ports during day-to-day
* Works on macOS + VS Code Dev Container + k3d

## Recommended Solution: The Opinionated Path

> **Update (2025-12-14):** The current GitOps local environment uses the `*.local.ameide.io` domain and also exposes ArgoCD via the local Gateway (`https://argocd.local.ameide.io`) in addition to the fallback `https://localhost:8443` port-forward.

### Core Design Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Domains** | `*.local.ameide.io` | Clear “this is local”, no collisions with production hostnames |
| **DNS** | `dnsmasq` wildcard → 127.0.0.1 | No manual `/etc/hosts` maintenance |
| **TLS** | cert-manager with self-signed issuer | Kubernetes-native cert management |
| **Cluster** | k3d | Lightweight, fast, production-like |
| **Routing** | Gateway API + Envoy Gateway | Production parity, host-based routing |
| **Host Ports** | 8080/8443 | Non-privileged, no daily sudo needed |
| **Auth** | Keycloak with edge proxy config | Proper HTTPS redirects, secure cookies |

### Why This Over Alternatives

#### Domain Choice: `.test` vs `.local` vs Others

| Domain | Pros | Cons | Verdict |
|--------|------|------|---------|
| `.local` | Common in tutorials | mDNS conflicts on macOS, VPN issues, no wildcards | ❌ Avoid |
| `.test` | Reserved for testing, no conflicts, wildcard-friendly | Requires DNS setup | ✅ Recommended |
| `.localhost` | Auto-loopback in browsers | Some tools don't support it | ⚠️ Limited |
| `.127.0.0.1.nip.io` | Zero setup | Ugly URLs, external dependency | 🔧 Quick demos only |

#### Port Strategy: Why 8080/8443

* **Ports < 1024 are privileged** - require sudo on macOS/Linux
* **VS Code won't forward privileged ports** from DevContainers
* **Industry standard** - Most K8s local dev uses 8080/8443
* **Clean separation** - Different from production, clear it's local

## Implementation Guide

### One-Time Setup (5-10 minutes)

```bash
# 1. Install tools
brew install k3d kubectl dnsmasq jq helm

# 2. Configure wildcard DNS for *.local.ameide.io -> 127.0.0.1
CONF_DIR="$(brew --prefix)/etc/dnsmasq.d"
sudo mkdir -p "$CONF_DIR"
echo 'address=/local.ameide.io/127.0.0.1' | sudo tee "$CONF_DIR/ameide.conf" >/dev/null
sudo brew services start dnsmasq
sudo mkdir -p /etc/resolver
echo 'nameserver 127.0.0.1' | sudo tee /etc/resolver/local.ameide.io >/dev/null

# Note: TLS certificates are now managed by cert-manager in Kubernetes
# No need for mkcert - self-signed certificates are automatically generated

# 4. Create k3d registry (one-time, outside cluster)
k3d registry create ameide-reg --port 32000

# 5. Create k3d cluster with registry config (use config file for repeatability)
cat > k3d-ameide.yaml <<EOF
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: ameide
servers: 1
agents: 1
ports:
  - port: 8080:80@loadbalancer
  - port: 8443:443@loadbalancer
registries:
  use:
    - k3d-ameide-reg:32000
options:
  k3s:
    extraArgs:
      - arg: --disable=traefik
        nodeFilters:
          - server:*
EOF

k3d cluster create --config k3d-ameide.yaml

# Note: For Docker Desktop, add to Settings → Docker Engine:
# "insecure-registries": ["docker.io/ameide"]

# 5. Bootstrap GitOps (recommended, idempotent)
# Gateway API CRDs, Envoy Gateway, cert-manager, and all platform routing are installed by Argo CD.
./bootstrap/bootstrap.sh --config bootstrap/configs/local.yaml --install-argo --apply-root-apps --wait-tiers

# Verify the controller is installed (control-plane runs in `argocd`)
kubectl wait --for=condition=Available -n argocd deployment/envoy-gateway --timeout=60s
kubectl get gatewayclass

# 6. Verify environment namespace exists (managed by GitOps)
kubectl get ns ameide-local

# Certificate will be automatically created as ameide-wildcard-tls
```

### Gateway Configuration

```yaml
# gitops/ameide-gitops/sources/values/env/local/apps/platform/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ameide
  namespace: ameide
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      hostname: "*.local.ameide.io"
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: allowed
    - name: https
      hostname: "*.local.ameide.io"
      port: 443
      protocol: HTTPS
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

### HTTPRoute Examples

```yaml
# Keycloak
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: keycloak
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames: ["auth.local.ameide.io"]
  rules:
    - backendRefs:
        - name: keycloak
          port: 8080

---
# Main Application
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: www-ameide
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames: ["local.ameide.io", "www.local.ameide.io"]
  rules:
    - backendRefs:
        - name: www-ameide
          port: 3000

---
# Platform Application
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: www-ameide-platform
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames: ["platform.local.ameide.io"]
  rules:
    - backendRefs:
        - name: www-ameide-platform
          port: 3001

---
# Grafana Metrics Dashboard
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames: ["metrics.local.ameide.io"]
  rules:
    - backendRefs:
        - name: observability-dashboard-grafana
          port: 3000
          namespace: ameide
```

### Keycloak Configuration

```yaml
# Chart values (in environments/local/infrastructure/keycloak.yaml)
extraEnvVars:
  - name: KC_PROXY
    value: "edge"  # Behind Gateway/proxy
  - name: KC_HOSTNAME
    value: "auth.local.ameide.io"
  - name: KC_HOSTNAME_PORT
    value: "8443"  # HTTPS port users access
  - name: KC_HTTP_RELATIVE_PATH
    value: "/"  # Root path for Keycloak
  - name: KC_HTTP_ENABLED
    value: "true"  # Allow HTTP internally (Gateway handles TLS)
```

**Note**: Envoy Gateway automatically forwards `X-Forwarded-Proto` and `X-Forwarded-Host` headers, which Keycloak uses when `KC_PROXY=edge` is set.

```bash
# Configure realm redirect URIs
kcadm.sh config credentials \
  --server https://auth.local.ameide.io \
  --realm master --user admin --password 'changeme'

# Get client ID
CID=$(kcadm.sh get clients -r ameide --fields id,clientId \
  | jq -r '.[] | select(.clientId=="ameide-app").id')

# Update redirect URIs and web origins
kcadm.sh update clients/$CID -r ameide \
  -s 'redirectUris=["https://local.ameide.io/*","https://platform.local.ameide.io/*","https://portal.local.ameide.io/*"]' \
  -s 'webOrigins=["https://local.ameide.io","https://platform.local.ameide.io","https://portal.local.ameide.io"]'

# Quick iteration option: use webOrigins=["+"] then lock down later
```

### DevContainer Configuration

```json
// .devcontainer/devcontainer.json
{
  // Mount Docker socket so k3d can work inside the container
  "mounts": [
    "source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind"
  ],
  
  // Use Docker from the host (not Docker-in-Docker)
  "features": {
    "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {}
  },
  
  "forwardPorts": [8080, 8443, 3000, 3001, 3002, 4000, 8000, 9001],
  
  "portsAttributes": {
    "8080": { "label": "Gateway HTTP", "onAutoForward": "notify" },
    "8443": { "label": "Gateway HTTPS", "onAutoForward": "notify" },
    "3000": { "label": "Main App (direct debug)", "onAutoForward": "silent" },
    "3001": { "label": "Canvas (direct debug)", "onAutoForward": "silent" },
    "3002": { "label": "Portal (direct debug)", "onAutoForward": "silent" },
    "4000": { "label": "Keycloak (direct debug)", "onAutoForward": "silent" },
    "8000": { "label": "API Gateway (direct debug)", "onAutoForward": "silent" },
    "9001": { "label": "Grafana (direct debug)", "onAutoForward": "silent" }
  }
}
```

### k3d Cluster Creation (DevContainer)

```bash
# .devcontainer/postCreateCommand.sh
k3d cluster create ameide \
  --api-port 45099 \
  --servers 1 \
  --agents 1 \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer" \
  --port "3000:3000@loadbalancer" \  # Keep for direct debug
  --port "3001:3001@loadbalancer" \
  --port "3002:3002@loadbalancer" \
  --port "4000:4000@loadbalancer" \
  --port "8000:8000@loadbalancer" \
  --registry-create docker.io/ameide \
  --k3s-arg "--disable=traefik@server:0" \
  --wait
```

## CORS Configuration for API

The API Gateway at `api.local.ameide.io` requires proper CORS headers for browser-based gRPC-Web clients. This is configured at the Gateway level:

```yaml
# HTTPRoute with CORS (in gateway/templates/httproute-api.yaml)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ameide-api
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames:
    - api.ameide.io
    - api.local.ameide.io
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        # Forward proxy headers
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: X-Forwarded-Proto
                value: https
        # Add CORS headers
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: Access-Control-Allow-Origin
                value: "*"  # Specify actual origins in production
              - name: Access-Control-Allow-Methods
                value: "GET, POST, PUT, DELETE, OPTIONS, PATCH"
              - name: Access-Control-Allow-Headers
                value: "Authorization, Content-Type, X-Requested-With, X-GRPC-Web"
              - name: Access-Control-Expose-Headers
                value: "grpc-status, grpc-message"
```

For production, use the `BackendTrafficPolicy` with specific allowed origins:

```yaml
# Advanced CORS policy (in gateway/templates/cors-policy.yaml)
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: api-cors-policy
  namespace: ameide
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: ameide-api
  cors:
    allowOrigins:
      - type: Exact
        value: "https://local.ameide.io"
      - type: Exact
        value: "https://platform.local.ameide.io"
      - type: Exact
        value: "https://portal.local.ameide.io"
    allowMethods: [GET, POST, PUT, DELETE, OPTIONS, PATCH, HEAD]
    allowHeaders: [Authorization, Content-Type, X-Requested-With, X-GRPC-Web]
    exposeHeaders: [grpc-status, grpc-message, grpc-status-details-bin]
    maxAge: 86400s
    allowCredentials: true
```

**Benefits of Gateway-level CORS:**
- No need to configure CORS in each backend service
- Consistent CORS policy across all APIs
- Handles preflight OPTIONS requests at the edge
- Better performance (no backend roundtrip for preflight)

## Access Points

### Production-Like Access (via Gateway)
**Preferred (HTTPS):**
- https://local.ameide.io - Main application
- https://auth.local.ameide.io - Keycloak authentication
- https://platform.local.ameide.io - Canvas platform
- https://portal.local.ameide.io - Portal application
- https://api.local.ameide.io - API Gateway
- https://metrics.local.ameide.io - Grafana dashboards

**Also Works (HTTP):**
- Same hosts on port 8080

### Direct Service Access (for debugging)
- http://localhost:3000 - Main app direct
- http://localhost:3001 - Canvas direct
- http://localhost:3002 - Portal direct
- http://localhost:4000 - Keycloak direct
- http://localhost:8000 - API Gateway direct
- http://localhost:9001 - Grafana direct (if port-forwarded)

## Testing & Verification

### Smoke Tests

```bash
# 1. DNS resolution
dscacheutil -q host -a name auth.local.ameide.io
# Should return: 127.0.0.1

# 2. Gateway routing to Keycloak
curl -sS https://auth.local.ameide.io/realms/master/.well-known/openid-configuration | jq .issuer
# Should return: "https://auth.local.ameide.io/realms/master"

# 3. Application health checks
curl -I https://platform.local.ameide.io/healthz
curl -I https://local.ameide.io/

# 4. Test with Host header (if ports not forwarded)
curl -sS -H "Host: auth.local.ameide.io" https://127.0.0.1/realms/master/.well-known/openid-configuration -k | jq .issuer
```

### Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| `auth.local.ameide.io` doesn't resolve | DNS not configured | Check dnsmasq and `/etc/resolver/local.ameide.io` |
| Port 8443 connection refused | Port not forwarded | Check VS Code Ports panel, ensure forwarding active |
| Certificate warnings | Self-signed cert | Expected for local dev, click "Advanced" → "Proceed" |
| Keycloak redirects to wrong URL | KC_HOSTNAME misconfigured | Verify KC_HOSTNAME and KC_HOSTNAME_PORT envs |
| Gateway not responding | Service not exposed | Check `kubectl -n argocd get svc` (Envoy Gateway control-plane/data-plane live in `argocd`) |
| VPN breaks resolution | `.test` intercepted | Disconnect VPN or use IP-based nip.io domains |

## Optional Enhancements

### Pretty URLs (ports 80/443)

If you want clean URLs without port numbers:

#### Option 1: Caddy Reverse Proxy
```bash
# Install Caddy
brew install caddy

# Caddyfile
cat > Caddyfile <<EOF
*.local.ameide.io {
  tls /path/to/ameide-local.crt /path/to/ameide-local.key
  reverse_proxy localhost:443
}
EOF

# Run with sudo (port 443)
sudo caddy run
```

#### Option 2: macOS pf redirect
```bash
# /etc/pf.anchors/ameide
rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 80 -> 127.0.0.1 port 8080
rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 443 -> 127.0.0.1 port 8443

# Enable
sudo pfctl -ef /etc/pf.anchors/ameide
```

### Makefile Automation

```makefile
# Makefile
.PHONY: setup up down trust-certs doctor clean

# Version pinning for stability
GATEWAY_API_VERSION := v1.2.1
ENVOY_GATEWAY_VERSION := v1.2.2

setup: ## One-time setup (DNS, certs)
	@echo "Setting up DNS and certificates..."
	@./scripts/setup-dns.sh
	@./scripts/setup-certs.sh

up: ## Start cluster and platform
	@echo "Creating k3d cluster..."
	@k3d cluster create ameide --config k3d-config.yaml

	@echo "Bootstrapping GitOps (installs Gateway API, Envoy Gateway, cert-manager, platform)..."
	@./bootstrap/bootstrap.sh --config bootstrap/configs/local.yaml --install-argo --apply-root-apps --wait-tiers

down: ## Stop cluster
	@k3d cluster delete ameide

trust-certs: ## Check cert-manager certificates
	@kubectl get certificate -n ameide-local
	@kubectl get secret ameide-wildcard-tls -n ameide-local

doctor: ## Check prerequisites
	@./scripts/doctor.sh
	@echo "Checking Gateway API..."
	@kubectl get gatewayclass 2>/dev/null || echo "⚠️  Gateway API not installed"
	@kubectl get crd | grep gateway.networking.k8s.io | wc -l | xargs -I {} test {} -gt 0 || echo "⚠️  Gateway CRDs missing"

clean: ## Remove all local setup
	@k3d cluster delete ameide
	@sudo rm -f /etc/resolver/local.ameide.io
	@sudo rm -f "$$(brew --prefix)/etc/dnsmasq.d/ameide.conf"
	@sudo brew services restart dnsmasq
```

## Rollback Plan

Complete cleanup if needed:

```bash
# Delete cluster
k3d cluster delete ameide

# Remove DNS configuration
sudo rm -f /etc/resolver/local.ameide.io
sudo rm -f "$(brew --prefix)/etc/dnsmasq.d/ameide.conf"
sudo brew services restart dnsmasq

# Remove certificates (optional)
rm -f ameide-local.crt ameide-local.key
```

## Critical Setup Requirements

### Gateway API & Envoy Gateway
- **Gateway API CRDs**: Required for HTTPRoute, GatewayClass resources
- **Envoy Gateway**: Implements the Gateway API specification
- **Version Pinning**: Pin versions in production scripts for stability
- **Idempotent Install**: Check existence before applying to avoid errors

### Namespace Configuration
- **Label Required**: `gateway-access=allowed` for allowedRoutes selector
- **Apply with `--overwrite`**: Ensures label is always present

### DevContainer Docker Access
- **Docker Socket Mount**: Required for k3d to create clusters
- **docker-outside-of-docker**: Better than Docker-in-Docker for performance
- **Not docker-in-docker**: Avoids nested virtualization issues

## Implementation Status

### ✅ Phase 1: Research & Planning (Completed)
- Researched industry best practices
- Identified `.local` mDNS conflicts
- Determined `.test` as optimal domain choice
- Documented port 8080/8443 standard
- Added critical setup requirements (Gateway API, namespace labels, Docker socket)

### ✅ Phase 2: Core Implementation (Completed)
- ✅ Switched all references to `.test` domains
- ✅ Updated Gateway configuration for port 8080/8443
- ✅ HTTPRoutes configured for all services
- ✅ k3d cluster creation script in postCreateCommand.sh
- ✅ DevContainer port forwarding configured
- ✅ cert-manager replaces mkcert for TLS management
- ✅ Self-signed ClusterIssuer configured
- ✅ Wildcard certificate (ameide-wildcard-tls) auto-generated

### ✅ Phase 3: Enhanced Setup (Completed)
- ✅ Created manage-host-domains.sh for dnsmasq setup
- ✅ Automated TLS via cert-manager (no mkcert needed)
- ✅ Helmfile manages deployments
- ✅ Documentation updated

### 🔮 Phase 4: Polish (Future)
- [ ] Add health check dashboard
- [ ] Create troubleshooting script
- [ ] Add performance monitoring
- [ ] Document production migration path

## Files Updated (All Completed)

### Core Configuration
- ✅ `backlog/109-local-development-domain-access.md` - This document (updated)
- ✅ `gitops/ameide-gitops/sources/values/env/local/apps/platform/gateway.yaml` - Gateway listeners configured
- ✅ `infra/kubernetes/charts/platform/cert-manager/` - cert-manager configuration
- ✅ HTTPRoutes configured via Helmfile deployments

### DevContainer Setup
- ✅ `.devcontainer/postCreateCommand.sh` - k3d cluster creation with .test domains
- ✅ `.devcontainer/devcontainer.json` - Port forwarding configured
- ✅ `.devcontainer/manage-host-domains.sh` - dnsmasq setup for macOS host

### Documentation
- ✅ `.devcontainer/README.md` - Updated with .test domains and cert-manager
- ✅ `README.md` - Quick start section updated
- ✅ `infra/kubernetes/README.md` - Deployment guide updated

## References

- [RFC 6761](https://datatracker.ietf.org/doc/html/rfc6761) - Special-Use Domain Names
- [cert-manager](https://cert-manager.io/) - Kubernetes native certificate management
- [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) - DNS forwarder
- [k3d](https://k3d.io/) - k3s in Docker
- [Gateway API](https://gateway-api.sigs.k8s.io/) - Kubernetes Gateway API
- [Envoy Gateway](https://gateway.envoyproxy.io/) - Gateway API implementation
