> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Backlog Item 323: Keycloak Realm Configuration for RBAC & Multi-Tenancy

**Status**: ✅ Completed (Single-Realm Implementation)
**Priority**: High
**Component**: Infrastructure, Authentication
**Related**: [320-header.md](./320-header.md), [322-rbac.md](./322-rbac.md), [330-dynamic-tenant-resolution.md](./330-dynamic-tenant-resolution.md), [333-realms.md](./333-realms.md)

---

## ⚠️ ARCHITECTURE UPDATE (2025-10-30)

**This configuration describes single-realm multi-tenancy.** The target architecture is **realm-per-tenant**. See [backlog/333-realms.md](./333-realms.md).

**What changes with realm-per-tenant**:

### Current Configuration (Single Realm)
```
Keycloak Realm: "ameide"
├─ Client Scopes:
│  ├─ profile, email (built-in OIDC client scopes - required)
│  ├─ roles (realm_access.roles + resource_access)
│  ├─ tenant (tenantId attribute)
│  └─ groups (org membership via groups)
├─ Users have attributes: tenantId, groups
└─ JWT issuer: https://keycloak/realms/ameide
```

> **⚠️ Vendor OIDC Scopes**: This design assumes the realm keeps the built-in OIDC client scopes `profile`, `email`, and (optionally) `offline_access`, and that we do NOT create a client scope called `openid`. The `openid` value in Auth.js's `scope=openid profile email` is the protocol-level meta-scope—Keycloak only looks up `profile` and `email` as client scopes. For details, see [426-keycloak-config-map.md §3.1](426-keycloak-config-map.md) and [460-keycloak-oidc-scopes.md](460-keycloak-oidc-scopes.md).

### Realm-Per-Tenant Configuration (Target)
```
Keycloak Realm: "acme"  ← Organization-specific
├─ Client Scopes:
│  ├─ roles (realm_access.roles only)
│  └─ NO tenant scope (realm IS the tenant)
├─ Users native to this realm
├─ Identity Provider: ACME's Okta/Azure AD
└─ JWT issuer: https://keycloak/realms/acme
```

**Simplifications**:
- ❌ No `tenant` client scope needed (realm name = org slug = tenant)
- ❌ No `groups` client scope for org membership (realm membership = org membership)
- ❌ No `resource_access` complexity (one realm = one organization's app)
- ✅ Simpler JWT: just `realm_access.roles` for that organization
- ✅ Each org configures their own realm independently

**What remains from this document**:
- ✅ `roles` client scope configuration pattern (applied per realm)
- ✅ Protocol mapper structure (realm roles mapper)
- ✅ JWT role extraction logic (simpler with one realm)
- ✅ Helm/GitOps deployment approach

**Migration timeline**:
- ✅ **Current**: Single realm with `roles` + `tenant` + `groups` scopes
- 🔄 **Week 1-2**: Implement realm provisioning (333-realms.md)
- 🔄 **Week 2-3**: Switch to realm-per-tenant authentication
- 🔄 **Week 4**: Deprecate `tenant` and `groups` scopes

---

## Problem Statement

After implementing server-side navigation with RBAC (backlog/320-header.md), all navigation tabs were showing as disabled for admin users despite having the admin role assigned in Keycloak. Investigation revealed that JWT tokens issued by Keycloak were not including user roles in the token claims.

### Symptoms
- Admin user logged in successfully but all protected tabs were disabled
- JWT token showed `realm_access: null` and `resource_access: null`
- Server logs showed `roles: []` and `isAdmin: false` for authenticated users
- Navigation tabs requiring permissions (Initiatives, Governance, Insights, Settings) were grayed out

### Root Cause
The Keycloak `platform-app` client was missing the **`roles` client scope** and associated protocol mappers. Without this scope:
- Realm roles were not included in JWT tokens
- The `extractRoles()` function in `lib/keycloak.ts` couldn't find roles in token claims
- Server-side RBAC checks in `features/navigation/server/access.ts` denied access to all protected resources

## Solution

### 1. Created `roles` Client Scope

Added a new client scope to the Keycloak realm configuration with two protocol mappers:

**Realm Roles Mapper**:
- Maps user's realm-level roles to `realm_access.roles` claim
- Example: `["admin", "user"]`

**Client Roles Mapper**:
- Maps user's client-specific roles to `resource_access.{client_id}.roles` claim
- Example: `resource_access.platform-app.roles: ["workflows-manager"]`

### 2. Updated Helm Configuration

Modified `infra/kubernetes/charts/platform/keycloak-realm/values.yaml`:

```yaml
"clientScopes": [
  {
    "name": "roles",
    "description": "OpenID Connect scope for add user roles to the access token",
    "protocol": "openid-connect",
    "attributes": {
      "include.in.token.scope": "true",
      "display.on.consent.screen": "true"
    },
    "protocolMappers": [
      {
        "name": "realm roles",
        "protocol": "openid-connect",
        "protocolMapper": "oidc-usermodel-realm-role-mapper",
        "consentRequired": false,
        "config": {
          "multivalued": "true",
          "userinfo.token.claim": "true",
          "id.token.claim": "true",
          "access.token.claim": "true",
          "claim.name": "realm_access.roles",
          "jsonType.label": "String"
        }
      },
      {
        "name": "client roles",
        "protocol": "openid-connect",
        "protocolMapper": "oidc-usermodel-client-role-mapper",
        "consentRequired": false,
        "config": {
          "multivalued": "true",
          "userinfo.token.claim": "true",
          "id.token.claim": "true",
          "access.token.claim": "true",
          "claim.name": "resource_access.${client_id}.roles",
          "jsonType.label": "String"
        }
      }
    ]
  },
  {
    "name": "groups",
    "description": "Groups claim",
    "protocol": "openid-connect",
    // ... existing groups scope
  }
]
```

### 3. Updated Default Client Scopes

Changed realm and client default scopes to include `roles`:

```yaml
# Realm-level default
"defaultDefaultClientScopes": ["profile", "email", "roles", "groups"]

# Each client's default scopes
"defaultClientScopes": ["openid", "profile", "email", "roles", "groups"]
```

### 4. Deployed via Helm

```bash
helm upgrade keycloak-realm /workspace/infra/kubernetes/charts/platform/keycloak-realm \
  -n ameide \
  -f /workspace/infra/kubernetes/charts/platform/keycloak-realm/values.yaml
```

The Keycloak Operator's `KeycloakRealmImport` CRD automatically applied the changes to the running Keycloak instance.

## How It Works

### JWT Token Structure (Before)
```json
{
  "preferred_username": "admin",
  "email": "admin@ameide.io",
  "name": "Francesco Magalini",
  "realm_access": null,
  "resource_access": null
}
```

### JWT Token Structure (After)
```json
{
  "preferred_username": "admin",
  "email": "admin@ameide.io",
  "name": "Francesco Magalini",
  "realm_access": {
    "roles": ["admin", "user"]
  },
  "resource_access": {
    "platform-app": {
      "roles": []
    }
  }
}
```

### Role Extraction Flow

1. **User logs in** via NextAuth with Keycloak provider
2. **Keycloak issues JWT** with roles claims (due to `roles` scope)
3. **auth.ts JWT callback** calls `extractRoles(decoded)` from `lib/keycloak.ts`
4. **extractRoles()** extracts roles from `realm_access.roles` and `resource_access.{client_id}.roles`
5. **Session object** includes `user.roles` array and `user.isAdmin` boolean
6. **Server-side RBAC** in `features/navigation/server/access.ts` checks roles via `isAdmin()` and `hasAnyRole()`

**Code References**:
- Role extraction: [lib/keycloak.ts:207-234](../services/www_ameide_platform/lib/keycloak.ts#L207-L234)
- Authorization helpers: [lib/auth/authorization.ts:41-43](../services/www_ameide_platform/lib/auth/authorization.ts#L41-L43)
- Access control: [features/navigation/server/access.ts](../services/www_ameide_platform/features/navigation/server/access.ts)

## Testing

### Manual Testing

1. **Decode JWT token** from browser:
   ```bash
   # Get access token via password grant
   curl -k -s -X POST https://auth.dev.ameide.io/realms/ameide/protocol/openid-connect/token \
     --data-urlencode "username=admin" \
     --data-urlencode "password=YOUR_PASSWORD" \
     --data-urlencode "grant_type=password" \
     --data-urlencode "client_id=platform-app" \
     --data-urlencode "client_secret=changeme" | jq -r '.access_token' | \
     cut -d'.' -f2 | base64 -d | jq '{realm_access, resource_access}'
   ```

2. **Expected output**:
   ```json
   {
     "realm_access": {
       "roles": ["admin"]
     },
     "resource_access": {
       "platform-app": {
         "roles": []
       }
     }
   }
   ```

3. **Test navigation RBAC**:
   - Login at https://platform.dev.ameide.io
   - Navigate to `/org/atlas`
   - Verify all tabs are enabled (not grayed out)
   - Hover over enabled tabs - no "Requires X role" tooltips

### Automated Testing

The existing integration tests verify RBAC behavior:
- `features/navigation/__tests__/integration/accessibility.test.tsx`
- `features/navigation/server/__tests__/access.test.ts`
- `features/navigation/server/__tests__/rbac.test.ts`

## Infrastructure-as-Code

The configuration is fully managed via Helm and GitOps:

**Chart**: `gitops/ameide-gitops/sources/charts/foundation/operators-config/keycloak_realm`  
**Values**: `gitops/ameide-gitops/sources/values/_shared/platform/platform-keycloak-realm.yaml` + environment overlay  
**CRD**: `KeycloakRealmImport` (managed by Keycloak Operator)

### Deployment Strategy

**Local Development**:
```bash
helm upgrade keycloak-realm ./infra/kubernetes/charts/platform/keycloak-realm \
  -n ameide \
  -f ./infra/kubernetes/charts/platform/keycloak-realm/values.yaml
```

**Production/Staging**:
- Managed by Argo CD via `environments/<env>/components/platform/auth/keycloak-realm`
- Sync order: `platform-postgres-clusters` (CNPG-managed DB creds) → `foundation-keycloak-admin-secrets` (bootstrap/admin/client secrets) → `platform-keycloak` → `platform-keycloak-realm`
- Changes propagate automatically via GitOps; Keycloak Operator applies the realm import declaratively once all prerequisite secrets exist.
- **Post-sync**: `client-patcher` Job extracts Keycloak-generated client secrets to Vault. See [426-keycloak-config-map.md §3.2](426-keycloak-config-map.md).

### Rollback

If issues occur, rollback via Helm:
```bash
helm rollback keycloak-realm -n ameide
```

To clear a bad realm import without downtime, patch the `KeycloakRealmImport` CR back to the last working revision and ensure the prerequisite secrets (`keycloak-bootstrap-admin`, `keycloak-master-bootstrap`, `platform-app-master-client`) still exist; the Operator will reconcile back to the previous state.

## Role Definitions

### Legacy (Single Realm)

The RBAC system recognizes these roles:

**Platform-Level Roles**:
- **`admin`** - Full system access (bypasses all permission checks)
- **`platform-admin`** - Platform administration
- **`platform-operator`** - Platform operations
- **`workflows-manager`** - Workflow management
- **`governance-viewer`** - View governance features
- **`architect`** - Architecture modeling
- **`analyst`** - Analytics and insights
- **`viewer`** - Read-only access
- **`contributor`** - Create and edit content
- **`repo-admin`** - Repository administration

**Organization-Specific Roles** (Legacy Pattern):
Format: `org:{orgId}:{role}` (deprecated with realm-per-tenant rollout)

Examples (legacy only):
- `org:atlas:admin` - Admin for Atlas organization
- `org:atlas:contributor` - Contributor for Atlas organization
- `org:acme:viewer` - Viewer for ACME organization

### Current (Realm-Per-Tenant)

**Realm Roles (per organization realm)**:
Each organization realm defines its own simple roles:

- **`admin`** / **`owner`** - Organization administrator
- **`contributor`** - Can create and edit content
- **`viewer`** - Read-only access
- **`guest`** - External user from another organization (B2B)

**Example**: In realm "acme":
```json
{
  "iss": "https://keycloak/realms/acme",
  "realm_access": {
    "roles": ["admin"]  // Realm-native role; no prefix required
  }
}
```

**Benefits**:
- Simpler role names (no prefixes)
- Each org defines roles independently
- No role name conflicts across orgs
- Standard Keycloak role management

### Permission Mapping

See [features/navigation/server/access.ts](../services/www_ameide_platform/features/navigation/server/access.ts) for detailed permission checks:

| Feature | Required Roles |
|---------|---------------|
| Initiatives | `admin`, `contributor`, `viewer`, `architect` |
| Governance | `admin`, `governance-viewer`, `architect`, `compliance-officer` |
| Insights | `admin`, `analyst`, `viewer` |
| Settings | `admin`, `owner` |
| Workflows | `admin`, `platform-admin`, `workflows-manager`, `platform-operator` |
| Repositories | `admin`, `repo-admin`, `platform-admin` |

## Future Enhancements

### 1. Fine-Grained Permissions
Consider moving from coarse-grained roles to fine-grained permissions:
- Create permission-based policies in Keycloak
- Use authorization services for resource-level access control
- Implement policy evaluation in API routes

### 2. Dynamic Role Assignment
- Implement UI for organization admins to assign roles
- Create API endpoints for role management
- Sync role changes via Keycloak Admin API

### 3. Audit Logging
- Log all role-based access decisions
- Track role assignment changes
- Alert on suspicious permission escalations

### 4. Role Hierarchy
Define role inheritance in Keycloak:
```
admin → platform-admin → workflows-manager → contributor → viewer
```

### 5. Conditional Access
- Implement context-aware access policies
- Consider time-based access restrictions
- Add MFA requirements for sensitive operations

## Related Documentation

- [backlog/320-header.md](./320-header.md) - Server-side navigation descriptor implementation
- [backlog/322-rbac.md](./322-rbac.md) - RBAC system design
- [features/navigation/README.md](../services/www_ameide_platform/features/navigation/README.md) - Navigation system documentation
- [Keycloak Operator Docs](https://www.keycloak.org/guides#operator)

## Lessons Learned

1. **Default scopes are not enough** - Standard OIDC scopes (`openid`, `profile`, `email`) don't include roles by default
2. **Protocol mappers are required** - Roles must be explicitly mapped to token claims via `oidc-usermodel-realm-role-mapper`
3. **Include in token scope matters** - The `include.in.token.scope` attribute must be `true` for the scope to be active
4. **Realm import is declarative** - The Keycloak Operator reconciles state, so changes via admin console may be overwritten
5. **Session refresh required** - Users must logout/login after role changes to get fresh JWT tokens

## Verification Checklist

- [x] `roles` client scope created in Keycloak
- [x] Realm roles mapper configured with `realm_access.roles` claim
- [x] Client roles mapper configured with `resource_access.{client_id}.roles` claim
- [x] `roles` scope added to realm default scopes
- [x] `roles` scope added to `platform-app` client default scopes
- [x] Configuration committed to Helm chart values
- [x] Helm release upgraded successfully
- [x] KeycloakRealmImport status shows "Done" with no errors
- [x] JWT token includes `realm_access.roles` claim
- [x] Admin user can access all navigation tabs
- [x] RBAC checks pass in server-side code
- [x] Session includes `user.roles` and `user.isAdmin`

## Update: Tenant ID Configuration (2025-10-30)

### Problem: Hardcoded Tenant Defaults

The application had hardcoded tenant defaults (`"atlas-org"`) in 40+ locations, violating multi-tenancy principles. User requirement:

> "i dont want default of any kind at any level. the application should work at all level with the needed application logic to work with tenant registered in database/keycloak"

### Solution: Tenant ID from Keycloak User Attributes

Added a new **`tenant` client scope** to include `tenantId` user attribute in JWT tokens.

#### Configuration Changes

**1. Added `tenant` Client Scope**

```yaml
"clientScopes": [
  {
    "name": "tenant",
    "description": "Tenant ID claim for multi-tenancy",
    "protocol": "openid-connect",
    "attributes": {
      "include.in.token.scope": "true",
      "display.on.consent.screen": "false"
    },
    "protocolMappers": [
      {
        "name": "tenantId",
        "protocol": "openid-connect",
        "protocolMapper": "oidc-usermodel-attribute-mapper",
        "consentRequired": false,
        "config": {
          "user.attribute": "tenantId",
          "claim.name": "tenantId",
          "jsonType.label": "String",
          "id.token.claim": "true",
          "access.token.claim": "true",
          "userinfo.token.claim": "true"
        }
      }
    ]
  },
  // ... existing roles and groups scopes
]
```

**2. Updated Default Client Scopes**

```yaml
# Realm-level default (was: ["profile", "email", "roles", "groups"])
"defaultDefaultClientScopes": ["profile", "email", "roles", "groups", "tenant"]

# Each client's default scopes
"defaultClientScopes": ["openid", "profile", "email", "roles", "groups", "tenant"]
```

**3. Added `tenantId` Attribute to Users**

```yaml
"users": [
  {
    "username": "admin",
    "attributes": {
      "tenantId": ["atlas-org"]  # NEW: Tenant assignment
    }
  },
  {
    "username": "user",
    "attributes": {
      "tenantId": ["atlas-org"]
    }
  }
]
```

#### JWT Token Structure (After Tenant Update)

```json
{
  "sub": "a1b2c3d4-...",
  "preferred_username": "admin",
  "email": "admin@ameide.io",
  "name": "Admin User",
  "tenantId": "atlas-org",    // ✅ NEW: Tenant from user attribute
  "realm_access": {
    "roles": ["admin"]
  },
  "resource_access": {
    "platform-app": {
      "roles": []
    }
  }
}
```

#### Tenant Resolution Flow

1. **User logs in** → Keycloak checks `user.attributes.tenantId`
2. **Protocol mapper** extracts `tenantId` attribute → adds to JWT as `tenantId` claim
3. **auth.ts JWT callback** extracts `tenantId` from token claims
4. **Session object** includes `session.user.tenantId`
5. **API routes** use `session.user.tenantId` for all database queries (no hardcoded defaults)

#### Benefits

- **No hardcoded defaults**: Tenant ID always comes from authenticated user context
- **Database-driven**: Tenants registered in database, users assigned in Keycloak
- **Multi-tenant isolation**: Each user sees only their tenant's data
- **Fail-safe**: Missing `tenantId` causes auth failure (not silent fallback)

#### Testing Tenant Configuration

**Decode JWT and verify `tenantId` claim**:

```bash
TOKEN=$(curl -k -s -X POST https://auth.dev.ameide.io/realms/ameide/protocol/openid-connect/token \
  --data-urlencode "username=admin" \
  --data-urlencode "password=YOUR_PASSWORD" \
  --data-urlencode "grant_type=password" \
  --data-urlencode "client_id=platform-app" \
  --data-urlencode "client_secret=changeme" | jq -r '.id_token')

echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | jq '{tenantId, realm_access, resource_access}'
```

**Expected output**:
```json
{
  "tenantId": "atlas-org",
  "realm_access": {
    "roles": ["admin"]
  },
  "resource_access": {
    "platform-app": {
      "roles": []
    }
  }
}
```

#### Next Phase: Code Refactoring

See [backlog/330-dynamic-tenant-resolution.md](./330-dynamic-tenant-resolution.md) for the code changes to:
- Extract `tenantId` from JWT in auth callback
- Remove all hardcoded `DEFAULT_TENANT_ID` constants
- Update 40+ API routes to use `session.user.tenantId`
- Remove environment variable fallbacks

## Completion Summary

**Date**: 2025-10-30
**Status**: ✅ Single-realm configuration complete
**Helm Release**: `keycloak-realm`
**Deployment**: Local cluster (k3d)
**Verification**: JWT tokens include roles AND tenantId claims

**Current Configuration** (Single Realm):
1. **`roles` client scope** - User role information for RBAC
2. **`tenant` client scope** - User tenant assignment for multi-tenancy
3. **`groups` client scope** - Organization membership via Keycloak groups

This configuration is managed declaratively via Helm and will propagate to all environments via GitOps.

**⚠️ Future Work**: Migrate to realm-per-tenant architecture (see [333-realms.md](./333-realms.md)) where:
- Each organization gets dedicated Keycloak realm
- Realm name = organization slug
- Simpler role structure (no prefixes)
- Organization configures own SSO (Okta, Azure AD, etc.)

---

# Realm-Per-Tenant Impact Analysis (Detailed)

This section provides a comprehensive analysis of how the realm-per-tenant architecture affects Keycloak configuration, JWT structure, protocol mappers, and infrastructure code.

## Overview of Changes

### Current Architecture: Single Realm with Multiple Client Scopes
```yaml
Keycloak:
  Realm: "ameide"  # Single realm for all organizations
  Client Scopes:
    - roles          # Maps realm_access.roles + resource_access.{client}.roles
    - tenant         # Maps tenantId user attribute
    - groups         # Maps organization membership via groups
    - organizations  # Maps org_groups for organization keys

  Users:
    - attributes:
        tenantId: "atlas-org"
        groups: ["/orgs/atlas", "/orgs/acme"]

  JWT Claims:
    iss: "https://keycloak/realms/ameide"
    tenantId: "atlas-org"
    realm_access.roles: ["admin", "contributor"]
    groups: ["/orgs/atlas", "/orgs/acme"]
    org_groups: ["/orgs/atlas"]
```

### Target Architecture: Realm-Per-Tenant
```yaml
Keycloak:
  Realm: "atlas"  # Dedicated realm for Atlas organization
  Client Scopes:
    - roles         # Maps realm_access.roles ONLY (no resource_access)
    # NO tenant scope (realm name IS the tenant)
    # NO groups scope (realm membership IS the organization)
    # NO organizations scope (only one org per realm)

  Users:
    # Native realm users OR federated from Atlas's Okta
    # NO tenantId attribute needed
    # NO groups for org membership

  JWT Claims:
    iss: "https://keycloak/realms/atlas"  # Realm = Org = Tenant
    realm_access.roles: ["admin"]          # Simple roles, no prefixes
    # NO tenantId claim
    # NO groups claim
    # NO org_groups claim
```

**Key Insight**: The realm-per-tenant architecture **eliminates 3 out of 4 client scopes**, simplifying Keycloak configuration by 75%.

---

## Impact 1: Client Scope Configuration Simplification

### Current Implementation (Single Realm)

**File**: [infra/kubernetes/charts/platform/keycloak-realm/values.yaml:94-140](infra/kubernetes/charts/platform/keycloak-realm/values.yaml#L94-L140)

```yaml
"clientScopes": [
  # 1. ROLES SCOPE (will remain)
  {
    "name": "roles",
    "description": "OpenID Connect scope for add user roles to the access token",
    "protocol": "openid-connect",
    "protocolMappers": [
      {
        "name": "realm roles",
        "protocolMapper": "oidc-usermodel-realm-role-mapper",
        "config": {
          "claim.name": "realm_access.roles"
        }
      },
      {
        "name": "client roles",
        "protocolMapper": "oidc-usermodel-client-role-mapper",
        "config": {
          "claim.name": "resource_access.${client_id}.roles"
        }
      }
    ]
  },

  # 2. TENANT SCOPE (will be removed)
  {
    "name": "tenant",
    "description": "Tenant ID claim for multi-tenancy",
    "protocol": "openid-connect",
    "protocolMappers": [
      {
        "name": "tenantId",
        "protocolMapper": "oidc-usermodel-attribute-mapper",
        "config": {
          "user.attribute": "tenantId",
          "claim.name": "tenantId"
        }
      }
    ]
  },

  # 3. GROUPS SCOPE (will be removed)
  {
    "name": "groups",
    "description": "Groups claim for organization membership",
    "protocol": "openid-connect",
    "protocolMappers": [
      {
        "name": "groups",
        "protocolMapper": "oidc-group-membership-mapper",
        "config": {
          "claim.name": "groups",
          "full.path": "true"
        }
      }
    ]
  },

  # 4. ORGANIZATIONS SCOPE (will be removed)
  {
    "name": "organizations",
    "description": "Organization membership from groups",
    "protocol": "openid-connect",
    "protocolMappers": [
      {
        "name": "org-groups",
        "protocolMapper": "oidc-group-membership-mapper",
        "config": {
          "claim.name": "org_groups",
          "full.path": "true"
        }
      }
    ]
  }
]

"defaultClientScopes": [
  "openid", "profile", "email",
  "roles", "tenant", "groups", "organizations"  # 4 custom scopes
]
```

**Complexity Metrics**:
- **Client scopes**: 4 custom scopes
- **Protocol mappers**: 6 mappers total
- **Configuration lines**: ~250 lines YAML
- **JWT claims**: 5 custom claims (`tenantId`, `realm_access`, `resource_access`, `groups`, `org_groups`)

### Future Implementation (Realm-Per-Tenant)

**File**: `infra/kubernetes/charts/platform/keycloak-realm-template/values.yaml` (NEW)

```yaml
"clientScopes": [
  # ONLY ROLES SCOPE (simplified)
  {
    "name": "roles",
    "description": "OpenID Connect scope for add user roles to the access token",
    "protocol": "openid-connect",
    "protocolMappers": [
      {
        "name": "realm roles",
        "protocolMapper": "oidc-usermodel-realm-role-mapper",
        "config": {
          "claim.name": "realm_access.roles"  # Only realm roles, no resource_access
        }
      }
      # NO client roles mapper (single realm = single app context)
    ]
  }
  # NO tenant scope
  # NO groups scope
  # NO organizations scope
]

"defaultClientScopes": [
  "openid", "profile", "email",
  "roles"  # Only 1 custom scope
]
```

**Complexity Metrics**:
- **Client scopes**: 1 custom scope (75% reduction)
- **Protocol mappers**: 1 mapper (83% reduction)
- **Configuration lines**: ~40 lines YAML (84% reduction)
- **JWT claims**: 1 custom claim (`realm_access.roles` only) (80% reduction)

### Quantified Impact

| Metric | Current (Single Realm) | Future (Realm-Per-Tenant) | Change |
|--------|------------------------|---------------------------|--------|
| **Client scopes** | 4 | 1 | **-75%** |
| **Protocol mappers** | 6 | 1 | **-83%** |
| **YAML configuration** | ~250 lines | ~40 lines | **-84%** |
| **JWT custom claims** | 5 | 1 | **-80%** |
| **Default scopes array** | 7 items | 4 items | **-43%** |

**Code Deletion**:
- ❌ Delete `tenant` client scope definition (~40 lines)
- ❌ Delete `groups` client scope definition (~50 lines)
- ❌ Delete `organizations` client scope definition (~50 lines)
- ❌ Delete `client roles` protocol mapper (~30 lines)

**Configuration Simplification**:
- ✅ Single protocol mapper per realm (realm roles only)
- ✅ Simpler default scopes (no org-specific scopes)
- ✅ Standard OIDC configuration (follows industry patterns)

---

## Impact 2: JWT Token Structure Simplification

### Current JWT Token (Single Realm)

**Token from**: `iss: "https://keycloak/realms/ameide"`

```json
{
  "iss": "https://keycloak/realms/ameide",
  "sub": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "aud": "platform-app",
  "exp": 1735689600,
  "iat": 1735686000,

  // Custom Claims (5 total)
  "tenantId": "atlas-org",                    // ❌ Will be removed
  "realm_access": {
    "roles": ["admin", "contributor"]     // Realm-native roles only
  },
  "resource_access": {                        // ❌ Will be removed
    "platform-app": {
      "roles": []
    }
  },
  "groups": [                                 // ❌ Will be removed
    "/orgs/atlas",
    "/orgs/acme",
    "/internal/engineers"
  ],
  "org_groups": [                             // ❌ Will be removed
    "/orgs/atlas"
  ],

  // Standard OIDC Claims
  "preferred_username": "admin",
  "email": "admin@atlas.com",
  "name": "Admin User"
}
```

**Token size**: ~850 bytes
**Custom claims**: 5
**Legacy role complexity**: Prefixed roles like `org:atlas:admin`

### Future JWT Token (Realm-Per-Tenant)

**Token from**: `iss: "https://keycloak/realms/atlas"` (realm = organization)

```json
{
  "iss": "https://keycloak/realms/atlas",    // ✅ Realm name = org slug
  "sub": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "aud": "app",                               // ✅ Simpler client name
  "exp": 1735689600,
  "iat": 1735686000,

  // Custom Claims (1 total)
  "realm_access": {
    "roles": ["admin"]                        // ✅ Simple roles, no prefixes
  },

  // NO tenantId (realm name IS the tenant)
  // NO resource_access (single app context)
  // NO groups (realm membership = org membership)
  // NO org_groups (only one org per realm)

  // Standard OIDC Claims
  "preferred_username": "admin",
  "email": "admin@atlas.com",
  "name": "Admin User"
}
```

**Token size**: ~420 bytes (50% reduction)
**Custom claims**: 1 (80% reduction)
**Roles complexity**: Simple strings like `admin`

### Quantified Impact

| Metric | Current | Future | Change |
|--------|---------|--------|--------|
| **Token size** | ~850 bytes | ~420 bytes | **-50%** |
| **Custom claims** | 5 | 1 | **-80%** |
| **Role name length** | `org:atlas:admin` (16 chars) | `admin` (5 chars) | **-69%** |
| **Nested claim levels** | 3 levels (resource_access.platform-app.roles) | 2 levels (realm_access.roles) | **-33%** |

**Benefits**:
- ✅ Smaller tokens = faster transmission
- ✅ Fewer claims = simpler parsing logic
- ✅ Standard structure = easier integration with 3rd party tools
- ✅ Realm name in issuer = automatic tenant resolution

---

## Impact 3: Token Validation & Parsing Logic

### Current Implementation (Single Realm)

**File**: [services/www_ameide_platform/lib/keycloak.ts:207-234](services/www_ameide_platform/lib/keycloak.ts#L207-L234)

```typescript
export function extractRoles(decoded: KeycloakAccessToken): string[] {
  const roles: string[] = [];

  // 1. Extract realm-level roles
  if (decoded.realm_access?.roles) {
    roles.push(...decoded.realm_access.roles);
  }

  // 2. Extract client-specific roles (complex logic)
  if (decoded.resource_access) {
    // Iterate through ALL clients
    Object.keys(decoded.resource_access).forEach((clientId) => {
      const clientRoles = decoded.resource_access?.[clientId]?.roles;
      if (clientRoles) {
        roles.push(...clientRoles);
      }
    });
  }

  return [...new Set(roles)];  // Deduplicate
}

// Called from auth.ts JWT callback
const roles = extractRoles(decoded);
const isAdmin = roles.includes('admin') || roles.includes('platform-admin');
```

**File**: [services/www_ameide_platform/app/(auth)/auth.ts:400](services/www_ameide_platform/app/(auth)/auth.ts#L400)

```typescript
// JWT callback extracts multiple claims
callbacks: {
  async jwt({ token, account, profile }) {
    if (account && profile) {
      // 1. Extract tenantId from custom claim
      const tenantId = (profile.tenantId as string) ??
                       (profile.organizationId as string);

      // 2. Extract roles from nested structure
      const roles = extractRoles(profile as KeycloakAccessToken);

      // 3. Extract org_groups from custom claim
      const orgGroups = extractOrgGroups(profile);

      token.tenantId = tenantId;
      token.roles = roles;
      token.organizations = await resolveOrganizations(orgGroups, tenantId);
    }
    return token;
  }
}
```

**Complexity**:
- 3 custom extraction functions (`extractRoles`, `extractOrgGroups`, `resolveOrganizations`)
- Nested loops (iterate through `resource_access` clients)
- Multiple claim sources (`tenantId`, `realm_access`, `resource_access`, `org_groups`)
- Error handling for missing claims

### Future Implementation (Realm-Per-Tenant)

**File**: `services/www_ameide_platform/lib/jwt.ts` (NEW simplified version)

```typescript
export function extractRoles(decoded: KeycloakAccessToken): string[] {
  // Simpler: Only realm_access.roles (no resource_access)
  return decoded.realm_access?.roles ?? [];
}

export function extractTenantFromIssuer(issuer: string): string {
  // Extract org slug from issuer URL
  // Example: "https://keycloak/realms/atlas" → "atlas"
  const match = issuer.match(/\/realms\/([^/]+)$/);
  if (!match) {
    throw new Error(`Invalid Keycloak issuer format: ${issuer}`);
  }
  return match[1];  // Realm name = org slug = tenant ID
}
```

**File**: `services/www_ameide_platform/app/(auth)/auth.ts` (simplified)

```typescript
// JWT callback simplified
callbacks: {
  async jwt({ token, account, profile }) {
    if (account && profile) {
      // 1. Extract tenant from issuer (NO custom claim needed)
      const tenantId = extractTenantFromIssuer(profile.iss);

      // 2. Extract roles (simplified, no client roles)
      const roles = extractRoles(profile as KeycloakAccessToken);

      // 3. Resolve organization (direct lookup by realm name)
      const organization = await getOrganizationByRealmName(tenantId);

      token.tenantId = tenantId;
      token.realmName = tenantId;  // Same value
      token.issuer = profile.iss;
      token.roles = roles;
      token.organization = organization;
      // NO organizations array (single org per realm)
    }
    return token;
  }
}
```

**Complexity**:
- 2 extraction functions (33% reduction)
- No nested loops (direct array access)
- Single claim source (`realm_access.roles` + `iss`)
- Simpler error handling

### Quantified Impact

| Metric | Current | Future | Change |
|--------|---------|--------|--------|
| **Extraction functions** | 3 | 2 | **-33%** |
| **Lines of code** | ~60 lines | ~25 lines | **-58%** |
| **Nested iterations** | Yes (resource_access loop) | No | **-100%** |
| **Claim sources** | 5 | 2 | **-60%** |
| **JWT callback complexity** | High (4 steps) | Medium (3 steps) | **-25%** |

**Code Deletion**:
- ❌ Delete `extractOrgGroups()` function
- ❌ Delete `resource_access` iteration logic
- ❌ Delete `tenantId` claim extraction (use issuer instead)

**New Code Required**:
- ✅ Add `extractTenantFromIssuer()` function (~10 lines)
- ✅ Add `getOrganizationByRealmName()` lookup (~15 lines)

---

## Impact 4: User Attribute Management

### Current Implementation (Single Realm)

**File**: [infra/kubernetes/charts/platform/keycloak-realm/values.yaml:467-481](infra/kubernetes/charts/platform/keycloak-realm/values.yaml#L467-L481)

```yaml
"users": [
  {
    "username": "admin",
    "email": "admin@ameide.io",
    "emailVerified": true,
    "enabled": true,
    "credentials": [
      {
        "type": "password",
        "value": "admin",
        "temporary": false
      }
    ],
    "attributes": {
      "tenantId": ["atlas-org"]  # ❌ Will be removed
    },
    "groups": [                  # ❌ Will be removed
      "/orgs/atlas"
    ],
    "realmRoles": [
      "admin"
    ],
    "clientRoles": {             # ❌ Will be removed
      "platform-app": []
    }
  },
  {
    "username": "user",
    "email": "user@ameide.io",
    "emailVerified": true,
    "enabled": true,
    "credentials": [
      {
        "type": "password",
        "value": "user",
        "temporary": false
      }
    ],
    "attributes": {
      "tenantId": ["atlas-org"]  # ❌ Will be removed
    },
    "groups": [                  # ❌ Will be removed
      "/orgs/atlas"
    ],
    "realmRoles": [
      "contributor"
    ]
  }
]
```

**Complexity per user**:
- `attributes.tenantId` - Custom attribute for tenant assignment
- `groups` - Array of group paths for organization membership
- `realmRoles` - Standard Keycloak realm roles
- `clientRoles` - Client-specific role assignments

### Future Implementation (Realm-Per-Tenant)

**File**: `infra/kubernetes/charts/platform/keycloak-realm-template/values.yaml` (per-org realm)

```yaml
# Realm: "atlas" (dedicated to Atlas organization)
"users": [
  {
    "username": "admin",
    "email": "admin@atlas.com",
    "emailVerified": true,
    "enabled": true,
    "credentials": [
      {
        "type": "password",
        "value": "${ADMIN_PASSWORD}",  # From secret
        "temporary": false
      }
    ],
    # NO attributes (no tenantId needed)
    # NO groups (realm membership = org membership)
    "realmRoles": [
      "admin"  # Simple role, no prefix
    ]
    # NO clientRoles (single app context)
  }
]

# OR federated from external IdP (no users defined in Keycloak at all)
"identityProviders": [
  {
    "alias": "atlas-okta",
    "providerId": "oidc",
    "enabled": true,
    "config": {
      "clientId": "${ATLAS_OKTA_CLIENT_ID}",
      "clientSecret": "${ATLAS_OKTA_CLIENT_SECRET}",
      "authorizationUrl": "https://atlas.okta.com/oauth2/v1/authorize",
      "tokenUrl": "https://atlas.okta.com/oauth2/v1/token"
    }
  }
]
```

**Complexity per user**:
- NO `attributes` (realm = tenant)
- NO `groups` (realm membership sufficient)
- `realmRoles` - Simpler role names
- NO `clientRoles` (single app)

### Quantified Impact

| Metric | Current (Single Realm) | Future (Per Realm) | Change |
|--------|------------------------|-------------------|--------|
| **User attributes** | 1 (`tenantId`) | 0 | **-100%** |
| **Group memberships** | 1+ (multiple orgs) | 0 | **-100%** |
| **Client roles** | Yes (per client) | No | **-100%** |
| **YAML lines per user** | ~25 lines | ~12 lines | **-52%** |

**OR: Federation Scenario**:
- **Users in Keycloak**: 2 (admin, user) → 0 (all federated) = **-100%**
- **Identity provider config**: ~20 lines (new)

**Benefits**:
- ✅ Simpler user management (fewer attributes)
- ✅ No group membership overhead
- ✅ Support for external IdP (Okta, Azure AD) per organization
- ✅ Organization admins control own realm

---

## Impact 5: Helm Chart Structure Changes

### Current Structure (Single Realm)

```
infra/kubernetes/charts/platform/
└── keycloak-realm/
    ├── Chart.yaml
    ├── values.yaml (1 file, ~600 lines)
    │   ├── realm: "ameide"
    │   ├── clientScopes: [roles, tenant, groups, organizations]
    │   ├── clients: [platform-app, threads-app, ...]
    │   ├── users: [admin, user]
    │   └── groups: [/orgs/atlas, /orgs/acme, ...]
    └── templates/
        └── keycloak-realm-import.yaml
```

**Deployment**:
```bash
# Single command deploys ONE realm for ALL organizations
helm upgrade keycloak-realm ./infra/kubernetes/charts/platform/keycloak-realm \
  -n ameide \
  -f ./infra/kubernetes/charts/platform/keycloak-realm/values.yaml
```

**Limitations**:
- All organizations share same realm configuration
- Cannot customize IdP per organization
- Changes affect ALL tenants simultaneously

### Future Structure (Realm-Per-Tenant)

```
infra/kubernetes/charts/platform/
├── keycloak-realm-template/          # NEW: Template chart
│   ├── Chart.yaml
│   ├── values.yaml (template with placeholders)
│   │   ├── realm: "{{ .Values.orgSlug }}"
│   │   ├── clientScopes: [roles]      # Simplified
│   │   ├── clients: [app]
│   │   ├── users: []                  # Usually federated
│   │   └── identityProviders: []      # Org-specific IdP
│   └── templates/
│       └── keycloak-realm-import.yaml
│
└── keycloak-realms/                   # NEW: Per-org instances
    ├── values-atlas.yaml
    │   └── orgSlug: "atlas"
    │       identityProviders:
    │         - alias: "atlas-okta"
    │           config: {...}
    ├── values-acme.yaml
    │   └── orgSlug: "acme"
    │       identityProviders:
    │         - alias: "acme-azure-ad"
    │           config: {...}
    └── helmfile.yaml                  # Manages multiple realm releases
```

**Deployment (Helmfile)**:
```yaml
# infra/kubernetes/charts/platform/keycloak-realms/helmfile.yaml
releases:
  - name: keycloak-realm-atlas
    chart: ../keycloak-realm-template
    namespace: ameide
    values:
      - values-atlas.yaml

  - name: keycloak-realm-acme
    chart: ../keycloak-realm-template
    namespace: ameide
    values:
      - values-acme.yaml
```

```bash
# Single command deploys MULTIPLE realms (one per org)
helmfile sync
```

**OR: Dynamic Realm Creation** (Preferred for scale):
```typescript
// services/platform/src/organizations/service.ts
async createOrganization(request: CreateOrganizationRequest) {
  // 1. Insert organization into database
  const org = await db.organizations.insert({
    slug: request.slug,
    name: request.name,
  });

  // 2. Create dedicated Keycloak realm via Admin API
  await keycloakAdmin.createRealm({
    realm: request.slug,
    enabled: true,
    displayName: request.name,
  });

  // 3. Configure realm (apply template)
  await keycloakAdmin.importRealmConfig(request.slug, realmTemplate);

  return org;
}
```

### Quantified Impact

| Metric | Current | Future (Helmfile) | Future (Dynamic) | Change |
|--------|---------|-------------------|------------------|--------|
| **Helm releases** | 1 | 1 per org | 1 template | 1 → N |
| **Values files** | 1 | 1 per org | Template only | 1 → N |
| **Deployment commands** | 1 | 1 (helmfile) | API call | Same |
| **Runtime realm creation** | No | No | Yes | NEW |

**Static Approach (Helmfile)**:
- ✅ GitOps-friendly (realms in version control)
- ✅ Easy rollback via Helm
- ❌ Requires Helm release per org (doesn't scale to 1000s of orgs)

**Dynamic Approach (Keycloak Admin API)**:
- ✅ Scales to unlimited organizations
- ✅ Realms created on-demand during org creation
- ✅ Supports self-service org provisioning
- ❌ Not managed by GitOps (harder to audit)

**Recommended**: Hybrid approach
- Core platform realms via Helmfile (predictable, few orgs)
- Customer organization realms via Admin API (scalable, many orgs)

---

## Impact 6: Integration with Platform Services

### Current Service Configuration (Single Realm)

**File**: [services/www_ameide_platform/app/(auth)/auth.ts:45-55](services/www_ameide_platform/app/(auth)/auth.ts#L45-L55)

```typescript
// NextAuth configuration (single realm)
export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Keycloak({
      clientId: process.env.AUTH_KEYCLOAK_CLIENT_ID!,
      clientSecret: process.env.AUTH_KEYCLOAK_CLIENT_SECRET!,
      issuer: process.env.AUTH_KEYCLOAK_ISSUER!,  // Single issuer: https://keycloak/realms/ameide
    }),
  ],
  // ...
});
```

**Environment Variables** (Single Realm):
```bash
# .env
AUTH_KEYCLOAK_CLIENT_ID=platform-app
AUTH_KEYCLOAK_CLIENT_SECRET=changeme
AUTH_KEYCLOAK_ISSUER=https://auth.dev.ameide.io/realms/ameide  # Hardcoded realm
```

**Problem**:
- Hardcoded to single realm ("ameide")
- Cannot authenticate users from different organization realms

### Future Service Configuration (Multi-Realm)

**Option 1: Dynamic Issuer Resolution** (Recommended)

**File**: `services/www_ameide_platform/app/(auth)/auth.ts` (modified)

```typescript
// NextAuth with dynamic realm resolution
export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    {
      id: "keycloak",
      name: "Keycloak",
      type: "oidc",

      // Dynamic well-known endpoint based on organization
      async wellKnown(options) {
        const orgSlug = await resolveOrgSlugFromRequest(options.request);
        return `https://keycloak/realms/${orgSlug}/.well-known/openid-configuration`;
      },

      // Dynamic token endpoint
      async token(options) {
        const orgSlug = await resolveOrgSlugFromToken(options.token);
        const response = await fetch(`https://keycloak/realms/${orgSlug}/protocol/openid-connect/token`, {
          method: "POST",
          headers: { "Content-Type": "application/x-www-form-urlencoded" },
          body: new URLSearchParams(options.params),
        });
        return response.json();
      },

      clientId: process.env.AUTH_KEYCLOAK_CLIENT_ID!,
      clientSecret: process.env.AUTH_KEYCLOAK_CLIENT_SECRET!,
    },
  ],
  // ...
});
```

**Helper Functions**:
```typescript
// Resolve organization from request (e.g., subdomain or path)
async function resolveOrgSlugFromRequest(request: Request): Promise<string> {
  const url = new URL(request.url);

  // Option A: Subdomain-based
  // https://atlas.platform.ameide.com → "atlas"
  const subdomain = url.hostname.split('.')[0];
  if (subdomain !== 'platform' && subdomain !== 'www') {
    return subdomain;
  }

  // Option B: Path-based
  // https://platform.ameide.com/org/atlas → "atlas"
  const pathMatch = url.pathname.match(/^\/org\/([^/]+)/);
  if (pathMatch) {
    return pathMatch[1];
  }

  throw new Error('Cannot determine organization from request');
}
```

**Environment Variables** (Multi-Realm):
```bash
# .env
AUTH_KEYCLOAK_CLIENT_ID=app            # Same client name across all realms
AUTH_KEYCLOAK_CLIENT_SECRET=changeme   # Same secret (or per-realm secrets from vault)
AUTH_KEYCLOAK_BASE_URL=https://auth.dev.ameide.io  # Base URL (no realm)
```

**Option 2: Multiple NextAuth Providers** (Alternative)

```typescript
export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Keycloak({
      id: "keycloak-atlas",
      clientId: "app",
      clientSecret: process.env.ATLAS_KEYCLOAK_SECRET!,
      issuer: "https://keycloak/realms/atlas",
    }),
    Keycloak({
      id: "keycloak-acme",
      clientId: "app",
      clientSecret: process.env.ACME_KEYCLOAK_SECRET!,
      issuer: "https://keycloak/realms/acme",
    }),
    // ... one provider per organization
  ],
});
```

**Problem with Option 2**:
- ❌ Doesn't scale (need to add provider for each new org)
- ❌ Requires application restart to add new organizations
- ❌ Not suitable for self-service org provisioning

**Recommended**: Option 1 (dynamic issuer resolution)

### Quantified Impact

| Metric | Current | Future (Dynamic) | Change |
|--------|---------|------------------|--------|
| **Issuers** | 1 (hardcoded) | N (dynamic) | **Dynamic** |
| **NextAuth providers** | 1 | 1 (multi-realm) | **Same** |
| **Environment variables** | 3 | 2 | **-33%** |
| **Realm resolution logic** | None | ~30 lines | **+30 LOC** |
| **Supports new orgs without redeploy** | No | Yes | **✅ NEW** |

---

## Impact 7: Testing Strategy Changes

### Current Test Approach (Single Realm)

**Integration Tests**: Mock single Keycloak issuer

```typescript
// services/www_ameide_platform/__tests__/integration/auth.test.ts
describe('Authentication', () => {
  it('should extract roles from JWT', () => {
    const token = {
      iss: "https://keycloak/realms/ameide",  // Hardcoded
      realm_access: { roles: ["admin"] },
      resource_access: { "platform-app": { roles: [] } },
      tenantId: "atlas-org",
      groups: ["/orgs/atlas"],
    };

    const roles = extractRoles(token);
    expect(roles).toContain("admin");
  });
});
```

**E2E Tests**: Single realm authentication flow

```typescript
// features/navigation/__tests__/e2e/auth.spec.ts
test('admin can access all tabs', async ({ page }) => {
  // Login with admin credentials (single realm)
  await page.goto('https://platform.dev.ameide.io/auth/signin');
  await page.fill('[name=username]', 'admin');
  await page.fill('[name=password]', 'admin');
  await page.click('button[type=submit]');

  await expect(page).toHaveURL('/org/atlas');
});
```

### Future Test Approach (Multi-Realm)

**Integration Tests**: Mock multiple Keycloak issuers

```typescript
// services/www_ameide_platform/__tests__/integration/auth-multi-realm.test.ts
describe('Multi-Realm Authentication', () => {
  describe('Atlas Organization', () => {
    it('should extract tenant from issuer', () => {
      const token = {
        iss: "https://keycloak/realms/atlas",  // Realm = Org
        realm_access: { roles: ["admin"] },
        // NO tenantId claim
        // NO groups claim
      };

      const tenant = extractTenantFromIssuer(token.iss);
      expect(tenant).toBe("atlas");
    });

    it('should extract simple roles', () => {
      const token = {
        iss: "https://keycloak/realms/atlas",
        realm_access: { roles: ["admin", "contributor"] },
      };

      const roles = extractRoles(token);
      expect(roles).toEqual(["admin", "contributor"]);
      expect(roles).not.toContain("org:atlas:admin");  // No prefixes
    });
  });

  describe('ACME Organization', () => {
    it('should isolate ACME realm tokens', () => {
      const token = {
        iss: "https://keycloak/realms/acme",  // Different realm
        realm_access: { roles: ["viewer"] },
      };

      const tenant = extractTenantFromIssuer(token.iss);
      expect(tenant).toBe("acme");
      expect(tenant).not.toBe("atlas");  // Isolation check
    });
  });

  describe('Cross-Realm Isolation', () => {
    it('should reject tokens from wrong realm', async () => {
      const atlasToken = createMockToken({ iss: "https://keycloak/realms/atlas" });
      const acmeOrg = { id: "acme", realmName: "acme" };

      // Attempt to use Atlas token for ACME organization
      await expect(validateTokenForOrg(atlasToken, acmeOrg)).rejects.toThrow(
        'Token issuer does not match organization realm'
      );
    });
  });
});
```

**E2E Tests**: Multi-realm authentication flows

```typescript
// features/auth/__tests__/e2e/multi-realm.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Multi-Realm Authentication', () => {
  test('Atlas user can login to Atlas realm', async ({ page }) => {
    // Navigate to Atlas organization
    await page.goto('https://platform.dev.ameide.io/org/atlas');

    // Should redirect to Atlas-specific Keycloak realm
    await expect(page).toHaveURL(/realms\/atlas/);

    // Login as Atlas admin
    await page.fill('[name=username]', 'atlas-admin');
    await page.fill('[name=password]', 'password');
    await page.click('button[type=submit]');

    // Should redirect back to Atlas organization
    await expect(page).toHaveURL('/org/atlas');

    // Verify JWT issuer is Atlas realm
    const session = await page.evaluate(() => {
      return fetch('/api/auth/session').then(r => r.json());
    });
    expect(session.user.issuer).toContain('realms/atlas');
  });

  test('ACME user cannot access Atlas realm', async ({ page }) => {
    // Login to ACME realm
    await page.goto('https://platform.dev.ameide.io/org/acme');
    await page.fill('[name=username]', 'acme-user');
    await page.fill('[name=password]', 'password');
    await page.click('button[type=submit]');

    // Verify logged into ACME
    await expect(page).toHaveURL('/org/acme');

    // Attempt to access Atlas organization
    await page.goto('https://platform.dev.ameide.io/org/atlas');

    // Should be denied or redirected to re-authenticate
    await expect(page).toHaveURL(/\/auth\/signin|\/org\/acme/);
    await expect(page.locator('text=Access Denied')).toBeVisible();
  });

  test('Guest user from Atlas can access ACME (B2B)', async ({ page }) => {
    // Atlas user invited as guest to ACME
    await page.goto('https://platform.dev.ameide.io/org/acme/invite/abc123');

    // Should redirect to ACME realm with guest flow
    await expect(page).toHaveURL(/realms\/acme.*guest/);

    // Login with Atlas credentials (federated)
    await page.click('text=Sign in with Atlas');
    await page.fill('[name=username]', 'atlas-user@atlas.com');
    await page.fill('[name=password]', 'password');
    await page.click('button[type=submit]');

    // Should have guest role in ACME
    const session = await page.evaluate(() => {
      return fetch('/api/auth/session').then(r => r.json());
    });
    expect(session.user.roles).toContain('guest');
    expect(session.user.organization.slug).toBe('acme');
  });
});
```

**Infrastructure Tests**: Keycloak realm provisioning

```typescript
// services/platform/__tests__/integration/keycloak-realms.test.ts
describe('Keycloak Realm Provisioning', () => {
  let keycloakAdmin: KeycloakAdminClient;

  beforeAll(async () => {
    keycloakAdmin = await getKeycloakAdminClient();
  });

  it('should create realm when organization is created', async () => {
    const orgSlug = 'test-org-' + Date.now();

    // Create organization via Platform API
    const org = await platformClient.createOrganization({
      slug: orgSlug,
      name: 'Test Organization',
    });

    // Verify Keycloak realm was created
    const realm = await keycloakAdmin.realms.findOne({ realm: orgSlug });
    expect(realm).toBeDefined();
    expect(realm.realm).toBe(orgSlug);
    expect(realm.enabled).toBe(true);
  });

  it('should configure realm with correct client scopes', async () => {
    const realm = await keycloakAdmin.realms.findOne({ realm: 'atlas' });
    const scopes = await keycloakAdmin.clientScopes.find({ realm: 'atlas' });

    // Should have simplified scopes
    const scopeNames = scopes.map(s => s.name);
    expect(scopeNames).toContain('roles');
    expect(scopeNames).not.toContain('tenant');    // ❌ Removed
    expect(scopeNames).not.toContain('groups');    // ❌ Removed
    expect(scopeNames).not.toContain('organizations');  // ❌ Removed
  });

  afterAll(async () => {
    // Cleanup test realms
    await keycloakAdmin.realms.del({ realm: 'test-org-*' });
  });
});
```

### Quantified Impact

| Test Type | Current Tests | New Tests Needed | Change |
|-----------|---------------|------------------|--------|
| **Integration (JWT)** | 5 | +8 (multi-realm) | **+160%** |
| **E2E (Auth flows)** | 3 | +6 (cross-realm) | **+200%** |
| **Infrastructure (Keycloak)** | 0 | +4 (provisioning) | **NEW** |
| **Total test code** | ~200 lines | ~650 lines | **+225%** |

**Test Infrastructure Changes**:
- ✅ Add `@testcontainers/keycloak` for isolated testing
- ✅ Add Keycloak Admin API test helpers
- ✅ Add multi-realm test fixtures
- ✅ Update Playwright config for realm-specific URLs

---

## Remediation Plan (Step-by-Step)

### Phase 1: Keycloak Configuration Template (Week 1, Days 1-3)

#### Step 1.1: Create Realm Template Chart (Day 1)

**Task**: Create Helm chart template for per-organization realms

**Files to create**:
```bash
mkdir -p infra/kubernetes/charts/platform/keycloak-realm-template
```

**File**: `infra/kubernetes/charts/platform/keycloak-realm-template/Chart.yaml`
```yaml
apiVersion: v2
name: keycloak-realm-template
description: Keycloak realm template for per-organization isolation
version: 1.0.0
appVersion: "22.0"
```

**File**: `infra/kubernetes/charts/platform/keycloak-realm-template/values.yaml`
```yaml
# Organization-specific values (provided at install time)
orgSlug: ""          # REQUIRED: Organization slug (becomes realm name)
orgName: ""          # REQUIRED: Organization display name
orgId: ""            # REQUIRED: Organization UUID from database
adminEmail: ""       # REQUIRED: Initial admin user email

# Realm configuration (simplified)
realm:
  enabled: true
  displayName: "{{ .Values.orgName }}"

  clientScopes:
    - name: roles
      protocolMappers:
        - name: realm roles
          protocolMapper: oidc-usermodel-realm-role-mapper
          config:
            claim.name: realm_access.roles

  clients:
    - clientId: app
      name: "{{ .Values.orgName }} Application"
      enabled: true
      clientAuthenticatorType: client-secret
      secret: changeme  # Override with secret from vault
      standardFlowEnabled: true
      directAccessGrantsEnabled: true
      publicClient: false
      redirectUris:
        - "https://platform.dev.ameide.io/*"
        - "https://{{ .Values.orgSlug }}.platform.dev.ameide.io/*"
      defaultClientScopes:
        - openid
        - profile
        - email
        - roles

  # Identity provider (optional, for SSO)
  identityProviders: []

  # Users (usually empty, managed by IdP)
  users: []
```

**File**: `infra/kubernetes/charts/platform/keycloak-realm-template/templates/keycloak-realm-import.yaml`
```yaml
apiVersion: k8s.keycloak.org/v2alpha1
kind: KeycloakRealmImport
metadata:
  name: realm-{{ .Values.orgSlug }}
  namespace: {{ .Release.Namespace }}
spec:
  keycloakCRName: keycloak
  realm:
    realm: {{ .Values.orgSlug }}
    enabled: {{ .Values.realm.enabled }}
    displayName: {{ .Values.realm.displayName }}

    # Simplified client scopes (only roles)
    clientScopes:
    {{- range .Values.realm.clientScopes }}
      - name: {{ .name }}
        protocol: openid-connect
        protocolMappers:
        {{- range .protocolMappers }}
          - name: {{ .name }}
            protocol: openid-connect
            protocolMapper: {{ .protocolMapper }}
            config:
            {{- range $key, $value := .config }}
              {{ $key }}: {{ $value | quote }}
            {{- end }}
        {{- end }}
    {{- end }}

    # Clients
    clients:
    {{- range .Values.realm.clients }}
      - clientId: {{ .clientId }}
        name: {{ .name }}
        enabled: {{ .enabled }}
        clientAuthenticatorType: {{ .clientAuthenticatorType }}
        secret: {{ .secret }}
        standardFlowEnabled: {{ .standardFlowEnabled }}
        directAccessGrantsEnabled: {{ .directAccessGrantsEnabled }}
        publicClient: {{ .publicClient }}
        redirectUris:
        {{- range .redirectUris }}
          - {{ . }}
        {{- end }}
        defaultClientScopes:
        {{- range .defaultClientScopes }}
          - {{ . }}
        {{- end }}
    {{- end }}

    # Identity providers (if configured)
    {{- if .Values.realm.identityProviders }}
    identityProviders:
    {{- range .Values.realm.identityProviders }}
      - alias: {{ .alias }}
        providerId: {{ .providerId }}
        enabled: {{ .enabled }}
        config:
        {{- range $key, $value := .config }}
          {{ $key }}: {{ $value | quote }}
        {{- end }}
    {{- end }}
    {{- end }}

    # Users (if any)
    {{- if .Values.realm.users }}
    users:
    {{- range .Values.realm.users }}
      - username: {{ .username }}
        email: {{ .email }}
        emailVerified: {{ .emailVerified }}
        enabled: {{ .enabled }}
        realmRoles:
        {{- range .realmRoles }}
          - {{ . }}
        {{- end }}
    {{- end }}
    {{- end }}
```

**Testing**:
```bash
# Validate template
helm template test-realm ./infra/kubernetes/charts/platform/keycloak-realm-template \
  --set orgSlug=testorg \
  --set orgName="Test Organization" \
  --set orgId=test-org-uuid \
  --set adminEmail=admin@testorg.com

# Should output valid KeycloakRealmImport YAML
```

**Success Criteria**:
- ✅ Template renders valid Kubernetes YAML
- ✅ Realm name is parameterized (orgSlug)
- ✅ Only `roles` client scope included
- ✅ No `tenant`, `groups`, or `organizations` scopes

#### Step 1.2: Test Template with Single Organization (Day 2)

**Task**: Deploy template for Atlas organization

**File**: `infra/kubernetes/charts/platform/keycloak-realms/values-atlas.yaml`
```yaml
orgSlug: atlas
orgName: Atlas Organization
orgId: atlas-org-uuid-from-db
adminEmail: admin@atlas.com

realm:
  users:
    - username: admin
      email: admin@atlas.com
      emailVerified: true
      enabled: true
      realmRoles:
        - admin
```

**Deploy**:
```bash
helm install keycloak-realm-atlas \
  ./infra/kubernetes/charts/platform/keycloak-realm-template \
  -n ameide \
  -f ./infra/kubernetes/charts/platform/keycloak-realms/values-atlas.yaml
```

**Verification**:
```bash
# Check KeycloakRealmImport status
kubectl get keycloakrealmimport realm-atlas -n ameide -o jsonpath='{.status.conditions[?(@.type=="Done")].status}'
# Expected: "True"

# Verify realm exists in Keycloak
curl -k -s -X POST https://auth.dev.ameide.io/realms/ameide/protocol/openid-connect/token \
  --data-urlencode "username=admin" \
  --data-urlencode "password=admin" \
  --data-urlencode "grant_type=password" \
  --data-urlencode "client_id=platform-app" \
  --data-urlencode "client_secret=changeme" | jq -r '.access_token' | \
  cut -d'.' -f2 | base64 -d | jq

# Get admin token
ADMIN_TOKEN=$(curl -k -s -X POST https://auth.dev.ameide.io/realms/master/protocol/openid-connect/token \
  --data-urlencode "username=admin" \
  --data-urlencode "password=admin" \
  --data-urlencode "grant_type=password" \
  --data-urlencode "client_id=admin-cli" | jq -r '.access_token')

# List realms
curl -k -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  https://auth.dev.ameide.io/admin/realms | jq '.[] | {realm: .realm, enabled: .enabled}'

# Expected output should include:
# {
#   "realm": "atlas",
#   "enabled": true
# }

# Test login to Atlas realm
curl -k -X POST https://auth.dev.ameide.io/realms/atlas/protocol/openid-connect/token \
  --data-urlencode "username=admin" \
  --data-urlencode "password=admin" \
  --data-urlencode "grant_type=password" \
  --data-urlencode "client_id=app" \
  --data-urlencode "client_secret=changeme"
```

**Success Criteria**:
- ✅ Realm "atlas" created successfully
- ✅ JWT issuer is `https://keycloak/realms/atlas`
- ✅ JWT contains `realm_access.roles: ["admin"]`
- ✅ JWT does NOT contain `tenantId`, `groups`, or `org_groups` claims

#### Step 1.3: Document Configuration Differences (Day 3)

**Task**: Create migration guide comparing old vs new configuration

**File**: `docs/architecture/keycloak-realm-migration.md` (NEW)

```markdown
# Keycloak Configuration Migration Guide

## Configuration Comparison

### Before (Single Realm)
- Realm: "ameide" (shared by all organizations)
- Client Scopes: 4 (roles, tenant, groups, organizations)
- JWT Claims: 5 custom claims
- User Attributes: tenantId
- Group Membership: /orgs/{slug}

### After (Realm-Per-Tenant)
- Realm: "{org-slug}" (one per organization)
- Client Scopes: 1 (roles only)
- JWT Claims: 1 custom claim
- User Attributes: None needed
- Group Membership: Not used (realm membership = org membership)

## Migration Steps
1. Deploy realm template chart
2. Create per-org realm instances
3. Update NextAuth configuration for multi-realm
4. Test authentication flows
5. Migrate users (or configure IdP federation)
6. Deprecate single realm

## Rollback Plan
- Keep single realm active during migration
- Gradual cutover per organization
- Fallback to single realm if issues occur
```

**Commit changes**:
```bash
git add infra/kubernetes/charts/platform/keycloak-realm-template/
git add infra/kubernetes/charts/platform/keycloak-realms/
git add docs/architecture/keycloak-realm-migration.md
git commit -m "feat(keycloak): add realm-per-tenant configuration template"
```

---

### Phase 2: Service Integration (Week 1, Days 4-5 + Week 2, Days 1-2)

#### Step 2.1: Implement Multi-Realm NextAuth Provider (Week 1, Day 4)

**Task**: Update NextAuth to support dynamic realm resolution

**File**: `services/www_ameide_platform/lib/auth/multi-realm.ts` (NEW)

```typescript
/**
 * Multi-realm authentication support
 *
 * Resolves Keycloak realm dynamically based on organization context
 */

import { OIDCConfig } from 'next-auth/providers/oidc';

export interface RealmResolutionStrategy {
  resolveRealmFromRequest(request: Request): Promise<string>;
}

/**
 * Path-based realm resolution
 * Example: /org/atlas → realm "atlas"
 */
export class PathBasedRealmResolution implements RealmResolutionStrategy {
  async resolveRealmFromRequest(request: Request): Promise<string> {
    const url = new URL(request.url);
    const pathMatch = url.pathname.match(/^\/org\/([^/]+)/);

    if (!pathMatch) {
      throw new Error('Cannot determine organization from path');
    }

    return pathMatch[1];  // Org slug = realm name
  }
}

/**
 * Subdomain-based realm resolution
 * Example: atlas.platform.ameide.com → realm "atlas"
 */
export class SubdomainBasedRealmResolution implements RealmResolutionStrategy {
  async resolveRealmFromRequest(request: Request): Promise<string> {
    const url = new URL(request.url);
    const subdomain = url.hostname.split('.')[0];

    if (subdomain === 'platform' || subdomain === 'www' || subdomain === 'localhost') {
      // Fallback to path-based
      const pathStrategy = new PathBasedRealmResolution();
      return pathStrategy.resolveRealmFromRequest(request);
    }

    return subdomain;
  }
}

/**
 * Create dynamic Keycloak provider with realm resolution
 */
export function createMultiRealmKeycloakProvider(
  strategy: RealmResolutionStrategy
): OIDCConfig<any> {
  return {
    id: 'keycloak',
    name: 'Keycloak',
    type: 'oidc',

    async wellKnown(context) {
      const realm = await strategy.resolveRealmFromRequest(context.request);
      const baseUrl = process.env.AUTH_KEYCLOAK_BASE_URL!;
      return `${baseUrl}/realms/${realm}/.well-known/openid-configuration`;
    },

    async authorization(context) {
      const realm = await strategy.resolveRealmFromRequest(context.request);
      const baseUrl = process.env.AUTH_KEYCLOAK_BASE_URL!;
      return {
        url: `${baseUrl}/realms/${realm}/protocol/openid-connect/auth`,
        params: {
          scope: 'openid profile email roles',
        },
      };
    },

    async token(context) {
      const realm = await strategy.resolveRealmFromRequest(context.request);
      const baseUrl = process.env.AUTH_KEYCLOAK_BASE_URL!;
      const response = await fetch(`${baseUrl}/realms/${realm}/protocol/openid-connect/token`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
        },
        body: new URLSearchParams({
          ...context.params,
          client_id: process.env.AUTH_KEYCLOAK_CLIENT_ID!,
          client_secret: process.env.AUTH_KEYCLOAK_CLIENT_SECRET!,
        }),
      });

      return response.json();
    },

    async userinfo(context) {
      const realm = await extractRealmFromIssuer(context.tokens.id_token);
      const baseUrl = process.env.AUTH_KEYCLOAK_BASE_URL!;
      const response = await fetch(`${baseUrl}/realms/${realm}/protocol/openid-connect/userinfo`, {
        headers: {
          Authorization: `Bearer ${context.tokens.access_token}`,
        },
      });

      return response.json();
    },

    clientId: process.env.AUTH_KEYCLOAK_CLIENT_ID!,
    clientSecret: process.env.AUTH_KEYCLOAK_CLIENT_SECRET!,
  };
}

/**
 * Extract realm name from JWT issuer
 * Example: "https://keycloak/realms/atlas" → "atlas"
 */
export function extractRealmFromIssuer(issuer: string): string {
  const match = issuer.match(/\/realms\/([^/]+)$/);
  if (!match) {
    throw new Error(`Invalid Keycloak issuer format: ${issuer}`);
  }
  return match[1];
}
```

**Update**: `services/www_ameide_platform/app/(auth)/auth.ts`

```typescript
import { createMultiRealmKeycloakProvider, PathBasedRealmResolution } from '@/lib/auth/multi-realm';

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    createMultiRealmKeycloakProvider(
      new PathBasedRealmResolution()  // Use path-based realm resolution
    ),
  ],
  callbacks: {
    async jwt({ token, account, profile }) {
      if (account && profile) {
        // Extract realm from issuer (replaces tenantId extraction)
        const realmName = extractRealmFromIssuer(profile.iss);

        // Extract roles (simplified, no resource_access)
        const roles = profile.realm_access?.roles ?? [];

        // Resolve organization by realm name (direct lookup)
        const organization = await getOrganizationByRealmName(realmName);

        token.realmName = realmName;
        token.issuer = profile.iss;
        token.roles = roles;
        token.organization = organization;
        token.tenantId = organization?.id;  // Backward compatibility
      }
      return token;
    },

    async session({ session, token }) {
      if (token) {
        session.user.realmName = token.realmName as string;
        session.user.issuer = token.issuer as string;
        session.user.roles = token.roles as string[];
        session.user.organization = token.organization as OrganizationContext;
        session.user.tenantId = token.tenantId as string;
      }
      return session;
    },
  },
});
```

**Environment Variables**:
```bash
# .env (updated)
AUTH_KEYCLOAK_BASE_URL=https://auth.dev.ameide.io  # Base URL, no realm
AUTH_KEYCLOAK_CLIENT_ID=app
AUTH_KEYCLOAK_CLIENT_SECRET=changeme
```

**Testing**:
```bash
# Start dev server
pnpm dev

# Test authentication to Atlas realm
curl -X POST http://localhost:3000/auth/signin/keycloak \
  -H "Content-Type: application/json" \
  -d '{"callbackUrl": "/org/atlas"}'

# Should redirect to: https://auth.dev.ameide.io/realms/atlas/protocol/openid-connect/auth
```

#### Step 2.2: Update JWT Validation Logic (Week 1, Day 5)

**Task**: Simplify role extraction and add issuer validation

**File**: `services/www_ameide_platform/lib/auth/jwt.ts` (NEW simplified version)

```typescript
import { KeycloakAccessToken } from '@/types/auth';

/**
 * Extract roles from JWT (simplified for realm-per-tenant)
 */
export function extractRoles(token: KeycloakAccessToken): string[] {
  // Only extract realm_access.roles (no resource_access)
  return token.realm_access?.roles ?? [];
}

/**
 * Extract tenant ID from JWT issuer
 */
export function extractTenantFromIssuer(issuer: string): string {
  const match = issuer.match(/\/realms\/([^/]+)$/);
  if (!match) {
    throw new Error(`Invalid Keycloak issuer format: ${issuer}`);
  }
  return match[1];  // Realm name = org slug = tenant ID
}

/**
 * Validate JWT issuer matches expected organization
 */
export function validateTokenForOrganization(
  token: KeycloakAccessToken,
  organization: { id: string; realmName: string }
): void {
  const tokenRealm = extractTenantFromIssuer(token.iss);

  if (tokenRealm !== organization.realmName) {
    throw new Error(
      `Token issuer realm "${tokenRealm}" does not match organization realm "${organization.realmName}"`
    );
  }
}
```

**Update**: Delete old extraction logic

```bash
# Remove old complex extraction functions
git rm services/www_ameide_platform/lib/keycloak.ts  # Contains old extractRoles with resource_access
```

**Testing**:
```typescript
// services/www_ameide_platform/__tests__/unit/jwt.test.ts
import { extractRoles, extractTenantFromIssuer, validateTokenForOrganization } from '@/lib/auth/jwt';

describe('JWT Utilities (Realm-Per-Tenant)', () => {
  describe('extractRoles', () => {
    it('should extract realm roles', () => {
      const token = {
        iss: 'https://keycloak/realms/atlas',
        realm_access: { roles: ['admin', 'contributor'] },
      } as KeycloakAccessToken;

      expect(extractRoles(token)).toEqual(['admin', 'contributor']);
    });

    it('should return empty array if no roles', () => {
      const token = {
        iss: 'https://keycloak/realms/atlas',
      } as KeycloakAccessToken;

      expect(extractRoles(token)).toEqual([]);
    });
  });

  describe('extractTenantFromIssuer', () => {
    it('should extract realm name from issuer', () => {
      expect(extractTenantFromIssuer('https://keycloak/realms/atlas')).toBe('atlas');
      expect(extractTenantFromIssuer('https://keycloak/realms/acme')).toBe('acme');
    });

    it('should throw on invalid issuer format', () => {
      expect(() => extractTenantFromIssuer('https://keycloak/invalid')).toThrow();
    });
  });

  describe('validateTokenForOrganization', () => {
    it('should pass when token realm matches org realm', () => {
      const token = { iss: 'https://keycloak/realms/atlas' } as KeycloakAccessToken;
      const org = { id: 'atlas-id', realmName: 'atlas' };

      expect(() => validateTokenForOrganization(token, org)).not.toThrow();
    });

    it('should throw when token realm does not match org realm', () => {
      const token = { iss: 'https://keycloak/realms/atlas' } as KeycloakAccessToken;
      const org = { id: 'acme-id', realmName: 'acme' };

      expect(() => validateTokenForOrganization(token, org)).toThrow(
        'Token issuer realm "atlas" does not match organization realm "acme"'
      );
    });
  });
});
```

```bash
# Run tests
pnpm test lib/auth/jwt.test.ts
```

#### Step 2.3: Add Organization-Realm Mapping (Week 2, Day 1)

**Task**: Update database schema and queries to include realm names

**File**: `db/flyway/sql/V38__add_realm_name_to_organizations.sql` (NEW)

```sql
-- Add realm_name column to organizations table
ALTER TABLE organizations
ADD COLUMN realm_name VARCHAR(255);

-- Create unique index (realm names must be unique across platform)
CREATE UNIQUE INDEX idx_organizations_realm_name ON organizations(realm_name);

-- Backfill existing organizations with realm names (slug = realm name)
UPDATE organizations
SET realm_name = slug;

-- Make realm_name NOT NULL after backfill
ALTER TABLE organizations
ALTER COLUMN realm_name SET NOT NULL;

-- Add comment
COMMENT ON COLUMN organizations.realm_name IS 'Keycloak realm name for this organization';
```

**File**: `packages/ameide_core_proto/src/ameide_core_proto/platform/v1/organizations.proto`

```protobuf
message Organization {
  string id = 1;
  string slug = 2;
  string name = 3;
  string realm_name = 4;  // NEW: Keycloak realm name

  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}

message GetOrganizationByRealmNameRequest {
  string realm_name = 1;
}

message GetOrganizationByRealmNameResponse {
  Organization organization = 1;
}
```

**Regenerate Protobuf**:
```bash
cd packages/ameide_core_proto
buf generate src
```

**File**: `services/platform/src/organizations/service.ts` (add new RPC)

```typescript
async getOrganizationByRealmName(
  request: platformOrganizations.GetOrganizationByRealmNameRequest
): Promise<platformOrganizations.GetOrganizationByRealmNameResponse> {
  const org = await this.db.organizations.findFirst({
    where: { realmName: request.realmName },
  });

  if (!org) {
    throw new Error(`Organization with realm "${request.realmName}" not found`);
  }

  return create(platformOrganizations.GetOrganizationByRealmNameResponseSchema, {
    organization: mapOrganizationToProto(org),
  });
}
```

**File**: `services/www_ameide_platform/lib/sdk/organizations.ts` (add helper)

```typescript
export async function getOrganizationByRealmName(
  realmName: string,
  client?: AmeideClient
): Promise<Organization> {
  const svc = client ?? getServerClient();

  const response = await svc.platformOrganizations.getOrganizationByRealmName({
    realmName,
  });

  return response.organization;
}
```

**Migration**:
```bash
# Apply migration
pnpm migrate

# Verify
kubectl exec -n ameide postgres-platform-1 -- \
  psql -U postgres -d platform -c \
  "SELECT id, slug, realm_name FROM organizations;"
```

#### Step 2.4: Update Organization Creation Flow (Week 2, Day 2)

**Task**: Auto-create Keycloak realm when organization is created

**File**: `services/platform/src/keycloak/admin.ts` (NEW)

```typescript
import KcAdminClient from '@keycloak/keycloak-admin-client';

interface RealmConfig {
  realm: string;
  displayName: string;
  enabled: boolean;
}

export class KeycloakAdminService {
  private client: KcAdminClient;

  constructor() {
    this.client = new KcAdminClient({
      baseUrl: process.env.KEYCLOAK_BASE_URL!,
      realmName: 'master',
    });
  }

  async authenticate() {
    await this.client.auth({
      username: process.env.KEYCLOAK_ADMIN_USER!,
      password: process.env.KEYCLOAK_ADMIN_PASSWORD!,
      grantType: 'password',
      clientId: 'admin-cli',
    });
  }

  async createRealm(config: RealmConfig): Promise<void> {
    await this.authenticate();

    // Load realm template
    const template = await this.loadRealmTemplate();

    // Render template with org-specific values
    const realmConfig = {
      ...template,
      realm: config.realm,
      displayName: config.displayName,
      enabled: config.enabled,
    };

    // Create realm via Admin API
    await this.client.realms.create(realmConfig);

    console.log(`[KeycloakAdmin] Created realm: ${config.realm}`);
  }

  async deleteRealm(realmName: string): Promise<void> {
    await this.authenticate();
    await this.client.realms.del({ realm: realmName });
    console.log(`[KeycloakAdmin] Deleted realm: ${realmName}`);
  }

  private async loadRealmTemplate(): Promise<any> {
    // Load template from file or inline
    return {
      enabled: true,
      clientScopes: [
        {
          name: 'roles',
          protocol: 'openid-connect',
          protocolMappers: [
            {
              name: 'realm roles',
              protocol: 'openid-connect',
              protocolMapper: 'oidc-usermodel-realm-role-mapper',
              config: {
                'claim.name': 'realm_access.roles',
                'jsonType.label': 'String',
                'multivalued': 'true',
                'access.token.claim': 'true',
                'id.token.claim': 'true',
                'userinfo.token.claim': 'true',
              },
            },
          ],
        },
      ],
      clients: [
        {
          clientId: 'app',
          enabled: true,
          clientAuthenticatorType: 'client-secret',
          secret: process.env.KEYCLOAK_CLIENT_SECRET!,
          standardFlowEnabled: true,
          directAccessGrantsEnabled: true,
          redirectUris: [
            'https://platform.dev.ameide.io/*',
          ],
          defaultClientScopes: ['openid', 'profile', 'email', 'roles'],
        },
      ],
    };
  }
}
```

**Update**: `services/platform/src/organizations/service.ts`

```typescript
import { KeycloakAdminService } from '../keycloak/admin';

export class OrganizationService {
  private keycloakAdmin: KeycloakAdminService;

  constructor(private db: Database) {
    this.keycloakAdmin = new KeycloakAdminService();
  }

  async createOrganization(
    request: platformOrganizations.CreateOrganizationRequest
  ): Promise<platformOrganizations.CreateOrganizationResponse> {
    const tenantId = requireTenantId(request.context);
    const orgId = uuidv4();
    const slug = slugify(request.name);

    // 1. Create organization in database
    const org = await this.db.organizations.create({
      data: {
        id: orgId,
        slug,
        name: request.name,
        realmName: slug,  // Realm name = slug
        tenantId,
      },
    });

    // 2. Create Keycloak realm
    try {
      await this.keycloakAdmin.createRealm({
        realm: slug,
        displayName: request.name,
        enabled: true,
      });

      console.log(`[OrganizationService] Created Keycloak realm for org: ${slug}`);
    } catch (error) {
      // Rollback database insert on Keycloak failure
      await this.db.organizations.delete({ where: { id: orgId } });
      throw new Error(`Failed to create Keycloak realm: ${error.message}`);
    }

    // 3. Create initial admin user in realm
    await this.provisionRealmAdmin(slug, request.adminEmail);

    return create(platformOrganizations.CreateOrganizationResponseSchema, {
      organization: mapOrganizationToProto(org),
    });
  }

  async deleteOrganization(
    request: platformOrganizations.DeleteOrganizationRequest
  ): Promise<platformOrganizations.DeleteOrganizationResponse> {
    const org = await this.db.organizations.findUnique({
      where: { id: request.organizationId },
    });

    if (!org) {
      throw new Error(`Organization ${request.organizationId} not found`);
    }

    // 1. Delete Keycloak realm
    await this.keycloakAdmin.deleteRealm(org.realmName);

    // 2. Delete organization from database
    await this.db.organizations.delete({
      where: { id: request.organizationId },
    });

    return create(platformOrganizations.DeleteOrganizationResponseSchema, {
      success: true,
    });
  }

  private async provisionRealmAdmin(realmName: string, adminEmail: string): Promise<void> {
    // TODO: Create admin user in realm via Keycloak Admin API
  }
}
```

**Environment Variables**:
```bash
# .env (add Keycloak admin credentials)
KEYCLOAK_BASE_URL=https://auth.dev.ameide.io
KEYCLOAK_ADMIN_USER=admin
KEYCLOAK_ADMIN_PASSWORD=admin
KEYCLOAK_CLIENT_SECRET=changeme
```

**Testing**:
```bash
# Test organization creation
curl -X POST http://localhost:50051/ameide_core_proto.platform.v1.OrganizationService/CreateOrganization \
  -H "Content-Type: application/json" \
  -d '{
    "context": {"tenantId": "platform"},
    "name": "Test Organization",
    "adminEmail": "admin@testorg.com"
  }'

# Verify realm created
ADMIN_TOKEN=$(curl -k -s -X POST https://auth.dev.ameide.io/realms/master/protocol/openid-connect/token \
  --data-urlencode "username=admin" \
  --data-urlencode "password=admin" \
  --data-urlencode "grant_type=password" \
  --data-urlencode "client_id=admin-cli" | jq -r '.access_token')

curl -k -H "Authorization: Bearer $ADMIN_TOKEN" \
  https://auth.dev.ameide.io/admin/realms/test-organization
```

---

### Phase 3: Testing & Validation (Week 2, Days 3-5)

#### Step 3.1: Add Integration Tests (Week 2, Day 3)

**File**: `services/www_ameide_platform/__tests__/integration/multi-realm-auth.test.ts` (NEW)

```typescript
import { extractRoles, extractTenantFromIssuer, validateTokenForOrganization } from '@/lib/auth/jwt';
import { getOrganizationByRealmName } from '@/lib/sdk/organizations';

describe('Multi-Realm Authentication Integration', () => {
  describe('Token Extraction', () => {
    it('should extract tenant from Atlas realm issuer', () => {
      const issuer = 'https://auth.dev.ameide.io/realms/atlas';
      const tenant = extractTenantFromIssuer(issuer);
      expect(tenant).toBe('atlas');
    });

    it('should extract simple roles from JWT', () => {
      const token = {
        iss: 'https://keycloak/realms/atlas',
        realm_access: { roles: ['admin', 'contributor'] },
      };

      const roles = extractRoles(token as any);
      expect(roles).toEqual(['admin', 'contributor']);
      expect(roles).not.toContain('org:atlas:admin');  // No prefixes
    });
  });

  describe('Organization Lookup', () => {
    it('should fetch organization by realm name', async () => {
      const org = await getOrganizationByRealmName('atlas');

      expect(org).toBeDefined();
      expect(org.realmName).toBe('atlas');
      expect(org.slug).toBe('atlas');
    });

    it('should throw when realm does not exist', async () => {
      await expect(getOrganizationByRealmName('nonexistent')).rejects.toThrow(
        'Organization with realm "nonexistent" not found'
      );
    });
  });

  describe('Cross-Realm Isolation', () => {
    it('should reject Atlas token for ACME organization', () => {
      const atlasToken = {
        iss: 'https://keycloak/realms/atlas',
        realm_access: { roles: ['admin'] },
      };

      const acmeOrg = { id: 'acme-id', realmName: 'acme' };

      expect(() => validateTokenForOrganization(atlasToken as any, acmeOrg)).toThrow(
        'Token issuer realm "atlas" does not match organization realm "acme"'
      );
    });

    it('should accept Atlas token for Atlas organization', () => {
      const atlasToken = {
        iss: 'https://keycloak/realms/atlas',
        realm_access: { roles: ['admin'] },
      };

      const atlasOrg = { id: 'atlas-id', realmName: 'atlas' };

      expect(() => validateTokenForOrganization(atlasToken as any, atlasOrg)).not.toThrow();
    });
  });
});
```

```bash
# Run integration tests
pnpm test --testPathPattern=multi-realm-auth.test.ts
```

#### Step 3.2: Add E2E Tests (Week 2, Day 4)

**File**: `services/www_ameide_platform/features/auth/__tests__/e2e/multi-realm.spec.ts` (NEW)

(Already provided in "Impact 7: Testing Strategy Changes" above - full test suite)

```bash
# Run E2E tests
BASE_URL=https://platform.dev.ameide.io pnpm exec playwright test multi-realm.spec.ts
```

#### Step 3.3: Update Documentation (Week 2, Day 5)

**Files to update**:

1. **README.md** - Add multi-realm authentication section
2. **docs/architecture/authentication.md** - Document realm resolution flow
3. **services/www_ameide_platform/README.md** - Update NextAuth configuration docs
4. **CHANGELOG.md** - Document breaking changes

**Example**: `docs/architecture/authentication.md`

```markdown
# Authentication Architecture

## Overview

The platform uses **realm-per-tenant** architecture where each organization has its own Keycloak realm.

## Key Concepts

### Realm = Organization = Tenant

- Each organization is mapped to a dedicated Keycloak realm
- Realm name equals organization slug (e.g., "atlas" organization → "atlas" realm)
- JWT issuer identifies the organization: `iss: "https://keycloak/realms/atlas"`

### Simplified JWT Structure

**Token from realm "atlas"**:
```json
{
  "iss": "https://keycloak/realms/atlas",
  "realm_access": {
    "roles": ["admin"]  // Simple roles, no prefixes
  }
}
```

### Authentication Flow

1. User navigates to `/org/atlas`
2. System resolves realm from path: `atlas`
3. User redirected to Keycloak realm: `https://keycloak/realms/atlas/protocol/openid-connect/auth`
4. User logs in (native Keycloak user OR federated from Atlas's Okta)
5. Keycloak issues JWT with `iss: "https://keycloak/realms/atlas"`
6. NextAuth callback extracts realm from issuer, looks up organization
7. Session includes organization context derived from realm

## Configuration

See [infra/kubernetes/charts/platform/keycloak-realm-template/](../../infra/kubernetes/charts/platform/keycloak-realm-template/) for realm configuration template.

## Migration from Single Realm

See [keycloak-realm-migration.md](./keycloak-realm-migration.md) for migration guide.
```

---

### Phase 4: Deprecation & Cleanup (Week 3-4)

#### Step 4.1: Mark Single Realm as Deprecated (Week 3, Day 1)

**File**: `infra/kubernetes/charts/platform/keycloak-realm/values.yaml` (add deprecation notice)

```yaml
# ⚠️ DEPRECATED: This single-realm configuration is deprecated in favor of realm-per-tenant.
# See: infra/kubernetes/charts/platform/keycloak-realm-template/
#
# This configuration will be maintained for backward compatibility until all organizations
# are migrated to dedicated realms.
#
# Migration timeline:
# - 2025-11-15: Stop creating new users in single realm
# - 2025-12-01: Migrate existing organizations to dedicated realms
# - 2025-12-15: Delete single "ameide" realm

# ... existing configuration
```

#### Step 4.2: Delete Deprecated Client Scopes (Week 3, Day 2)

**Task**: Remove `tenant`, `groups`, and `organizations` scopes from realm template

**File**: `infra/kubernetes/charts/platform/keycloak-realm-template/values.yaml`

```yaml
# BEFORE: 4 client scopes
"clientScopes": [
  {name: "roles", ...},
  {name: "tenant", ...},        # ❌ DELETE
  {name: "groups", ...},        # ❌ DELETE
  {name: "organizations", ...}  # ❌ DELETE
]

# AFTER: 1 client scope
"clientScopes": [
  {name: "roles", ...}
]
```

**Testing**:
```bash
# Deploy updated template
helm upgrade keycloak-realm-atlas \
  ./infra/kubernetes/charts/platform/keycloak-realm-template \
  -n ameide \
  -f ./infra/kubernetes/charts/platform/keycloak-realms/values-atlas.yaml

# Verify JWT does not contain deprecated claims
TOKEN=$(curl -k -s -X POST https://auth.dev.ameide.io/realms/atlas/protocol/openid-connect/token \
  --data-urlencode "username=admin" \
  --data-urlencode "password=admin" \
  --data-urlencode "grant_type=password" \
  --data-urlencode "client_id=app" \
  --data-urlencode "client_secret=changeme" | jq -r '.id_token')

echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | jq

# Expected: NO tenantId, groups, or org_groups claims
```

#### Step 4.3: Remove Deprecated Code (Week 3, Days 3-5)

**Files to delete**:

```bash
# Delete deprecated extraction functions
rm services/www_ameide_platform/lib/keycloak.ts

# Delete deprecated org resolution logic
rm services/www_ameide_platform/lib/auth/org-groups.ts

# Remove tenant attribute extraction
# (search for all references to `profile.tenantId` and remove)
```

**Files to update**:

```typescript
// services/www_ameide_platform/app/(auth)/auth.ts
// REMOVE:
// const tenantId = (profile.tenantId as string) ?? getFallbackOrgId();
// const orgGroups = extractOrgGroups(profile);

// REPLACE WITH:
const realmName = extractTenantFromIssuer(profile.iss);
const organization = await getOrganizationByRealmName(realmName);
```

**Run tests**:
```bash
pnpm test
pnpm -r --if-present typecheck
```

#### Step 4.4: Final Migration & Cutover (Week 4)

**Tasks**:
1. Migrate all existing users to dedicated realms
2. Configure external IdPs (Okta, Azure AD) for organizations that need SSO
3. Update DNS/routing for subdomain-based realm resolution (if using subdomains)
4. Delete single "ameide" realm
5. Update production ArgoCD manifests

---

## Risk Assessment & Mitigation

### Risk 1: Authentication Breaks During Migration
**Likelihood**: High
**Impact**: Critical (users cannot log in)

**Mitigation**:
- ✅ **Gradual rollout**: Keep single realm active during migration, cutover one org at a time
- ✅ **Feature flag**: Environment variable to toggle multi-realm vs single-realm mode
- ✅ **Fallback**: If multi-realm auth fails, redirect to single realm
- ✅ **Monitoring**: Alert on auth failure rate > 5%

**Rollback Plan**:
```bash
# Revert NextAuth to single realm provider
export AUTH_KEYCLOAK_ISSUER=https://auth.dev.ameide.io/realms/ameide
helm rollback keycloak-realm-atlas
```

### Risk 2: Token Validation Failures
**Likelihood**: Medium
**Impact**: High (API requests rejected)

**Mitigation**:
- ✅ **Comprehensive tests**: Integration tests covering all realm combinations
- ✅ **Issuer validation**: Strict checks on token issuer matching organization realm
- ✅ **Logging**: Detailed logs for token validation failures
- ✅ **Grace period**: Accept tokens from both single and multi-realm during migration

**Monitoring**:
```typescript
// Add metrics for token validation
metrics.increment('auth.token.validation.failed', {
  reason: 'issuer_mismatch',
  expected_realm: org.realmName,
  actual_realm: tokenRealm,
});
```

### Risk 3: Keycloak Admin API Failures
**Likelihood**: Medium
**Impact**: High (cannot create organizations)

**Mitigation**:
- ✅ **Retries**: Retry Keycloak Admin API calls with exponential backoff
- ✅ **Idempotency**: Check if realm exists before creating
- ✅ **Transactional rollback**: Delete database records if Keycloak creation fails
- ✅ **Manual recovery**: Document process for manually creating realms if automation fails

**Example**:
```typescript
async createRealm(config: RealmConfig): Promise<void> {
  try {
    await this.client.realms.create(config);
  } catch (error) {
    if (error.response?.status === 409) {
      // Realm already exists, continue
      console.log(`[KeycloakAdmin] Realm ${config.realm} already exists`);
      return;
    }
    throw error;
  }
}
```

### Risk 4: Performance Degradation
**Likelihood**: Low
**Impact**: Medium (slower auth)

**Mitigation**:
- ✅ **Caching**: Cache organization-realm mappings in Redis
- ✅ **Connection pooling**: Reuse Keycloak Admin API connections
- ✅ **Async operations**: Create realms asynchronously (don't block org creation)

**Performance Targets**:
- Realm resolution: < 10ms (cached)
- Token validation: < 50ms
- Organization creation (with realm): < 5s

### Risk 5: Data Loss During Migration
**Likelihood**: Low
**Impact**: Critical (user accounts lost)

**Mitigation**:
- ✅ **Backup**: Full Keycloak database backup before migration
- ✅ **Dry run**: Test migration on staging environment first
- ✅ **User export/import**: Export users from single realm, import to org-specific realms
- ✅ **Validation**: Verify user count matches before/after migration

**Backup Command**:
```bash
kubectl exec -n ameide keycloak-0 -- \
  /opt/keycloak/bin/kc.sh export --dir /tmp/keycloak-backup --realm ameide

kubectl cp ameide/keycloak-0:/tmp/keycloak-backup ./keycloak-backup-$(date +%Y%m%d)
```

---

## Success Metrics

### Configuration Metrics
| Metric | Target | How to Measure |
|--------|--------|----------------|
| **Client scopes reduced** | -75% (4→1) | Count scopes in values.yaml |
| **JWT claims reduced** | -80% (5→1) | Parse JWT, count custom claims |
| **YAML configuration lines** | -84% (250→40) | Line count in values.yaml |
| **Protocol mappers** | -83% (6→1) | Count mappers in clientScopes |

### Code Quality Metrics
| Metric | Target | How to Measure |
|--------|--------|----------------|
| **Extraction functions** | -33% (3→2) | Count functions in lib/auth/ |
| **JWT callback complexity** | -25% (4→3 steps) | Cyclomatic complexity |
| **User attributes** | -100% (1→0) | Count attributes in users |

### Test Coverage Metrics
| Test Type | Baseline | Target | How to Measure |
|-----------|----------|--------|----------------|
| **Integration tests** | 5 | 13 (+160%) | Count test cases |
| **E2E tests** | 3 | 9 (+200%) | Count test scenarios |
| **Infrastructure tests** | 0 | 4 (new) | Count Keycloak provisioning tests |

### Performance Metrics
| Metric | Baseline | Target | How to Measure |
|--------|----------|--------|----------------|
| **Token size** | ~850 bytes | ~420 bytes (-50%) | JWT byte length |
| **Auth latency** | N/A | < 200ms | Measure signin duration |
| **Realm creation** | N/A | < 5s | Measure org creation API |

### Operational Metrics
| Metric | Target | How to Measure |
|--------|--------|----------------|
| **Auth failure rate** | < 0.1% | (failed_auths / total_auths) * 100 |
| **Token validation failures** | < 1% | (rejected_tokens / total_tokens) * 100 |
| **Realm creation success rate** | > 99% | (successful_creates / total_creates) * 100 |

---

## Completion Checklist

### Phase 1: Configuration Template
- [ ] Create `keycloak-realm-template` Helm chart
- [ ] Define simplified client scopes (roles only)
- [ ] Test template deployment for single organization
- [ ] Document configuration differences

### Phase 2: Service Integration
- [ ] Implement multi-realm NextAuth provider
- [ ] Add realm resolution logic (path-based or subdomain-based)
- [ ] Simplify JWT extraction functions
- [ ] Add organization-realm database mapping
- [ ] Integrate Keycloak Admin API for realm creation
- [ ] Update organization creation flow

### Phase 3: Testing
- [ ] Add integration tests for multi-realm JWT validation
- [ ] Add E2E tests for cross-realm isolation
- [ ] Add infrastructure tests for realm provisioning
- [ ] Verify test coverage metrics meet targets

### Phase 4: Migration & Cleanup
- [ ] Mark single realm as deprecated
- [ ] Delete deprecated client scopes from template
- [ ] Remove deprecated code (extractOrgGroups, etc.)
- [ ] Migrate existing organizations to dedicated realms
- [ ] Delete single "ameide" realm
- [ ] Update production deployment manifests

### Phase 5: Monitoring & Validation
- [ ] Deploy to staging environment
- [ ] Verify authentication flows work end-to-end
- [ ] Monitor error rates and performance metrics
- [ ] Gradual rollout to production (one org at a time)
- [ ] Full production cutover
- [ ] Post-migration validation

---

## References

- [Keycloak Multi-Tenancy Documentation](https://www.keycloak.org/docs/latest/server_admin/#_multi-tenancy)
- [Keycloak Admin REST API](https://www.keycloak.org/docs-api/latest/rest-api/)
- [NextAuth.js Custom Providers](https://next-auth.js.org/configuration/providers/oauth#using-a-custom-provider)
- [backlog/333-realms.md](./333-realms.md) - Realm-per-tenant architecture master document
- [backlog/322-rbac.md](./322-rbac.md) - RBAC impact analysis
- [Microsoft Entra ID Multi-Tenant Architecture](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-convert-app-to-be-multi-tenant) - Industry reference
