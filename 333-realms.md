# ADR-333: Realm-Per-Tenant Architecture (Entra ID Model)

## Status
**STATUS** - Reference Design (implementation deferred)

## Date
- Proposed: 2025-10-30
- Implemented: 2025-10-30

## Current Snapshot (Nov 5 2025)

- ✅ Single Keycloak realm (`ameide`) hardened with chart-owned ExternalSecrets and CNPG-managed DB credentials.
- ⏳ Realm-per-tenant provisioning scripts remain in design; no feature-flag pilot is running.
- ⚠️ Guest federation, realm discovery, and realm teardown automation pending first implementation.
- ✅ Tenant catalog entries must now include `metadata.realmName`/`tenantSlug` at creation and update time so every downstream service (auth session, discovery, orchestration) can deterministically map a tenant to its Keycloak realm.

**Next Steps Before Rollout**
- Build the realm bootstrap CLI + Flyway migrations that persist organization↔realm mappings.
- Enable staging pilot behind feature flag once provisioning path is demonstrably idempotent.
- Extend runbook with teardown + guest federation flows after first working slice.

**Related Backlog Items**
- `backlog/319-onboarding-v2.md` – End-to-end onboarding flow built on this ADR
- `backlog/331-tenant-resolution.md` – How tenant IDs flow through auth/session layers
- `backlog/324-user-org-settings.md` – Post-onboarding org management UX

## Context

We are building a multi-tenant enterprise platform where each customer organization should have complete isolation and control over their authentication, similar to Microsoft Entra ID (formerly Azure AD).

### Business Requirements
- **Target customers**: Enterprise B2B customers
- **Tenant isolation**: Each customer is a completely separate tenant
- **SSO ownership**: Customers configure their own Identity Provider (Okta, Azure AD, Google Workspace, SAML)
- **Guest users**: Users from one tenant can be invited as guests to another tenant (B2B collaboration)
- **Admin delegation**: Customers should be able to manage users/groups within their tenant
- **Compliance**: Some enterprises require complete data and identity isolation

### Current Implementation (Single Realm)

**Architecture:**
```
Keycloak Realm: "ameide" (single shared realm)
├─ Users: admin@ameide.io, user1@acme.com, user2@competitor.com
├─ Groups: /orgs/atlas, /orgs/acme, /orgs/competitor
├─ tenantId attribute: "atlas-org" (on user)
└─ JWT claim: org_groups: ["/orgs/atlas"]
```

**Flow:**
1. User logs into single "ameide" realm
2. JWT token includes `org_groups` claim with group memberships
3. Backend filters data by organization keys from groups
4. All organizations share realm configuration (password policies, themes, IdP settings)

**Problems:**
- ❌ Cannot give customers control over their SSO configuration
- ❌ All orgs share password policies, session timeouts, branding
- ❌ Cannot delegate Keycloak admin access per organization
- ❌ Identity Providers configured in one realm (no isolation)
- ❌ Doesn't support enterprise requirement for complete tenant isolation
- ❌ Not aligned with standard B2B SaaS patterns (Entra ID, Okta, Auth0)

**What works:**
- ✅ Users can belong to multiple organizations
- ✅ Simple realm management
- ✅ Lower operational overhead
- ✅ Good for internal tools or B2C scenarios

## Decision

We will implement **Realm-Per-Tenant Architecture** where each organization gets its own Keycloak realm, aligned with Microsoft Entra ID's multi-tenant model.

### Target Architecture

```
Keycloak Instance
├─ Realm: "atlas" (Organization: Atlas Corp)
│   ├─ Identity Provider: Okta (configured by Atlas admin)
│   ├─ Users: alice@atlas.com, bob@atlas.com (native)
│   ├─ Guest Users: consultant@acme.com (federated from "acme" realm)
│   ├─ Roles: org-admin, org-member, org-guest
│   ├─ Clients: platform-app-atlas
│   └─ Settings: Atlas's password policy, branding, session config
│
├─ Realm: "acme" (Organization: ACME Corp)
│   ├─ Identity Provider: Azure AD (configured by ACME admin)
│   ├─ Users: user1@acme.com, user2@acme.com (native)
│   ├─ Guest Users: consultant@competitor.com (federated)
│   ├─ Roles: org-admin, org-member, org-guest
│   ├─ Clients: platform-app-acme
│   └─ Settings: ACME's password policy, branding, session config
│
└─ Realm: "competitor" (Organization: Competitor Inc)
    ├─ Identity Provider: Google Workspace
    ├─ Users: ...
    └─ Settings: Competitor's configuration
```

### Key Concepts

**Realm = Organization = Tenant**
- Each organization gets a dedicated Keycloak realm
- Realm name matches organization slug: `https://keycloak/realms/{org-slug}`
- Complete isolation: authentication, users, groups, configuration

**Native Users vs Guest Users**
- **Native users**: Belong to the realm (home tenant)
- **Guest users**: Federated from other realms (B2B collaboration)
- Implemented via Keycloak Identity Brokering between realms

**Realm Discovery**
- User enters email → domain lookup → redirect to correct realm
- Example: `alice@atlas.com` → realm "atlas"
- Example: `user@acme.com` → realm "acme"

**OIDC Client Per Realm**
- Each realm has its own OAuth client configuration
- JWT issuer identifies the realm: `iss: "https://keycloak/realms/atlas"`
- No need for `tenantId` claim - realm IS the tenant

## Implementation Plan

### Phase 1: Core Realm Provisioning (Week 1)

#### 1.1 Keycloak Admin Client - Realm Management
**File**: `services/www_ameide_platform/lib/keycloak-admin.ts`

Add functions:
```typescript
/**
 * Create a new realm for an organization
 * @param orgSlug - Organization slug (becomes realm name)
 * @param orgId - Organization UUID
 * @param adminUser - First admin user info
 * @returns Realm name
 */
export async function createOrganizationRealm(
  orgSlug: string,
  orgId: string,
  adminUser: {
    email: string;
    firstName: string;
    lastName: string;
    keycloakUserId: string; // From registration in master/ameide realm
  }
): Promise<string>

/**
 * Configure OIDC client in realm for platform app
 */
async function configureRealmClient(realmName: string): Promise<void>

/**
 * Create initial admin user in new realm
 */
async function provisionRealmAdmin(realmName: string, user: AdminUser): Promise<void>

/**
 * Delete realm (for cleanup/testing)
 */
export async function deleteOrganizationRealm(realmName: string): Promise<void>
```

**Realm Configuration Template:**
```typescript
const realmTemplate = {
  realm: orgSlug,
  enabled: true,
  displayName: organizationName,

  // Authentication settings
  registrationAllowed: false, // Org admins manage users
  resetPasswordAllowed: true,
  rememberMe: true,
  loginWithEmailAllowed: true,
  duplicateEmailsAllowed: false,

  // Session settings (can be customized per org later)
  ssoSessionIdleTimeout: 1800,
  ssoSessionMaxLifespan: 36000,

  // Tokens
  accessTokenLifespan: 300,

  // Attributes for linking to platform
  attributes: {
    organizationId: orgId,
    organizationSlug: orgSlug,
  },

  // Roles
  roles: {
    realm: [
      { name: "org-admin", description: "Organization administrator" },
      { name: "org-member", description: "Organization member" },
      { name: "org-guest", description: "Guest user from another organization" },
    ],
  },

  // Client for platform app
  clients: [{
    clientId: `platform-app-${orgSlug}`,
    name: `Platform Application (${organizationName})`,
    enabled: true,
    publicClient: false,
    standardFlowEnabled: true,
    directAccessGrantsEnabled: true,
    redirectUris: [
      `https://platform.dev.ameide.io/*`,
      `https://${orgSlug}.ameide.io/*`,
      `https://platform.ameide.io/*`,
    ],
    webOrigins: ["+"],
    protocol: "openid-connect",
    attributes: {
      "backchannel.logout.url": `https://platform.dev.ameide.io/api/auth/backchannel-logout`,
      "backchannel.logout.session.required": "true",
    },
  }],
};
```

#### 1.2 Onboarding Flow Update
**File**: `services/www_ameide_platform/features/identity/lib/orchestrator.ts`

Replace group creation with realm creation:

```typescript
// After organization created in database
const orgId = orgResponse.organization?.metadata?.id || '';

console.log('[orchestrator:completeRegistration] Creating Keycloak realm', {
  requestId,
  orgSlug: slug,
  orgId,
});

try {
  const { createOrganizationRealm } = await import('@/lib/keycloak-admin');

  const realmName = await createOrganizationRealm(slug, orgId, {
    email: request.email,
    firstName: request.firstName || '',
    lastName: request.lastName || '',
    keycloakUserId: request.keycloakUserId,
  });

  console.log('[orchestrator:completeRegistration] Realm created', {
    requestId,
    realmName,
    issuer: `${KEYCLOAK_BASE_URL}/realms/${realmName}`,
  });
} catch (error) {
  console.error('[orchestrator:completeRegistration] Failed to create realm', {
    requestId,
    error: error instanceof Error ? error.message : String(error),
  });
  throw error; // Realm creation is critical - fail the onboarding
}
```

#### 1.3 Database Schema Update
**File**: `db/flyway/sql/V37__organization_realm_mapping.sql`

```sql
-- Add realm_name column to organizations table
ALTER TABLE platform.organizations
  ADD COLUMN realm_name VARCHAR(255);

-- Create unique index on realm_name
CREATE UNIQUE INDEX idx_organizations_realm_name
  ON platform.organizations(realm_name)
  WHERE realm_name IS NOT NULL;

-- Add comments
COMMENT ON COLUMN platform.organizations.realm_name IS
  'Keycloak realm name for this organization (realm-per-tenant architecture)';

-- Backfill existing organizations (if any)
-- UPDATE platform.organizations SET realm_name = key WHERE realm_name IS NULL;
```

### Phase 2: Realm Discovery & Switching (Week 2)

#### 2.1 Realm Discovery Mechanism
**File**: `services/www_ameide_platform/lib/auth/realm-discovery.ts`

```typescript
/**
 * Determine which realm a user should authenticate against
 * based on their email domain
 */
export async function discoverRealmFromEmail(email: string): Promise<string | null> {
  const domain = email.split('@')[1];

  // Query database for organization with verified domain
  const org = await findOrganizationByDomain(domain);
  if (org?.realm_name) {
    return org.realm_name;
  }

  // Default: return null for email entry screen
  return null;
}

/**
 * Get organization's realm name from slug
 */
export async function getRealmForOrganization(orgSlug: string): Promise<string> {
  const org = await findOrganizationBySlug(orgSlug);
  if (!org?.realm_name) {
    throw new Error(`No realm found for organization: ${orgSlug}`);
  }
  return org.realm_name;
}

/**
 * Build Keycloak authorization URL for specific realm
 */
export function buildRealmAuthUrl(realmName: string, redirectUri: string): string {
  const baseUrl = process.env.AUTH_KEYCLOAK_BASE_URL;
  return `${baseUrl}/realms/${realmName}/protocol/openid-connect/auth?...`;
}
```

#### 2.2 Authentication Flow Update
**File**: `services/www_ameide_platform/app/(auth)/auth.ts`

**Current (single realm):**
```typescript
providers: [
  {
    id: "keycloak",
    name: "Keycloak",
    type: "oidc",
    issuer: process.env.AUTH_KEYCLOAK_ISSUER, // Single realm
  }
]
```

**Updated (multi-realm):**
```typescript
// Dynamic provider based on organization
// Requires custom authorize flow with realm discovery

async authorize(credentials) {
  // 1. User enters email on login page
  const email = credentials.email;

  // 2. Discover realm from email domain
  const realmName = await discoverRealmFromEmail(email);

  // 3. Redirect to realm-specific authorization URL
  const authUrl = buildRealmAuthUrl(realmName, callbackUrl);

  // 4. Continue OAuth flow with discovered realm
  return { redirect: authUrl };
}
```

**Alternative: Org-based login URLs**
```
https://platform.ameide.io/auth/signin?org=atlas
https://platform.ameide.io/auth/signin?org=acme
```

#### 2.3 Middleware Update
**File**: `services/www_ameide_platform/middleware.ts`

Extract realm from JWT issuer:
```typescript
const session = await getSession();
if (session?.user) {
  // JWT issuer identifies the realm/org
  // iss: "https://keycloak/realms/atlas"
  const issuer = session.accessToken?.iss;
  const realmName = issuer?.split('/realms/')[1];

  // Set organization context from realm
  request.headers.set('x-org-realm', realmName);
}
```

### Phase 3: Guest Users & B2B Federation (Week 3)

#### 3.1 Realm-to-Realm Identity Brokering
Configure each realm to federate with other organization realms:

```typescript
async function setupRealmFederation(sourceRealm: string, targetRealm: string) {
  // In sourceRealm, add targetRealm as Identity Provider
  await createIdentityProvider(sourceRealm, {
    alias: `org-${targetRealm}`,
    providerId: 'keycloak-oidc',
    enabled: true,
    config: {
      clientId: `platform-app-${sourceRealm}`,
      tokenUrl: `${KEYCLOAK_URL}/realms/${targetRealm}/protocol/openid-connect/token`,
      authorizationUrl: `${KEYCLOAK_URL}/realms/${targetRealm}/protocol/openid-connect/auth`,
      defaultScope: 'openid email profile',
    },
    // Map to "org-guest" role
    mappers: [
      {
        name: 'guest-role-mapper',
        identityProviderMapper: 'hardcoded-role-idp-mapper',
        config: { role: 'org-guest' },
      },
    ],
  });
}
```

#### 3.2 Guest Invitation Flow
**File**: `services/www_ameide_platform/app/api/v1/invitations/route.ts`

```typescript
async function inviteGuestUser(
  hostOrgRealm: string,
  guestEmail: string,
  guestOrgRealm: string
) {
  // 1. Verify guest user exists in their home realm
  const guestUser = await getKeycloakUser(guestOrgRealm, guestEmail);

  // 2. Create federated link in host realm
  await createFederatedIdentity(hostOrgRealm, {
    identityProvider: `org-${guestOrgRealm}`,
    userId: guestUser.id,
    userName: guestEmail,
  });

  // 3. Assign guest role in host realm
  await assignRoleToUser(hostOrgRealm, guestUser.id, 'org-guest');
}
```

### Phase 4: Migration & Cleanup (Week 4)

#### 4.1 Remove Group-Based Code
Files to update:
- ❌ Remove `org_groups` JWT claim extraction
- ❌ Remove group-based filtering in `services/platform/src/organizations/service.ts`
- ❌ Remove `orgKeys` parameter from `ListOrganizationsRequest`
- ❌ Remove Keycloak group creation functions

#### 4.2 Data Migration
```sql
-- Migrate existing organizations to realms
-- (Manual process with careful validation)

-- 1. Create realm for each organization
-- 2. Migrate users to their organization's realm
-- 3. Update organization.realm_name
-- 4. Verify all users can authenticate
```

#### 4.3 Update Documentation
- Architecture diagrams
- Authentication flow
- Onboarding process
- Realm management runbook

## Technical Considerations

### JWT Token Validation
**Single realm approach:**
```typescript
// Validate against single issuer
const issuer = "https://keycloak/realms/ameide";
```

**Multi-realm approach:**
```typescript
// Validate against any organization realm
async function validateToken(token: string) {
  const decoded = jwt.decode(token);
  const issuer = decoded.iss; // "https://keycloak/realms/atlas"

  // Verify issuer is a valid organization realm
  const realmName = issuer.split('/realms/')[1];
  const org = await findOrganizationByRealm(realmName);

  if (!org) {
    throw new Error('Invalid realm');
  }

  // Fetch realm's public keys for signature verification
  const jwks = await fetchRealmJWKS(realmName);
  return jwt.verify(token, jwks);
}
```

### Session Management
- Users may need to access multiple organizations (switch realms)
- Implement org switcher that triggers re-authentication in different realm
- Store active realm in session/cookie

### Realm Isolation
- Each realm has separate user database
- Guest users appear in multiple realms (federated)
- Data access controlled by organization ID in database, verified against JWT issuer realm

### Performance & Scalability
- **Realm limit**: Keycloak supports thousands of realms
- **Memory**: Each realm has overhead (~10-50MB depending on config)
- **Startup time**: More realms = longer Keycloak startup
- **Solution**: Use Keycloak clustering for production

### Backup & Disaster Recovery
- Export all realms regularly
- Automate realm backup on creation
- Test realm restore procedures

## Open Questions

### Q1: Registration Realm
**Question:** Where do users initially register before an organization exists?

**Options:**
1. **Master realm**: Register in Keycloak master realm, then migrate to org realm on first org join
2. **Default realm**: Use "ameide" realm for pre-org users, migrate on org creation
3. **Deferred creation**: User enters email → organization created → user created in org realm

**Recommendation:** Option 3 - Create realm during onboarding, create user directly in org realm

### Q2: Realm Discovery UX
**Question:** How does a user know which organization to log into?

**Options:**
1. **Email domain lookup**: Enter email → auto-discover realm
2. **Organization selector**: Show list of orgs user belongs to
3. **Direct URL**: Each org has unique subdomain: `atlas.ameide.io`, `acme.ameide.io`

**Recommendation:** Combination of 1 & 3 - domain lookup + org subdomains

### Q3: Cross-Realm User Management
**Question:** How do we manage users who exist in multiple realms as guests?

**Options:**
1. **Separate identities**: User has different ID in each realm (federated)
2. **Global user index**: Maintain mapping of user across realms in platform database
3. **Master user realm**: All users in master, federated to org realms

**Recommendation:** Option 2 - Platform database tracks user's home realm + guest realms

### Q4: Realm Admin Delegation
**Question:** Can customers admin their own Keycloak realm?

**Options:**
1. **Full access**: Give org admins Keycloak realm admin role
2. **Limited access**: Only allow user/group management, not IdP configuration
3. **No access**: All realm management via platform UI (calls admin API)

**Recommendation:** Start with Option 3, consider Option 2 for enterprise tier

### Q5: Realm Naming
**Question:** What if organization slug changes?

**Options:**
1. **Immutable realm name**: Realm name never changes (use UUID)
2. **Mutable realm name**: Support realm renaming (complex)
3. **Redirect alias**: Keep old realm as alias, redirect to new

**Recommendation:** Option 1 - Use immutable realm name (org UUID or initial slug)

### Q6: Client Secrets Management
**Question:** Where to store per-realm client secrets?

**Options:**
1. **Database**: Store in platform.organizations table (encrypted)
2. **Key Vault**: Store in Azure Key Vault per environment
3. **Keycloak**: Let Keycloak manage, fetch via admin API

**Recommendation:** Option 2 - Azure Key Vault with naming convention: `realm-{org-slug}-client-secret`

## Consequences

### Positive
✅ **Enterprise-ready**: Meets customer expectations for tenant isolation
✅ **SSO ownership**: Customers configure their own IdP (Okta, Azure AD, etc.)
✅ **Compliance**: Strong security boundaries for regulated industries
✅ **Branding**: Per-org login pages, themes, email templates
✅ **Admin delegation**: Can give customers realm admin access
✅ **Scalability**: Clear resource boundaries per tenant
✅ **Industry standard**: Aligns with Entra ID, Okta, Auth0 patterns

### Negative
❌ **Complex onboarding**: Must create realm during org creation
❌ **Guest users**: Requires realm-to-realm federation for B2B
❌ **Operational overhead**: More realms to monitor and maintain
❌ **Startup time**: Keycloak takes longer to start with many realms
❌ **Migration effort**: Need to migrate from current single-realm setup
❌ **Testing complexity**: Each test may need a new realm

### Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Realm creation fails during onboarding | User can't complete signup | Implement retry logic, transaction rollback, clear error messages |
| Realm proliferation | High memory usage, slow startup | Monitor realm count, implement archival for inactive orgs |
| Guest user federation breaks | Cross-org collaboration fails | Automate federation setup, add health checks |
| Realm discovery ambiguity | User can't find their org | Implement org lookup API, email domain verification |
| Client secret management | Secrets leak or lost | Use Key Vault, rotation policies, audit logging |
| Migration complexity | Data loss or downtime | Phased rollout, comprehensive testing, rollback plan |

## Implementation Checklist

### Week 1: Core Realm Provisioning ✅ COMPLETE
- [x] Implement `createOrganizationRealm()` in keycloak-admin.ts
- [x] Add realm template with client configuration
- [x] Update orchestrator to create realm instead of group
- [x] Add realm_name column to organizations table
- [x] Verify realm-specific JWT tokens
- [x] Add retry/backoff handling for Keycloak admin operations
- [ ] Test realm creation end-to-end in local environment (pending local testing)

**Files:**
- `services/www_ameide_platform/lib/keycloak-admin.ts` - Realm creation functions
- `services/www_ameide_platform/features/identity/lib/orchestrator.ts` - Onboarding flow
- `services/platform/src/organizations/service.ts` - Platform service updates
- `db/flyway/sql/V37__organization_realm_mapping.sql` - Database migration
- `packages/ameide_core_proto/src/ameide_core_proto/platform/v1/organizations.proto` - Proto updates

### Week 2: Realm Discovery ✅ COMPLETE
- [x] Implement email domain → realm lookup
- [x] Add organization-specific login URLs
- [x] Update auth.ts for multi-realm validation
- [x] Update middleware to extract realm from JWT issuer
- [x] Update backend services to validate realm
- [ ] Test login flow for multiple organizations (pending local testing)

**Files:**
- `services/www_ameide_platform/lib/auth/realm-discovery.ts` - Realm discovery utilities
- `services/www_ameide_platform/lib/auth/realm-manager.ts` - Realm management
- `services/www_ameide_platform/app/(auth)/auth.ts` - Multi-realm token handling
- `services/www_ameide_platform/types/auth.d.ts` - Type definitions
- `services/www_ameide_platform/lib/sdk/organizations.ts` - SDK helpers

### Week 3: Guest Users ✅ COMPLETE
- [x] Implement realm-to-realm identity brokering
- [x] Add guest invitation API
- [x] Create guest role mapping
- [ ] Test cross-realm user access (pending local testing)
- [ ] Document B2B collaboration flow (pending)

**Files:**
- `services/www_ameide_platform/lib/auth/realm-federation.ts` - Federation & guest access

### Week 4: Migration & Cleanup ⏳ PENDING
- [ ] Remove group-based authentication code
- [ ] Remove org_groups JWT claim usage
- [ ] Update all auth documentation
- [ ] Create realm management runbook
- [ ] Plan production migration strategy
- [ ] Load test Keycloak with 100+ realms

### Operational Updates (2025-11)
- Identity orchestrator now provisions dedicated realms unconditionally during onboarding.
- Keycloak admin calls still use exponential backoff with jitter; retry logs include `create-realm`, `create-realm-client`, and `create-realm-admin-user` labels.
- Onboarding verification: create org via UI, confirm realm exists in Keycloak, confirm `organizations.realm_name` populated.

## Cost & Resource Implications

**Development Time:**
- Estimated: 3-4 weeks for full implementation
- Complexity: High (authentication, multi-tenancy, federation)

**Infrastructure:**
- Keycloak memory: ~20-50MB per realm
- 100 organizations ≈ 2-5GB additional RAM
- Recommendation: Scale Keycloak horizontally

**Operational:**
- Monitoring: Per-realm metrics needed
- Backup: Automated realm export/import
- Support: Realm-specific debugging tools

## References

- [Keycloak Realm Management](https://www.keycloak.org/docs/latest/server_admin/#_realms)
- [Microsoft Entra ID Multi-Tenant Architecture](https://learn.microsoft.com/en-us/azure/active-directory/develop/multi-tenant-applications)
- [Keycloak Identity Brokering](https://www.keycloak.org/docs/latest/server_admin/#_identity_broker)
- [B2B Federation Patterns](https://www.keycloak.org/docs/latest/server_admin/#_identity_broker_first_login)
- Backlog #331 (Tenant Resolution)
- Backlog #319 (Onboarding)
- Backlog #322 (RBAC)

## Related Work

**Dependencies:**
- #331 - Tenant resolution (provides foundation)
- #319 - Onboarding flow (integration point)
- #322 - RBAC (role mapping in realms)

**Supersedes:**
- Current group-based organization membership
- Single "ameide" realm approach
- org_groups JWT claim

**Blocked by:**
- None (can start immediately)

---

## Appendix A: Comparison Matrix

| Feature | Single Realm + Groups | Realm Per Tenant |
|---------|----------------------|------------------|
| SSO per org | ❌ Shared IdP config | ✅ Dedicated IdP per org |
| Tenant isolation | ⚠️ Application-level | ✅ Keycloak-level |
| Admin delegation | ❌ Can't give realm access | ✅ Can delegate realm admin |
| Multi-org users | ✅ Native support | ⚠️ Via federation |
| Branding | ❌ Shared theme | ✅ Per-org themes |
| Compliance | ⚠️ Shared identity DB | ✅ Strong boundaries |
| Operational overhead | ✅ Low (one realm) | ⚠️ High (many realms) |
| Startup time | ✅ Fast | ⚠️ Slower with many realms |
| Resource usage | ✅ Low | ⚠️ Higher (~50MB/realm) |
| Migration effort | ✅ Already implemented | ❌ Requires full rewrite |

## Appendix B: Example User Journey

### Scenario: ACME Corp admin sets up SSO

1. **Organization Creation**
   - Admin completes onboarding form
   - Platform creates:
     - Organization record in database
     - Keycloak realm: "acme"
     - Admin user in realm
     - OAuth client for platform app

2. **SSO Configuration**
   - Admin navigates to Settings > Authentication
   - Clicks "Configure Azure AD"
   - Enters Azure AD tenant ID, client ID, secret
   - Platform calls Keycloak Admin API to create Identity Provider in "acme" realm

3. **User Login**
   - Employee visits `https://acme.ameide.io` or `https://platform.ameide.io`
   - Enters email: `user@acme.com`
   - Platform discovers realm "acme" from domain
   - Redirects to Keycloak realm "acme"
   - Keycloak detects Azure AD IdP for domain
   - Redirects to Azure AD
   - User authenticates with corporate credentials
   - Returns to platform with JWT from realm "acme"

4. **Guest Invitation**
   - ACME admin invites consultant from Atlas Corp: `consultant@atlas.com`
   - Platform creates federated identity link from "acme" realm to "atlas" realm
   - Consultant receives invitation email
   - Clicks link, authenticates in home realm "atlas"
   - Gets access to ACME org with "org-guest" role
   - Can switch between Atlas (home) and ACME (guest)
