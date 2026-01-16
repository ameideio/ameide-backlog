> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.
>
> Update (2026-01): Keycloak is operator-managed (Keycloak CR) and exposed via Gateway HTTPRoute (`auth.<env>.ameide.io` → `Service/keycloak:8080`). Some older k3d/localhost `:4000` notes remain as historical context.

# Keycloak Configuration and Setup

## Overview
Keycloak is deployed as the identity and access management solution for the AMEIDE platform, providing OAuth2/OIDC authentication and authorization.

## Current Configuration

### Deployment Details
- **Helm Chart**: bitnami/keycloak v17.3.6
- **Keycloak Version**: 22.0.5
- **Namespace**: ameide
- **Service Type**: LoadBalancer
- **External Port**: 4000
- **Internal Port**: 8080

### Database Configuration
- **Type**: PostgreSQL (CloudNativePG)
- **Host**: postgres-keycloak-rw.ameide.svc.cluster.local
- **Database**: keycloak
- **User**: keycloak

### Access Points
- **Admin Console**: http://localhost:4000
- **OIDC Discovery**: http://localhost:4000/realms/ameide/.well-known/openid-configuration
- **Token Endpoint**: http://localhost:4000/realms/ameide/protocol/openid-connect/token
- **Internal Service URL**: http://keycloak.ameide.svc.cluster.local:8080

### Realms Configuration

#### Master Realm
- **Purpose**: Keycloak administration
- **Admin User**: admin
- **Admin Password**: changeme
- **Client**: admin-cli (for API access)

#### Ameide Realm
- **Purpose**: Application authentication
- **Configured Clients**:
  - `web-app`: Main application client
  - `account`: User account management
  - `account-console`: Account management UI
  - `admin-cli`: Administrative CLI
  - `broker`: Identity brokering
  - `realm-management`: Realm administration
  - `security-admin-console`: Security administration

### Test Credentials
- **Standard User**: user@ameide.io / password123
- **Admin User**: admin@ameide.io / password123
- **Test User**: testuser@ameide.local / testpass
- **Test Client**: web-app (confidential client with secret)

## Known Issues and Resolutions

### 1. Liveness Probe Configuration Issue (FIXED)
**Problem**: StatefulSet fails to create pods due to invalid liveness probe configuration with both `httpGet` and `tcpSocket` handlers.

**Root Cause**: The Bitnami Keycloak Helm chart v17.3.6 can merge probe configurations incorrectly during upgrades if the probe type is not explicitly specified in values.

**Error Message**:
```
Pod "keycloak-0" is invalid: spec.containers[0].livenessProbe.tcpSocket: 
Forbidden: may not specify more than 1 handler type
```

**Prevention (IMPLEMENTED)**:
- Explicitly specified `httpGet` probe type in `/workspace/infra/kubernetes/values/infrastructure/keycloak.yaml`
- Created validation script at `/workspace/infra/kubernetes/scripts/validate-keycloak.sh`
- Probes now explicitly define handler type to prevent merge conflicts

**Resolution if it occurs**:
1. Run validation script: `/workspace/infra/kubernetes/scripts/validate-keycloak.sh`
2. Delete the problematic StatefulSet: `kubectl delete statefulset keycloak -n ameide`
3. Reinstall with correct configuration using Helm

### 2. Stuck Helm Release
**Problem**: Helm release stuck in `pending-upgrade` or `pending-rollback` state.

**Resolution**:
```bash
# Check release status
helm status keycloak -n ameide

# Rollback to previous version
helm rollback keycloak -n ameide

# Verify status
helm list -n ameide --pending
```

### 3. Authentication Flow Configuration Issue (FIXED)
**Problem**: Keycloak authentication flow references non-existent "organization" authenticator.

**Error Message**:
```
Unable to find factory for AuthenticatorFactory: organization
```

**Root Cause**: Imported realm configuration included custom authentication flows with unavailable authenticators.

**Resolution**: 
- Reset browser flow to default via Admin API
- Updated client from public to confidential with secret
- Fixed authentication flow to use standard username/password form

## Testing Authentication

### 1. Obtain Admin Token (Master Realm)
```bash
curl -X POST http://172.20.0.3:4000/realms/master/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin" \
  -d "password=changeme" \
  -d "grant_type=password" \
  -d "client_id=admin-cli"
```

### 2. Obtain User Token (Ameide Realm)
```bash
# Test with standard user
curl -X POST http://localhost:4000/realms/ameide/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=testuser" \
  -d "password=testpass" \
  -d "grant_type=password" \
  -d "client_id=web-app"

# Test with admin user
curl -X POST http://localhost:4000/realms/ameide/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin" \
  -d "password=adminpass" \
  -d "grant_type=password" \
  -d "client_id=web-app"
```

### 3. Create New User via API
```bash
# Get admin token
TOKEN=$(curl -s -X POST http://172.20.0.3:4000/realms/master/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin" \
  -d "password=changeme" \
  -d "grant_type=password" \
  -d "client_id=admin-cli" | jq -r '.access_token')

# Create user
curl -X POST http://172.20.0.3:4000/admin/realms/ameide/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "newuser",
    "enabled": true,
    "emailVerified": true,
    "credentials": [{
      "type": "password",
      "value": "password123",
      "temporary": false
    }]
  }'
```

## Monitoring and Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n ameide | grep keycloak
kubectl logs keycloak-0 -n ameide --tail=50
```

### Check Events
```bash
kubectl get events -n ameide --field-selector involvedObject.name=keycloak-0
kubectl describe statefulset keycloak -n ameide
```

### Verify Service Endpoints
```bash
# Check service
kubectl get svc keycloak -n ameide

# Test connectivity
curl -s http://172.20.0.3:4000/realms/ameide/.well-known/openid-configuration
```

## Configuration Files

### Main Values
- `/workspace/infra/kubernetes/values/infrastructure/keycloak.yaml`
- Contains base configuration including database settings, probes, and realm import

### Local Environment Overrides
- `/workspace/infra/kubernetes/environments/local/infrastructure/keycloak.yaml`
- Overrides service type to LoadBalancer for local development

### Realm Configuration
- Mounted from ConfigMap: `keycloak-realm-config`
- Path: `/opt/bitnami/keycloak/data/import/ameide-realm.json`
- Auto-imported on startup with `KEYCLOAK_EXTRA_ARGS=--import-realm`

## NextAuth Integration

### Configuration for www-ameide-canvas

The frontend applications use NextAuth.js with the Keycloak provider:

#### Environment Variables (Updated for Production-Ready Setup)
```yaml
AUTH_URL: http://localhost:3001
AUTH_SECRET: development-secret-change-in-production
AUTH_TRUST_HOST: true  # For development
KEYCLOAK_CLIENT_ID: web-app
KEYCLOAK_CLIENT_SECRET: development-client-secret-change-in-production
# Dual URL configuration for internal vs public access
KEYCLOAK_ISSUER: http://keycloak:8080/realms/ameide  # Internal server-side
KEYCLOAK_ISSUER_PUBLIC: http://localhost:4000/realms/ameide  # Browser OAuth
```

#### Authentication Flow
1. User visits protected page
2. Middleware redirects to `/login` (custom login page)
3. User clicks "Sign in with SSO" button
4. NextAuth initiates OAuth flow with Keycloak
5. Redirected to Keycloak login page at port 4000
6. User enters credentials (e.g., user@ameide.io / password123)
7. After successful auth, redirected back with session

#### Testing Authentication
```bash
# Check available providers
curl http://localhost:3001/api/auth/providers

# Should return:
{
  "keycloak": {
    "id": "keycloak",
    "name": "Keycloak",
    "type": "oidc",
    "signinUrl": "http://localhost:3001/api/auth/signin/keycloak",
    "callbackUrl": "http://localhost:3001/api/auth/callback/keycloak"
  }
}
```

## Current OAuth Integration Status

### ✅ Completed (2025-08-08)
- ✅ Keycloak deployed and accessible at port 4000
- ✅ Test users configured in realm (user@ameide.io, admin@ameide.io, testuser@ameide.local)
- ✅ Client updated from public to confidential with secret
- ✅ SSO button added to login page alongside email/password option
- ✅ Dual URL configuration for internal/external access
- ✅ OAuth flow initiates successfully and redirects to Keycloak
- ✅ Keycloak login form displays correctly
- ✅ Fixed authentication flow "organization" authenticator issue
- ✅ Fixed OAuth callback configuration error
- ✅ NextAuth properly configured with separate public/internal URLs
- ✅ Session creation working after successful authentication
- ✅ User data properly extracted from Keycloak (ID, email, name)

### Working Configuration
The final working configuration uses:
- **Public issuer** as base for browser-facing discovery
- **Public URL** for authorization endpoint (browser redirect)  
- **Internal URL** for token and userinfo endpoints (server-side calls)
- **Profile mapping** to extract user data from Keycloak claims

### Helm Configuration
All OAuth configuration is now fully declarative in Helm:

#### www-ameide-canvas values (`/workspace/infra/kubernetes/values/platform/www-ameide-canvas.yaml`)
```yaml
auth:
  secret: "development-secret-change-in-production"

keycloak:
  clientId: "web-app"
  clientSecret: "development-client-secret-change-in-production"
  issuerInternal: "http://keycloak:8080/realms/ameide"  # Server-side calls
  issuerPublic: "http://localhost:4000/realms/ameide"   # Browser redirects
```

#### Keycloak Realm ConfigMap (`/workspace/infra/kubernetes/charts/platform/keycloak-realm-configmap.yaml`)
- Defines `simple-browser` authentication flow to avoid "organization" authenticator issues
- Configures web-app client as confidential with secret
- Sets up test users with credentials
- Includes all redirect URIs for ports 3000-3003

#### Secret Template (`/workspace/infra/kubernetes/charts/platform/www-ameide-canvas/templates/secret.yaml`)
```yaml
data:
  KEYCLOAK_CLIENT_SECRET: {{ .Values.keycloak.clientSecret | b64enc | quote }}
  AUTH_SECRET: {{ .Values.auth.secret | b64enc | quote }}
```

#### ConfigMap Template (`/workspace/infra/kubernetes/charts/platform/www-ameide-canvas/templates/configmap.yaml`)
```yaml
data:
  KEYCLOAK_CLIENT_ID: "{{ .Values.keycloak.clientId }}"
  KEYCLOAK_ISSUER: "{{ .Values.keycloak.issuerInternal | default .Values.keycloak.issuer }}"
  KEYCLOAK_ISSUER_PUBLIC: "{{ .Values.keycloak.issuerPublic | default .Values.keycloak.issuer }}"
```

### Implementation Notes

#### Client Configuration via Admin API
```bash
# Update client to confidential
TOKEN=$(kubectl exec -n ameide statefulset/keycloak -- curl -s -X POST \
  http://localhost:8080/realms/master/protocol/openid-connect/token \
  -d "client_id=admin-cli" -d "username=admin" -d "password=changeme" \
  -d "grant_type=password" | grep -o '"access_token":"[^"]*' | cut -d'"' -f4)

kubectl exec -n ameide statefulset/keycloak -- curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:8080/admin/realms/ameide/clients/<CLIENT_ID> \
  -d '{
    "publicClient": false,
    "secret": "development-client-secret-change-in-production"
  }'
```

#### Playwright E2E Tests
Tests created to verify OAuth flow:
- `/workspace/services/www_ameide_canvas/tests/e2e/sso-button.test.ts` - Verifies SSO button exists
- `/workspace/services/www_ameide_canvas/tests/e2e/keycloak-debug.test.ts` - Debug Keycloak redirect
- `/workspace/services/www_ameide_canvas/tests/e2e/full-oauth-login.test.ts` - Complete login flow
- `/workspace/services/www_ameide_canvas/tests/e2e/oauth-session.test.ts` - Verifies session creation

#### Application Code Changes
- **Login Page** (`/workspace/services/www_ameide_canvas/app/(auth)/login/page.tsx`): Added SSO button
- **Auth Config** (`/workspace/services/www_ameide_canvas/app/(auth)/auth.ts`): Configured Keycloak provider with dual URLs
- **API Route** (`/workspace/services/www_ameide_canvas/app/(auth)/api/auth/[...nextauth]/route.ts`): Exports NextAuth handlers

## ⚠️ Critical Configuration Limitation

**The current dual-URL OAuth configuration only works in k3d/Docker Desktop** due to their special networking where `localhost:4000` is accessible from inside pods via LoadBalancer services.

### The Issuer URL Problem
- Keycloak puts the accessed URL in the JWT's `iss` claim
- NextAuth must validate this matches the configured issuer
- If browser uses `localhost:4000` but pod uses `keycloak:8080`, tokens won't validate
- See `/workspace/infra/kubernetes/charts/platform/www-ameide-canvas/IMPORTANT-OAUTH-NOTES.md` for details

## Production Deployment Checklist

For production, you MUST solve the issuer URL problem first:

### Option A: Single URL (Recommended)
Configure DNS/ingress so `https://auth.yourdomain.com` is accessible from both browser and pods:
```yaml
keycloak:
  issuer: "https://auth.yourdomain.com/realms/ameide"
  # issuerPublic not needed - same URL works everywhere
```

### Option B: Configure Keycloak Frontend URL
Force Keycloak to use a consistent issuer:
```yaml
extraEnvVars:
  - name: KC_HOSTNAME_URL
    value: "https://auth.yourdomain.com"
  - name: KC_HOSTNAME_STRICT
    value: "false"
```

### Then Complete Setup:

1. **Generate Secure Secrets**:
   ```bash
   # Generate AUTH_SECRET
   openssl rand -base64 32
   
   # Generate Keycloak client secret in Keycloak Admin Console
   ```

2. **Update Helm Values**:
   ```yaml
   # production/www-ameide-canvas.yaml
   auth:
     secret: "<generated-secret>"
   
   keycloak:
     clientSecret: "<keycloak-generated-secret>"
     issuerPublic: "https://auth.yourdomain.com/realms/ameide"
   ```

3. **Update Keycloak Configuration**:
   - Add production redirect URIs: `https://app.yourdomain.com/*`
   - Add production web origins: `https://app.yourdomain.com`
   - Enable HTTPS required
   - Set production mode: true

4. **Use Secret Management**:
   - Consider External Secrets Operator
   - Or Sealed Secrets for GitOps
   - Never commit real secrets to git

## Future Improvements

1. **Enhanced Security**:
   - Enable HTTPS/TLS
   - Set `production: true` in Helm values
   - Configure proper hostnames
   - Enable strict hostname checking

2. **High Availability**:
   - Increase replica count
   - Configure Infinispan clustering
   - Set up session replication

3. **Security Hardening**:
   - Change default admin password
   - Configure password policies
   - Enable MFA/2FA
   - Set up proper CORS policies

4. **Integration**:
   - Configure LDAP/AD integration
   - Set up social login providers
   - Implement fine-grained authorization

5. **Monitoring**:
   - Enable metrics endpoint
   - Configure Prometheus scraping
   - Set up alerts for authentication failures

## Related Documentation
- [Bitnami Keycloak Chart](https://github.com/bitnami/charts/tree/main/bitnami/keycloak)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [OIDC Specification](https://openid.net/connect/)

---
