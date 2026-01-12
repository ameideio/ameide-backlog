> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Dynamic Tenant Resolution from Keycloak

**Status**: Completed (Phase 1) - See [331-tenant-resolution.md](./331-tenant-resolution.md)
**Priority**: High
**Area**: Authentication & Multi-tenancy
**Related**: [329-authz.md](./329-authz.md), [322-rbac.md](./322-rbac.md), [331-tenant-resolution.md](./331-tenant-resolution.md), [333-realms.md](./333-realms.md)

---

## ⚠️ ARCHITECTURE UPDATE (2025-10-30)

**This document is superseded by realm-per-tenant architecture.** See [backlog/333-realms.md](./333-realms.md).

**Current status**:
- ✅ **Phase 1 Complete**: JWT-based tenant resolution from Keycloak `tenantId` attribute (see [331-tenant-resolution.md](./331-tenant-resolution.md))
- ⚠️ **Architecture evolving**: Moving from single realm to realm-per-tenant

**Impact of realm-per-tenant**:
- Each organization will have its own Keycloak realm
- Tenant routing becomes **issuer-driven**: treat OIDC `iss` (issuer URL) as an opaque identifier and resolve `issuer → tenant_id` server-side (do not parse realm name out of `iss`)
- The `tenantId` user attribute approach described here becomes **redundant**
- Realm discovery will replace tenant resolution

**What remains valid**:
- Principle: No hardcoded tenant defaults anywhere
- Principle: Tenant/organization context must come from authenticated user
- Principle: Fail gracefully if tenant cannot be determined

**Migration path**:
1. Continue using `tenantId` attribute in single realm (current state)
2. Implement realm-per-tenant architecture (333-realms.md Week 1-2)
3. Switch to issuer-driven routing with `issuer → tenant_id` mapping (see `backlog/597-login-onboarding-primitives.md`)
4. Deprecate `tenantId` user attribute

---

## Problem Statement

The application currently has hardcoded tenant defaults (`"atlas-org"`) scattered throughout the codebase, violating true multi-tenancy principles. The user has explicitly requested:

> "i dont want default of any kind at any level. the application should work at all level with the needed application logic to work with tenant registered in database/keycloak"

## Current Architecture Issues

### 1. Hardcoded Tenant Defaults

**Location**: Multiple files require `DEFAULT_TENANT_ID` environment variable

```typescript
// lib/sdk/organizations.ts:11-16
function getRequiredTenantId(): string {
  const tenantId = process.env.DEFAULT_TENANT_ID?.trim();
  if (!tenantId) {
    throw new Error('DEFAULT_TENANT_ID environment variable is required but not set');
  }
  return tenantId;
}
```

This function is called by:
- `fetchOrganizations()` - Line 30
- `buildRequestContext()` - Line 21

**Problem**: SDK layer requires a global default tenant instead of using authenticated user's tenant.

### 2. Fallback Chain in Authentication

**Location**: `app/(auth)/auth.ts:400`

```typescript
const tenantId = (profile.tenantId as string) ??
                 (profile.organizationId as string) ??
                 getFallbackOrgId();  // ❌ Falls back to hardcoded default
```

**Problem**: When Keycloak profile doesn't contain `tenantId`, falls back to hardcoded value.

### 3. Hardcoded Defaults in API Routes

**Locations**: 40+ API route files

```typescript
// app/api/repositories/route.ts:13
const DEFAULT_TENANT_ID = process.env.NEXT_PUBLIC_TENANT_ID ?? 'atlas-org';

// app/api/transformations/route.ts:7
const DEFAULT_TENANT_ID = process.env.NEXT_PUBLIC_TENANT_ID ?? 'atlas-org';

// app/api/elements/route.ts:9
const DEFAULT_TENANT_ID = process.env.NEXT_PUBLIC_TENANT_ID ?? 'atlas-org';
```

**Problem**: Every API route has its own fallback to hardcoded tenant.

### 4. Organization Context Resolution

**Location**: `lib/auth/org-scope.ts:123-131`

```typescript
const defaultTenant = process.env.NEXT_PUBLIC_DEFAULT_TENANT_ID?.trim();
if (defaultTenant && !tenantHints.includes(defaultTenant)) {
  tenantHints.push(defaultTenant);
}

const internalDefault = process.env.DEFAULT_TENANT_ID?.trim();
if (internalDefault && !tenantHints.includes(internalDefault)) {
  tenantHints.push(internalDefault);
}
```

**Problem**: Organization resolution tries environment variable fallbacks before failing.

## Desired Architecture

### Tenant Resolution Flow

```
User Login → Keycloak → JWT Token with tenantId claim →
  Session → All API calls use session.user.tenantId
```

### Key Principles

1. **No Hardcoded Defaults**: Every tenant ID must come from authenticated user context
2. **Keycloak as Source of Truth**: User's tenant assignment stored in Keycloak user attributes
3. **Fail Gracefully**: If tenant cannot be determined, redirect to error page (not crash with defaults)
4. **Database Registration**: Tenants must exist in database before users can be assigned to them

## Solution Design

### Phase 1: Keycloak Configuration

**Add Custom User Attribute**

1. In Keycloak Admin Console, navigate to: `Realm Settings → User Profile`
2. Add new attribute:
   - **Attribute Name**: `tenantId`
   - **Display Name**: "Tenant ID"
   - **Required**: Yes
   - **Validation**: Regex pattern `^[a-z0-9][a-z0-9-]*$`

**Configure Token Mapper**

1. Navigate to: `Clients → platform-app → Client Scopes → Dedicated Scope → Mappers`
2. Create new mapper:
   - **Mapper Type**: User Attribute
   - **Name**: `tenantId`
   - **User Attribute**: `tenantId`
   - **Token Claim Name**: `tenantId`
   - **Claim JSON Type**: String
   - **Add to ID token**: Yes
   - **Add to access token**: Yes
   - **Add to userinfo**: Yes

**Set Default Tenant for Existing Users**

```bash
# Using Keycloak Admin API
curl -X PUT "https://auth.dev.ameide.io/admin/realms/ameide/users/{userId}" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "attributes": {
      "tenantId": ["atlas-org"]
    }
  }'
```

### Phase 2: Code Changes

#### 2.1. Update JWT Callback to Extract Tenant from Claims

**File**: `app/(auth)/auth.ts`

```typescript
// Line 400 - BEFORE:
const tenantId = (profile.tenantId as string) ??
                 (profile.organizationId as string) ??
                 getFallbackOrgId();

// Line 400 - AFTER:
const tenantIdFromProfile = (profile.tenantId as string) ??
                            (profile.organizationId as string);

if (!tenantIdFromProfile || tenantIdFromProfile.trim().length === 0) {
  throw new Error(
    'User account is missing tenantId. Please contact your administrator to assign you to a tenant.'
  );
}

const tenantId = tenantIdFromProfile.trim();
```

#### 2.2. Store Tenant in JWT Token

**File**: `app/(auth)/auth.ts`

```typescript
// Add to JWT callback (around line 506)
token.tenantId = tenantId;  // Store tenant from Keycloak claim
```

#### 2.3. Expose Tenant in Session

**File**: `app/(auth)/auth.ts`

```typescript
// Add to session callback (around line 694)
session.user.tenantId = token.tenantId as string;
```

#### 2.4. Update SDK Layer to Use Session Tenant

**File**: `lib/sdk/organizations.ts`

```typescript
// BEFORE:
function getRequiredTenantId(): string {
  const tenantId = process.env.DEFAULT_TENANT_ID?.trim();
  if (!tenantId) {
    throw new Error('DEFAULT_TENANT_ID environment variable is required but not set');
  }
  return tenantId;
}

// AFTER:
function getTenantIdFromContext(tenantId?: string): string {
  if (!tenantId || tenantId.trim().length === 0) {
    throw new Error('Tenant ID is required but not provided in request context');
  }
  return tenantId.trim();
}

// Update function signatures to require tenantId parameter
export async function fetchOrganizations(
  userId?: string,
  tenantId?: string,  // NEW: Required parameter
  client?: AmeideClient
) {
  const svc = client ?? getServerClient();
  const resolvedTenantId = getTenantIdFromContext(tenantId);

  const requestContext = create(common.RequestContextSchema, {
    tenantId: resolvedTenantId,
    userId,
    requestId: `org-fetch-${Date.now()}`,
  });

  // ... rest of function
}
```

#### 2.5. Update API Routes to Use Session Tenant

**Pattern for all API routes**:

```typescript
// BEFORE:
const DEFAULT_TENANT_ID = process.env.NEXT_PUBLIC_TENANT_ID ?? 'atlas-org';

// AFTER:
import { auth } from '@/app/(auth)/auth';

export async function GET(request: Request) {
  const session = await auth();
  if (!session?.user?.tenantId) {
    return Response.json(
      { error: 'Tenant context required' },
      { status: 401 }
    );
  }

  const tenantId = session.user.tenantId;

  // Use tenantId from session...
}
```

**Files to update** (40+ files):
- `app/api/elements/route.ts`
- `app/api/transformations/route.ts`
- `app/api/transformations/[transformationId]/route.ts`
- `app/api/repositories/route.ts`
- `app/api/repositories/[repositoryId]/route.ts`
- `app/api/threads/stream/route.ts`
- `lib/api/hooks.ts`
- ... (see grep results from earlier)

#### 2.6. Update Organization Context Resolution

**File**: `lib/auth/org-scope.ts`

```typescript
// REMOVE lines 123-131 (env var fallbacks)
// REMOVE:
// const defaultTenant = process.env.NEXT_PUBLIC_DEFAULT_TENANT_ID?.trim();
// const internalDefault = process.env.DEFAULT_TENANT_ID?.trim();

// Historical implementation (now deprecated after safeAuth removal):
// async function resolveOrgScopeInternal(params: ResolveOrgScopeParams = {}): Promise<ResolvedOrgScope> {
//   const session = await safeAuth();
//   if (!session) {
//     throw new Error('resolveOrgScope requires an authenticated session');
//   }
//
//   if (!session.user.tenantId) {
//     throw new Error('User session is missing tenant context');
//   }
//
//   // Use session.user.tenantId instead of environment variables
//   const tenantHints: string[] = [session.user.tenantId];
//   // ... rest of function
// }
//
// Current guidance (2025‑12‑01):
// - `resolveOrgScope` is deprecated and throws if called.
// - New org/tenant resolution should rely on:
//   - Middleware-injected headers (`x-tenant-id`, `x-user-id`, `x-user-kc-sub`, `x-org-home`) for routing and layout decisions.
//   - Direct use of `getSession()` in targeted server utilities when full session details are strictly required.
```

#### 2.7. Remove Organization Fallback Helper

**File**: `lib/auth/organization.ts`

```typescript
// REMOVE getDefaultFallback() function (lines 48-57)
// REMOVE getFallbackOrgId() export

// Keep only slugifyOrgId and resolveOrganizationContext
```

### Phase 3: Infrastructure Changes

#### 3.1. Organization context is session-only (no default-org env)

`www-ameide-platform` must derive **tenant + org** exclusively from the auth session (JWT claims + middleware headers), not from environment defaults.

- Dev data determinism comes from **GitOps seeding + verification** (see [582 – Local Dev Seeding](../582-local-dev-seeding.md)), not app fallbacks.
- **Update (648):** Telepresence should only require connectivity-critical env vars (e.g., `AMEIDE_GRPC_BASE_URL`). Org defaults and browser RPC env vars are not part of the runtime contract.

If tests need an explicit org ID/slug, use a **test-only entrypoint/fixture** rather than runtime fallbacks (e.g. `WWW_AMEIDE_PLATFORM_ORG_ID` for integration tests).

#### 3.2. Update Type Definitions

**File**: `app/(auth)/types.d.ts` (or equivalent)

```typescript
import 'next-auth';

declare module 'next-auth' {
  interface Session {
    user: {
      id: string;
      tenantId: string;  // NEW: Add tenant to session type
      // ... other fields
    }
  }

  interface User {
    tenantId?: string;  // NEW: Add tenant to user type
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    tenantId?: string;  // NEW: Add tenant to JWT type
    // ... other fields
  }
}
```

## Implementation Plan

### Week 1: Infrastructure & Keycloak Setup

- [ ] Configure Keycloak user attribute for `tenantId`
- [ ] Add token mapper to include `tenantId` in JWT claims
- [ ] Set `tenantId` attribute for all existing users in Keycloak
- [ ] Verify token claims include `tenantId` after login

### Week 2: Core Authentication Layer

- [ ] Update JWT callback to extract `tenantId` from claims (fail if missing)
- [ ] Store `tenantId` in JWT token
- [ ] Expose `tenantId` in session object
- [ ] Update TypeScript type definitions
- [ ] Remove `getFallbackOrgId()` and `getDefaultFallback()` functions

### Week 3: SDK & API Layer Refactoring

- [ ] Update `lib/sdk/organizations.ts` to accept `tenantId` parameter
- [ ] Update all API routes to use `session.user.tenantId` (40+ files)
- [ ] Remove `DEFAULT_TENANT_ID` constants from all files
- [ ] Update `lib/auth/org-scope.ts` to remove env var fallbacks

### Week 4: Testing & Validation

- [ ] Test login flow with users having `tenantId` attribute
- [ ] Test login flow with users MISSING `tenantId` (should fail gracefully)
- [ ] Test API routes with tenant isolation
- [ ] Test organization switching within same tenant
- [ ] Remove environment variables from Helm configuration
- [ ] Deploy to staging and validate

## Error Handling

### Missing tenantId in Keycloak

**Scenario**: User logs in but Keycloak profile doesn't contain `tenantId`

**Current Behavior**: Falls back to hardcoded "atlas-org"

**New Behavior**:
1. JWT callback throws error
2. Sign-in fails
3. User sees error page: "Your account is not configured for multi-tenant access. Please contact your administrator."

### Invalid tenantId

**Scenario**: User's `tenantId` doesn't exist in platform database

**Behavior**:
1. Authentication succeeds (Keycloak is source of truth)
2. First API call fails with 404
3. UI shows error: "Tenant not found. Please contact your administrator."

### Tenant Reassignment

**Scenario**: Administrator changes user's `tenantId` in Keycloak

**Behavior**:
1. User must log out and log back in
2. New session will have updated `tenantId`
3. Access to old tenant's data is revoked immediately

## Testing Strategy

### Unit Tests

```typescript
describe('getTenantIdFromContext', () => {
  it('should return tenant when provided', () => {
    expect(getTenantIdFromContext('tenant-123')).toBe('tenant-123');
  });

  it('should trim whitespace', () => {
    expect(getTenantIdFromContext('  tenant-123  ')).toBe('tenant-123');
  });

  it('should throw when tenant is undefined', () => {
    expect(() => getTenantIdFromContext(undefined)).toThrow('Tenant ID is required');
  });

  it('should throw when tenant is empty string', () => {
    expect(() => getTenantIdFromContext('')).toThrow('Tenant ID is required');
  });
});
```

### Integration Tests

```typescript
describe('API Routes with Tenant Isolation', () => {
  it('should only return organizations from user tenant', async () => {
    const session = await signIn('user@tenant-a.com');
    const response = await fetch('/api/organizations');
    const orgs = await response.json();

    expect(orgs.every(org => org.tenantId === 'tenant-a')).toBe(true);
  });

  it('should reject requests without session', async () => {
    const response = await fetch('/api/organizations');
    expect(response.status).toBe(401);
  });
});
```

## Migration Guide for Existing Deployments

### Step 1: Configure Keycloak (Non-Breaking)

Add `tenantId` attribute and mapper to Keycloak. This does NOT break existing deployments because the code still has fallbacks.

### Step 2: Populate User Attributes

Run migration script to set `tenantId` for all existing users:

```bash
./scripts/migrate-keycloak-tenant-ids.sh
```

### Step 3: Deploy Code Changes

Deploy the refactored code that removes fallbacks. At this point, all users MUST have `tenantId` set.

### Step 4: Remove Environment Variables

Update Helm values to remove `DEFAULT_TENANT_ID` and `AUTH_DEFAULT_ORG`.

### Step 5: Verify & Monitor

Monitor authentication errors and API failures. Any issues indicate users without `tenantId` set.

## Success Criteria

- [ ] No hardcoded tenant defaults anywhere in codebase
- [ ] All tenant IDs come from Keycloak user attributes
- [ ] API routes enforce tenant isolation using session context
- [ ] Login fails gracefully if user missing `tenantId`
- [ ] Application works with multiple tenants without configuration changes
- [ ] Tests pass for multi-tenant scenarios

## References

- [Keycloak User Attributes Documentation](https://www.keycloak.org/docs/latest/server_admin/#user-attributes)
- [Token Mappers](https://www.keycloak.org/docs/latest/server_admin/#_protocol-mappers)
- backlog/329-authz.md - Authorization architecture gaps
- backlog/322-rbac.md - Role-based access control
