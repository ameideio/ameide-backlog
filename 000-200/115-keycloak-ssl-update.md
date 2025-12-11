> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Keycloak SSL/TLS Configuration

**Status**: ✅ Mostly Implemented
**Created**: 2025-01-13
**Updated**: 2025-01-13
**Priority**: Medium (Certificate fix needed)
**Dependencies**: 
- [104-keycloak-config-v2.md](./104-keycloak-config-v2.md) - OAuth configuration
- [105-domain-strategy-ameide-io.md](./105-domain-strategy-ameide-io.md) - Domain strategy
- [106-cert-manager-letsencrypt.md](./106-cert-manager-letsencrypt.md) - Certificate management
- [114-ssl.md](./114-ssl.md) - SSL security implementation

## Executive Summary

Keycloak SSL/TLS is implemented with TLS termination at the Gateway layer. The system uses a split-endpoint strategy: HTTPS for browser-facing URLs and internal HTTP for server-side communication. One critical issue remains: the local certificate covers `*.ameide.io` instead of `*.dev.ameide.io`.

## Current State

### What's Implemented ✅

1. **Gateway TLS Termination**
   - HTTPS listener on port 443 (exposed as 8443 via k3d)
   - Internal services use HTTP (Keycloak on port 4000)
   - HTTPRoute for Keycloak already configured
   - Gateway handles all TLS complexity

2. **Split Endpoint Strategy**
   - Browser requests: `https://auth.dev.ameide.io` (HTTPS)
   - Server-side calls: `http://keycloak:4000` (internal HTTP)
   - Optimizes both security and cluster performance
   - NextAuth v5 configured with separate endpoints

3. **Authentication Working**
   - OAuth flows functional with PKCE + state + nonce
   - JWT session strategy with secure cookies
   - Cookie security properly configured per environment
   - Session persistence across page refreshes

### Critical Issue to Fix ⚠️

**Certificate DNS Mismatch**
- Certificate configured for: `*.ameide.io`
- Local development uses: `*.dev.ameide.io`
- Impact: Browser certificate errors for local development

## Architecture Overview

```mermaid
graph LR
    Browser -->|HTTPS:443| k3d
    k3d -->|port-forward| Gateway
    Gateway -->|HTTP:4000| Keycloak
    Gateway -->|HTTP:3001| Platform
    
    subgraph "k3d Port Mapping"
        k3d[k3d loadbalancer<br/>8443→443]
    end
    
    subgraph "TLS Termination"
        Gateway[Envoy Gateway<br/>listener: https (443)]
    end
    
    subgraph "Internal HTTP"
        Keycloak[Keycloak<br/>keycloak:4000]
        Platform[Platform<br/>www-ameide-platform:3001]
    end
```

## Current Implementation

### Local Development Configuration

**Gateway Configuration:**
```yaml
# gitops/ameide-gitops/sources/values/local/apps/platform/gateway.yaml
listeners:
  - name: https
    port: 443  # Internal port (k3d maps 8443→443)
    protocol: HTTPS
    hostname: "*.dev.ameide.io"
    tls:
      certificateRefs:
        - name: ameide-wildcard-tls
```

**Keycloak HTTPRoute (Already Exists):**
```yaml
# infra/kubernetes/charts/platform/keycloak/templates/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  parentRefs:
    - name: ameide
      sectionName: https  # References the 'https' listener
  hostnames:
    - auth.dev.ameide.io
  rules:
    - backendRefs:
        - name: keycloak
          port: 4000
```

**NextAuth Split Endpoint Configuration:**
```typescript
// services/www_ameide_platform/app/(auth)/auth.ts
const keycloakProvider = Keycloak({
  // Browser-facing URL (HTTPS)
  authorization: {
    url: "https://auth.dev.ameide.io/realms/ameide/protocol/openid-connect/auth",
  },
  // Server-side endpoints (internal HTTP)
  token: "http://keycloak:4000/realms/ameide/protocol/openid-connect/token",
  userinfo: "http://keycloak:4000/realms/ameide/protocol/openid-connect/userinfo",
  // ... other endpoints
});
```

### 2. Staging Environment

```yaml
# environments/staging/platform/keycloak.yaml
keycloak:
  issuer: "https://auth.test.ameide.io/realms/ameide"
  
extraEnvVars:
  - name: KC_HOSTNAME
    value: "auth.test.ameide.io"
  - name: KC_HOSTNAME_STRICT
    value: "true"
  - name: KC_PROXY
    value: "edge"  # TLS at Gateway
  - name: KC_HTTP_ENABLED
    value: "true"  # HTTP internally
```

### 3. Production Environment

```yaml
# environments/production/platform/keycloak.yaml
keycloak:
  issuer: "https://auth.ameide.io/realms/ameide"
  
extraEnvVars:
  - name: KC_HOSTNAME
    value: "auth.ameide.io"
  - name: KC_HOSTNAME_STRICT
    value: "true"
  - name: KC_PROXY
    value: "edge"
  - name: KC_HTTP_ENABLED
    value: "true"
```

## Required Fix: Certificate DNS Names

### The Issue
The local development certificate is misconfigured:

```yaml
# Current (WRONG) - infra/kubernetes/environments/local/platform/cert-manager.yaml
certificates:
  - name: ameide-wildcard-local
    secretName: ameide-wildcard-tls
    dnsNames:
      - "*.ameide.io"    # Should be *.dev.ameide.io
      - "ameide.io"      # Should be dev.ameide.io
      - "localhost"
```

### The Fix
```yaml
# Required change
certificates:
  - name: ameide-wildcard-local
    secretName: ameide-wildcard-tls
    dnsNames:
      - "*.dev.ameide.io"  # Correct for local development
      - "dev.ameide.io"     # Correct for local development
      - "localhost"
```

After applying this fix, regenerate the certificate:
```bash
# Delete the old certificate and secret
kubectl delete certificate ameide-wildcard-local -n ameide
kubectl delete secret ameide-wildcard-tls -n ameide

# Reapply cert-manager configuration
helmfile -e local sync --selector name=cert-manager

# Verify new certificate
kubectl get certificate -n ameide
kubectl describe certificate ameide-wildcard-local -n ameide
```

## Certificate Trust (Developer Experience)

### Option A: Install Self-Signed Certificate (Recommended)
```bash
# Extract certificate from cluster
kubectl get secret ameide-wildcard-tls -n ameide \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/ameide-test.crt

# macOS
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain /tmp/ameide-test.crt

# Linux
sudo cp /tmp/ameide-test.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Windows (PowerShell as Admin)
Import-Certificate -FilePath C:\tmp\ameide-test.crt \
  -CertStoreLocation Cert:\LocalMachine\Root
```

### Option B: Browser Exception (Quick)
1. Visit https://auth.dev.ameide.io
2. Accept certificate warning
3. Visit https://platform.dev.ameide.io
4. Accept certificate warning

## Validation

### Health Checks
```bash
# 1. Verify Keycloak HTTPS access
curl -k https://auth.dev.ameide.io/realms/ameide/.well-known/openid-configuration

# 2. Check issuer URL in response
curl -k https://auth.dev.ameide.io/realms/ameide/.well-known/openid-configuration \
  | jq -r .issuer
# Returns: https://auth.dev.ameide.io/realms/ameide

# 3. Verify certificate DNS names (after fix)
kubectl get certificate ameide-wildcard-local -n ameide -o yaml | grep -A 3 dnsNames
# Should show *.dev.ameide.io, not *.ameide.io

# 4. Test OAuth flow in browser
# - Visit https://platform.dev.ameide.io/login
# - Click "Sign in with Keycloak"
# - Should redirect to HTTPS Keycloak login
# - After auth, check cookie has Secure flag in DevTools
```

## Security Considerations

### 1. Mixed Content Prevention
- All OAuth URLs must use HTTPS in browser context
- Server-side can use HTTP for internal communication
- Gateway ensures consistent HTTPS for external access

### 2. Cookie Security Matrix

| Environment | Cookie Name | Secure | SameSite | Domain |
|------------|-------------|---------|----------|---------|
| Local Dev | `authjs.session-token` | Yes | Lax | None |
| Staging | `__Secure-authjs.session-token` | Yes | Lax | `.test.ameide.io` |
| Production | `__Secure-authjs.session-token` | Yes | Lax | `.ameide.io` |

### 3. CORS Configuration
```typescript
// Ensure CORS allows HTTPS origins
const corsOrigins = [
  'https://platform.dev.ameide.io',
  'https://auth.dev.ameide.io',
  'https://api.dev.ameide.io'
];
```

## NextAuth v5 Configuration

The current implementation uses a split-endpoint strategy that's already optimized:

```typescript
// services/www_ameide_platform/app/(auth)/auth.ts
const keycloakProvider = Keycloak({
  clientId: process.env.AUTH_KEYCLOAK_ID,
  clientSecret: process.env.AUTH_KEYCLOAK_SECRET,
  checks: ['state', 'pkce', 'nonce'],
  
  // Browser-facing endpoints (HTTPS)
  issuer: "https://auth.dev.ameide.io/realms/ameide",
  authorization: {
    url: "https://auth.dev.ameide.io/realms/ameide/protocol/openid-connect/auth",
  },
  
  // Server-side endpoints (internal HTTP for performance)
  token: "http://keycloak:4000/realms/ameide/protocol/openid-connect/token",
  userinfo: "http://keycloak:4000/realms/ameide/protocol/openid-connect/userinfo",
  wellKnown: "http://keycloak:4000/realms/ameide/.well-known/openid-configuration",
  
  httpOptions: {
    timeout: 10000,
    // Handle self-signed certs in development
    ...(process.env.NODE_ENV !== 'production' && {
      agent: new https.Agent({ rejectUnauthorized: false }),
    }),
  },
});
```

## Testing Checklist

### Prerequisites
- [x] Keycloak HTTPRoute exists and configured
- [x] Gateway HTTPS listener on port 443
- [x] NextAuth split-endpoint configuration
- [ ] Certificate DNS names fixed (*.dev.ameide.io)

### After Certificate Fix
- [ ] Certificate shows `*.dev.ameide.io` in DNS names
- [ ] Keycloak accessible via `https://auth.dev.ameide.io`
- [ ] Platform accessible via `https://platform.dev.ameide.io`
- [ ] OAuth flow completes successfully
- [ ] Cookies have Secure flag set
- [ ] No certificate warnings (after trust installation)
- [ ] Session persists across page refreshes

## Action Items

1. **Immediate**: Fix certificate DNS names configuration
   - Update `infra/kubernetes/environments/local/platform/cert-manager.yaml`
   - Change `*.ameide.io` → `*.dev.ameide.io`
   - Regenerate certificates

2. **Documentation**: Update developer onboarding
   - Add certificate trust installation steps
   - Document the split-endpoint strategy
   - Explain k3d port mapping (8443→443)

3. **Production Path**: 
   - Staging/Production already configured correctly
   - Use Let's Encrypt for production certificates
   - Keep split-endpoint strategy for optimal performance

## Troubleshooting Guide

### Common Issues

#### 1. "Invalid redirect_uri" Error
**Cause**: Keycloak expects HTTPS but receives HTTP callback
**Solution**: 
```yaml
# Update Keycloak client configuration
Valid Redirect URIs:
  - https://platform.dev.ameide.io/*
  - https://platform.dev.ameide.io/*
```

#### 2. Cookie Not Set
**Cause**: Secure cookie on HTTP connection
**Solution**: Ensure all URLs use HTTPS or adjust cookie security for development

#### 3. Certificate Warnings
**Cause**: Self-signed certificate not trusted
**Solution**: Install certificate in system trust store (see Phase 2)

#### 4. Mixed Content Blocked
**Cause**: HTTPS page loading HTTP resources
**Solution**: Update all resource URLs to use HTTPS

## Summary

### Current Status
- ✅ **TLS Termination**: Gateway handles HTTPS on port 443 (exposed as 8443)
- ✅ **Split Endpoints**: HTTPS for browser, HTTP for internal cluster
- ✅ **OAuth Working**: Full authentication flow with secure cookies
- ✅ **HTTPRoute**: Keycloak routing already configured
- ❌ **Certificate Issue**: DNS names show `*.ameide.io` instead of `*.dev.ameide.io`

### Only Action Required
Fix the certificate DNS names in `infra/kubernetes/environments/local/platform/cert-manager.yaml`:
- Change `*.ameide.io` → `*.dev.ameide.io`
- Regenerate the certificate
- Optionally install certificate in system trust store

The rest of the SSL/TLS configuration is already properly implemented and working.
