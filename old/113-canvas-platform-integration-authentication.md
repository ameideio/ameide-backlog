# Platform Authentication Integration

## Executive Summary

This document details the authentication architecture for the AMEIDE platform, with a unified Keycloak SSO implementation across the authenticated platform service.

## Target Architecture

### Service Structure

```
services/
├── www-ameide/              # Public portal (no authentication)
│   └── Landing page, documentation, marketing
│
└── www_ameide_platform/     # Authenticated platform (Keycloak SSO)
    └── Chat, canvas, BPMN editor, all platform features
```

### Authentication Flow

```mermaid
graph TD
    A[User] -->|Visits| B[www-ameide - Public Portal]
    B -->|Clicks Login| C[Redirect to Platform]
    C -->|No Session| D[Keycloak Login]
    D -->|Authenticates| E[Keycloak]
    E -->|OIDC Callback| F[Platform Auth Handler]
    F -->|Store Tokens| G[Redis Session Store]
    F -->|Create Session| H[NextAuth Session]
    H -->|Access| I[Protected Platform Features]
    
    J[Token Refresh] -->|Before Expiry| K[Refresh Handler]
    K -->|Distributed Lock| L[Prevent Race Conditions]
    L -->|Update Tokens| G
```

## Implementation Details

### www_ameide_platform Service

**Complete Keycloak Integration Features:**
- ✅ NextAuth.js v5 with Keycloak OIDC provider
- ✅ PKCE, state, and nonce security checks
- ✅ Redis-based session storage with distributed locking
- ✅ Automatic token refresh with race condition prevention
- ✅ Role extraction from JWT claims (admin detection)
- ✅ Comprehensive auth logging and metrics
- ✅ Middleware-based route protection
- ✅ Backchannel logout support via SID mapping
- ✅ CSRF protection with secure cookies
- ✅ Multi-tab session synchronization

**File Structure:**
```
services/www_ameide_platform/
├── app/(auth)/
│   ├── auth.ts              # Main NextAuth configuration
│   ├── auth.config.ts       # Base auth config
│   └── login/
│       └── page.tsx         # Auto-redirect to Keycloak
├── lib/
│   ├── keycloak.ts         # Token refresh & role extraction
│   ├── session-store.ts    # Redis session management
│   ├── auth-logger.ts      # Logging utilities
│   └── cache/
│       └── redis.ts         # Redis implementation
├── middleware.ts            # Route protection
└── tests/e2e/
    └── auth-*.test.ts      # Comprehensive auth tests
```

### www-ameide Service (Public Portal)

**No Authentication Required:**
- Public landing page
- Documentation
- Marketing content
- Links to authenticated platform

**File Structure:**
```
services/www-ameide/
├── app/
│   ├── page.tsx            # Public landing page
│   ├── docs/               # Public documentation
│   └── api/
│       └── health/         # Health endpoints
└── (no auth files)         # No authentication needed
```

## Configuration

### Keycloak Setup

```javascript
// Realm: ameide
// Client: platform-app
{
  "clientId": "platform-app",
  "protocol": "openid-connect",
  "publicClient": false,
  "standardFlowEnabled": true,
  "attributes": {
    "pkce.code.challenge.method": "S256",
    "access.token.lifespan": "300",
    "refresh.token.max.reuse": "0"
  },
  "redirectUris": [
    "http://platform.dev.ameide.io:8080/*"
  ]
}
```

### Environment Variables

```bash
# Platform service configuration
AUTH_URL=http://platform.dev.ameide.io:8080
AUTH_SECRET=development-secret-at-least-32-chars
AUTH_TRUST_HOST=true

# Keycloak
AUTH_KEYCLOAK_ISSUER=http://auth.dev.ameide.io:8080/realms/ameide
AUTH_KEYCLOAK_ID=platform-app
AUTH_KEYCLOAK_SECRET=changeme

# Redis sessions (CoreDNS handles resolution)
REDIS_URL=redis://redis-node-1.redis-headless:6379/0

# Database for shadow users (CoreDNS handles resolution)
POSTGRES_URL=postgres://dbuser:dbpassword@postgres-platform-rw:5432/platform

# Session settings
AUTH_SESSION_MAX_AGE=86400
AUTH_SESSION_UPDATE_AGE=3600
```

### Kubernetes Configuration

```yaml
# infra/kubernetes/values/platform/www-ameide-platform.yaml
service:
  name: www-ameide-platform
  port: 3001

auth:
  enabled: true
  
keycloak:
  issuer: "http://auth.dev.ameide.io:8080/realms/ameide"
  clientId: "platform-app"
  clientSecret: "changeme"

redis:
  url: "redis://redis-node-1.redis-headless:6379/0"

# Database for shadow user storage (CoreDNS resolves service names)
database:
  url: "postgres://dbuser:dbpassword@postgres-platform-rw:5432/platform"
```

```yaml
# infra/kubernetes/values/platform/www-ameide.yaml
service:
  name: www-ameide
  port: 3000

# No auth configuration - public service
```

## Security Features

### Token Storage
- Access tokens stored in Redis (never in cookies)
- Refresh tokens in Redis with TTL
- Minimal session pointer in httpOnly cookie

### Session Security
- Distributed lock for token refresh
- Session fixation protection
- Automatic session extension
- Idle timeout handling

### Network Security
- CSRF protection with double-submit cookies
- SameSite=lax cookie attributes
- Origin validation in callbacks
- Secure redirect validation

## Testing

### Integration Tests
```typescript
describe('Platform Authentication', () => {
  test('redirects to Keycloak when not authenticated');
  test('allows access after successful login');
  test('refreshes token before expiry');
  test('handles concurrent refresh attempts');
  test('validates admin roles');
  test('single sign-out works');
});
```

### Security Tests
- CSRF token validation
- XSS prevention
- Token expiry enforcement
- Secure redirect handling

## Deployment

### HTTPRoute Configuration

```yaml
# Platform (authenticated)
hostnames:
  - platform.dev.ameide.io
  - app.dev.ameide.io

# Public portal
hostnames:
  - dev.ameide.io
  - www.dev.ameide.io
```

### Health Checks

Both services expose health endpoints:
- `www-ameide`: `/api/health` (public)
- `www_ameide_platform`: `/api/health` (public endpoint, auth not required)

## Success Metrics

- **Authentication Coverage**: 100% of platform features protected
- **Session Reliability**: 99.9% uptime
- **Token Refresh Success**: >99% success rate
- **Auth Latency**: <100ms p99
- **Security**: Zero auth-related breaches

## Implementation Status

### ✅ Completed - Phase 1: Authentication Infrastructure
- Keycloak OIDC integration with NextAuth.js v5
- Token storage and refresh with Redis
- Middleware-based route protection
- CSRF and security features
- Health check endpoints
- Comprehensive test coverage

### ✅ Completed - Phase 2: Shadow User Pattern
- Database schema updated with Keycloak integration fields
- User upsert on login via `upsertUserFromKeycloak()`
- Local user records synced from Keycloak profile
- Role-based authorization helpers in `lib/auth/authorization.ts`
- UserContext provider for centralized user state
- Removed legacy guest/password authentication code

### ✅ Completed - Phase 3: Test Coverage
- Replaced all legacy authentication tests with Keycloak SSO tests
- Created comprehensive shadow user pattern integration tests
- Added role-based access control test suite
- Removed all guest user and email/password test references
- Documented shadow user pattern benefits and implementation

### ✅ Completed - Phase 4: Database Integration & Fixes (August 2025)
- Fixed missing `POSTGRES_URL` environment variable configuration
- Added database connection to Kubernetes ConfigMap and values
- Fixed role-based entitlements (`getUserEntitlements` function)
- Removed all references to `user.type` in favor of role-based access
- Successfully tested authentication flow with database persistence
- Verified shadow user creation with both `id` and `kcSub` fields

## Implemented Shadow User Pattern

### Architecture Decision
Use the **shadow user pattern** - let Keycloak own identity while maintaining minimal local user records for referential integrity.

**Keycloak owns:**
- Authentication (login, MFA, sessions, password resets)
- Core profile (name, email, picture)
- Roles and group assignments
- Token issuance with claims

**App database owns:**
- Shadow user table keyed by Keycloak `sub`
- App-specific data (preferences, quotas)
- Foreign keys for domain tables (messages, threadss)
- Audit and analytics data

### Implementation Steps

#### 1. Database Migration
```sql
-- Update User table for shadow records
ALTER TABLE "User" 
  ADD COLUMN kc_sub VARCHAR(255) UNIQUE NOT NULL,
  DROP COLUMN password,
  ADD COLUMN name VARCHAR(255),
  ADD COLUMN avatar_url TEXT,
  ADD COLUMN created_at TIMESTAMP DEFAULT NOW(),
  ADD COLUMN updated_at TIMESTAMP DEFAULT NOW();
-- Don't store roles - read from token
```

#### 2. User Sync on Login
```typescript
// In auth.ts callbacks.signIn
const upsertUser = await db.user.upsert({
  where: { kc_sub: profile.sub },
  create: {
    id: generateUUID(),
    kc_sub: profile.sub,
    email: profile.email,
    name: profile.name,
    avatar_url: profile.picture,
  },
  update: {
    email: profile.email,
    name: profile.name,
    avatar_url: profile.picture,
    updated_at: new Date(),
  },
});

// Store local ID in token for FKs
token.userId = upsertUser.id;
token.kc_sub = profile.sub;
```

#### 3. Authorization Helpers
```typescript
// Read roles from JWT, not database
interface SessionUser {
  id: string;        // Local DB ID for FKs
  kc_sub: string;    // Keycloak subject
  email: string;
  roles: string[];   // From JWT claims
}

function hasRole(user: SessionUser, role: string): boolean {
  return user.roles.includes(role);
}
```

#### 4. Cleanup Tasks
- Remove guest user system (`createGuestUser`, guest routes)
- Remove password-based authentication
- Remove hardcoded `UserType = 'keycloak'`
- Update all FK references to use local user.id

#### 5. Migration Path
1. Add `kc_sub` column (nullable initially)
2. Populate on next login for existing users
3. Make `kc_sub` required once populated
4. Remove old auth code

## Benefits
- **Referential integrity**: Local FKs work properly
- **Stable identity**: `kc_sub` never changes
- **Simple sync**: Upsert on login only
- **Clean separation**: Identity vs application data
- **Performance**: No external calls for queries

## Production Hardening

### Secret Management Patterns

#### Pattern 1: Full Connection String in Secret
```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "chart.fullname" . }}-db
stringData:
  POSTGRES_URL: {{ .Values.database.url | quote }}
  
# templates/deployment.yaml
envFrom:
  - secretRef:
      name: {{ include "chart.fullname" . }}-db
```

#### Pattern 2: External Secrets Operator (Recommended)
```yaml
# templates/externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "chart.fullname" . }}-db
spec:
  secretStoreRef:
    name: vault-backend
  target:
    name: {{ include "chart.fullname" . }}-db
  data:
    - secretKey: POSTGRES_URL
      remoteRef:
        key: database/credentials
        property: connection_string
```

#### Pattern 3: Assembled from Components
```yaml
# templates/secret.yaml
stringData:
  DB_HOST: postgres-platform-rw
  DB_NAME: platform
  DB_USER: {{ .Values.database.user }}
  DB_PASSWORD: {{ .Values.database.password }}

# App constructs: postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:5432/${DB_NAME}
```

### Helm Chart Improvements

#### Auto-rollout on Config Changes
```yaml
# templates/deployment.yaml
metadata:
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
```

#### Health Probes with Database Check
```yaml
readinessProbe:
  httpGet:
    path: /api/health
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /api/health
    port: http
  initialDelaySeconds: 15
  periodSeconds: 10
```

#### Values Schema Validation
```json
// values.schema.json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "database": {
      "type": "object",
      "required": ["url"],
      "properties": {
        "url": {
          "type": "string",
          "pattern": "^postgres://.*"
        }
      }
    }
  },
  "required": ["database"]
}
```

### Security Best Practices

1. **Never use `${VAR}` syntax in ConfigMaps** - Kubernetes doesn't expand them
2. **Quote all values**: Use `| quote` filter to prevent YAML injection
3. **Use defaults**: `| default ""` prevents nil pointer errors
4. **Separate concerns**: Secrets in Secret objects, config in ConfigMaps
5. **Validate health**: Ensure `/api/health` checks database connectivity

### Testing Commands

```bash
# Verify ConfigMap has the right values
kubectl get cm www-ameide-platform-config -n ameide -o yaml | grep POSTGRES_URL

# Check environment in pod
kubectl exec -n ameide deploy/www-ameide-platform -- printenv | grep -E 'POSTGRES|REDIS'

# Test database connectivity from cluster
kubectl run psql --rm -i -t --image=postgres:16 --restart=Never -- \
  psql "postgres://dbuser:dbpassword@postgres-platform-rw:5432/platform" -c 'SELECT 1;'

# Check deployment annotations
kubectl describe deploy/www-ameide-platform -n ameide | grep checksum/

# Verify probes are working
kubectl get deploy/www-ameide-platform -n ameide -o yaml | grep -A10 Probe
```

## Future Enhancements

1. **User deletion sync**: Subscribe to Keycloak admin events
2. **Profile management UI**: Settings page for preferences
3. **Fine-grained permissions**: Resource-level ACLs if needed
4. **Session analytics**: Track user behavior and metrics
5. **Enhanced health checks**: Include database, Redis, and Keycloak connectivity