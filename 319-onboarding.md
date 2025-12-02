> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Onboarding & Invitation System - Implementation Status

**Last Updated**: 2026-01-07  
**Overall Status**: ⚠️ Frontend ready, backend dependencies blocking end-to-end  
**Summary**: Onboarding gate, UI, and orchestration remain in place. A new bootstrap pathway (`POST /api/v1/registrations/bootstrap`) now provisions pending tenants for self-serve signups, but realm provisioning and platform service availability are still required before the flow can complete successfully.

---

## 🔍 Architecture Alignment (2026-01-07)

**Status**: Core APIs continue to enforce tenant-scoped access with fail-fast semantics. Temporary fallbacks (auto-provisioned tenants, realm warnings, in-memory orgs) have been removed. The flow now blocks until infrastructure is healthy.

**Verification Snapshot**:
- ✅ `features/identity/lib/orchestrator.ts` – Requires `tenantId` and propagates Keycloak/Platform errors
- ✅ `app/api/v1/registrations/complete/route.ts` – Validates session tenant, re-checks Keycloak attribute, and returns 409 when bootstrap is incomplete
- ✅ `/api/v1/organizations/create` – Still relies on session tenantId (no defaults)
- ✅ Invitation routes – Continue to enforce tenant scoping on write operations

**Security Posture**:
- ✅ No environment fallbacks; Keycloak attribute is the single source
- ✅ Missing tenantId → 409 Conflict (intentional block)
- ✅ Tenant isolation enforced at API and RLS layers
- ⚠️ Keycloak admin client needs `realm-admin` + `create-realm`; provisioning currently fails with 403 until ops restores roles

---

## ❗ Current Blockers

1. **Keycloak realm provisioning fails (403)** – `platform-app` client still lacks `realm-admin` + `create-realm` in the master realm. Orchestrator now aborts instead of logging a warning.
2. **Platform service disabled** – `helmfile` leaves the platform release `installed: false`. As a result, `organizationService.createOrganization` throws `ConnectError` and onboarding stops.
3. **Tenant catalog automation pending** – No job creates the `pending_onboarding` row. Without it, `ensureUserTenant` cannot resolve tenants and `/registrations/complete` returns 409.
4. **Email delivery** – Invitation emails still require manual send (unchanged).

---

## 🏗️ Target Architecture: Tenant vs Organization

### Architectural Model

Our platform uses a **two-level hierarchy** for multi-tenancy:

```
DEPLOYMENT (Kubernetes Cluster)
  └── TENANT (Infrastructure Isolation)
       └── ORGANIZATION (User Workspace)
            └── USERS (Team Members)
```

### Level 1: Tenant (Infrastructure)

**Definition**: Platform deployment container providing database and infrastructure isolation

**Examples**: `atlas-org`, `enterprise-customer`, `ameide-eu`

**Characteristics**:
- ✅ Provisioned by platform administrators in Keycloak (user attribute)
- ✅ Flows from JWT token (`session.user.tenantId`)
- ✅ Static per cluster/region
- ✅ Used for compliance, data residency, operational isolation
- ❌ **NEVER** created by end users
- ❌ **NEVER** visible in UI
- ❌ **NEVER** from environment variables or hardcoded defaults

**Provisioning Process**:
1. Admin creates/selects tenant (e.g., `atlas-org`)
2. Admin creates user in Keycloak
3. Admin sets user attribute: `tenantId = "atlas-org"`
4. User logs in → JWT includes `tenantId` claim
5. All operations scoped to this tenant automatically

### Level 2: Organization (User Workspace)

**Definition**: Customer/team workspace within a tenant

**Examples**: `acme`, `competitor`, `startup` (all within `atlas-org` tenant)

**Characteristics**:
- ✅ Created by end users during onboarding
- ✅ Visible in UI (org name, slug)
- ✅ Multiple organizations per tenant
- ✅ Users can belong to multiple organizations
- ✅ URL structure: `/org/{slug}`
- ✅ Created with `tenantId` from JWT (not environment variable)

**User-Facing Flow**:
1. User logs in (has `tenantId` in JWT from Keycloak)
2. User goes through onboarding wizard
3. User creates **organization** (NOT tenant)
4. Organization is scoped to user's JWT `tenantId` automatically

### What Users Create vs What Admins Create

| Entity | Created By | When | How | Visible to Users |
|--------|-----------|------|-----|------------------|
| **Tenant** | Platform Admin | Before user signup | Keycloak user attribute | ❌ No (infrastructure) |
| **Organization** | End User | During onboarding | Onboarding wizard | ✅ Yes (workspace) |
| **Membership** | Org Owner/Admin | After org creation | Invitations | ✅ Yes (team) |

### Correct Terminology

**In UI/Documentation**:
- ✅ "Create your organization"
- ✅ "Organization name"
- ✅ "Switch organizations"
- ❌ "Create tenant" ← Never use
- ❌ "Tenant selection" ← Never use
- ❌ "Tenant ID" ← Never expose to users

**In Code**:
- ✅ `session.user.tenantId` (from JWT)
- ✅ `organization.name` (user input)
- ❌ `process.env.DEFAULT_TENANT_ID` ← Remove
- ❌ `'atlas-org'` as fallback ← Remove

---

### Progress Snapshot

```
Frontend wizard & middleware:     ████████████████████ 100% ✅
Registration API + orchestrator:  ████████████████████ 100% ✅
Keycloak realm-per-tenant:        ████████░░░░░░░░░░░░  40% ⚠️ (blocked by admin roles)
Platform service availability:    ██░░░░░░░░░░░░░░░░░  10% ⚠️ (helm release disabled)
Tenant catalog automation:        ███░░░░░░░░░░░░░░░░  15% ⚠️ (manual entries)
Email / SSO / Billing:            ░░░░░░░░░░░░░░░░░░░░   0% ⚠️ (Phase 2)
```

### Test Coverage

| Test Type | Files | Lines | Coverage |
|-----------|-------|-------|----------|
| E2E Tests - Onboarding | 1 | 269 | ✅ Redirect & happy path (requires healthy backend) |
| E2E Tests - Invitations | 1 | 254 | ✅ End-to-end acceptance |
| Integration Tests | Multiple | ~500 | ✅ API contracts |
| Unit Tests | Multiple | ~300 | ✅ UI + orchestrator |

---

## Quick Reference: Implementation Files

| Feature | Status | Implementation Files |
|---------|--------|---------------------|
| **Database Schema** | ✅ | `db/flyway/sql/V34__platform_invitations.sql` |
| **Protobuf Contracts** | ✅ | `packages/ameide_core_proto/src/ameide_core_proto/platform/v1/invitations.proto` |
| **Backend Service** | ✅ | `services/platform/src/invitations/service.ts` (620 lines, 7 methods) |
| **Frontend SDK** | ✅ | `services/www_ameide_platform/features/invitations/lib/service.ts` |
| **Identity Orchestrator** | ✅ **VERIFIED** | `services/www_ameide_platform/features/identity/lib/orchestrator.ts` |
| **Onboarding Wizard** | ✅ | `services/www_ameide_platform/app/(app)/onboarding/page.tsx` |
| **Invitation Acceptance** | ✅ | `services/www_ameide_platform/app/(app)/accept/page.tsx` |
| **API: Create Invitation** | ✅ **VERIFIED** | `app/api/v1/invitations/route.ts` (POST) |
| **API: Validate Invitation** | ✅ | `app/api/v1/invitations/validate/route.ts` (GET) |
| **API: Accept Invitation** | ✅ **VERIFIED** | `app/api/v1/invitations/accept/route.ts` (POST) |
| **API: Get/Revoke** | ✅ **VERIFIED** | `app/api/v1/invitations/[id]/route.ts` (GET/DELETE) |
| **API: Resend** | ✅ **VERIFIED** | `app/api/v1/invitations/[id]/resend/route.ts` (POST) |
| **API: List** | ✅ **VERIFIED** | `app/api/v1/organizations/[orgId]/invitations/route.ts` (GET) |
| **API: Registration** | ✅ **VERIFIED** | `app/api/v1/registrations/complete/route.ts` (POST) |
| **Middleware Onboarding Gate** | ✅ | `middleware.ts` (redirect logic for new users) |
| **E2E Tests: Onboarding** | ✅ | `features/onboarding/__tests__/e2e/onboarding-flow.spec.ts` (269 lines) |
| **E2E Tests: New User Redirect** | ✅ | `features/onboarding/__tests__/e2e/new-user-redirect.spec.ts` (NEW) |
| **E2E Tests: Invitations** | ✅ | `features/invitations/__tests__/e2e/invitation-flow.spec.ts` (254 lines) |
| **Email Integration** | ⚠️ | NOT IMPLEMENTED - URLs returned, manual send |
| **Event Bus** | ⚠️ | NOT IMPLEMENTED - Direct gRPC works |
| **Domain Discovery** | ⚠️ | NOT IMPLEMENTED - Phase 2 |
| **Corporate SSO** | ⚠️ | NOT IMPLEMENTED - Phase 2 |

---

## Implementation Snapshot

### Backend & Services
- ✅ Invitation schema and gRPC service remain production-ready
- ✅ Registration API & orchestrator create orgs/memberships when dependencies are online
- ✅ `/api/v1/registrations/bootstrap` provisions pending tenants and stamps Keycloak attributes for self-serve signups
- ⚠️ Keycloak realm provisioning fails (missing master realm roles)
- ⚠️ Platform service disabled by Helm (`installed: false`), causing gRPC `ConnectError`

### Frontend & Middleware
- ✅ Onboarding modal collects organization plus optional tenant slug
- ✅ Session update triggers `refreshOrganizations` and `refreshTenant`
- ✅ Middleware gate enforces `hasRealOrganization`
- ⚠️ Without backend readiness the user receives surfaced error (expected fail-fast)

### Operational Follow-ups
- ⚠️ Restore Keycloak `platform-app` client permissions (`realm-admin`, `create-realm`)
- ⚠️ Deploy platform service and dependencies (Tilt/Helm)
- ⚠️ Monitor bootstrap endpoint (rate limiting, analytics) and close the loop from `pending_onboarding` → `active`
- ⚠️ Implement email delivery, event bus, domain verification, billing, SSO (Phase 2)

---

## Original Design Specification

Below is a pragmatic architecture and end‑to‑end onboarding flow for a modern, multi‑tenant SaaS built on your stack:

* **Kubernetes cluster**
* **Public marketing site:** `ameide.io`
* **Next.js app:** `platform.ameide.io`
* **Keycloak auth:** `auth.ameide.io`
* **Microservices** that need new users/tenants to be "seeded"

---

## 1) High‑level architecture

```
[User Browser]
   │
   ├─> ameide.io  (public site; captures email -> hands off to platform)  ⚠️ Marketing site exists, handoff not automated
   │
   └─> platform.ameide.io (Next.js)  ✅ IMPLEMENTED
         ├─ OIDC (PKCE) -> auth.ameide.io (Keycloak)  ✅ IMPLEMENTED
         │        ├─ Local auth (password, WebAuthn, OTP)  ✅ IMPLEMENTED (password + email)
         │        └─ Enterprise IdP via Keycloak Identity Brokering (SAML/OIDC)  ⚠️ NOT IMPLEMENTED
         │
         ├─ /api (BFF / Gateway)  ✅ IMPLEMENTED
         │     ├─ Identity Orchestrator (user, tenant, membership)  ✅ IMPLEMENTED
         │     ├─ Invitations API  ✅ IMPLEMENTED (7 endpoints)
         │     ├─ Domain Discovery API  ⚠️ NOT IMPLEMENTED
         │     └─ Billing/Plans API (optional)  ⚠️ NOT IMPLEMENTED
         │
         └─ Event Producer  ———┐  ⚠️ NOT IMPLEMENTED
                               │  (Kafka/NATS/SNS+SQS)
                         [Event Bus: tenant.created, user.created, membership.created]  ⚠️ NOT IMPLEMENTED
                               │
        ┌──────────────────────┴──────────────────────┐
        │              │                │             │
   Service A      Service B        Search/BI     Audit/Logs  ⚠️ NOT IMPLEMENTED
 (provisioner)  (notifications)    projections     (immutable)
```

**Key ideas**

* **Keycloak** remains the single source of identity and auth; **your app** is source of truth for **tenants and memberships**.  ✅ **IMPLEMENTED**
* **Event‑driven seeding**: after user/tenant creation, publish canonical events so microservices can provision themselves idempotently.  ⚠️ **NOT IMPLEMENTED** - Direct gRPC calls work but no event bus
* **Multi‑tenant enforcement** in every service (e.g., Postgres row‑level security + `tenant_id` in JWT claims).  ✅ **IMPLEMENTED** - RLS on invitations, tenant context in all operations

---

## 2) Tenancy model & URL strategy (CORRECTED)

**Architecture**: Two-level hierarchy (see section above)

### Tenant Level (Infrastructure)
* **Tenant**: Infrastructure isolation container (e.g., `atlas-org`, `enterprise-customer`)
  - ✅ **Provisioned**: Platform admin sets Keycloak user attribute `tenantId`
  - ✅ **JWT Flow**: `session.user.tenantId` from Keycloak token
  - ✅ **Implementation**: All API routes extract JWT tenantId with fail-secure validation
  - ✅ **No fallbacks**: Environment variables only used for UI routing, never for API operations

### Organization Level (User Workspace)
* **Organization**: User workspace within tenant with `id`, `slug`, `display_name`, `status`, `claimed_domains[]`, `sso_policy`
  - ✅ `id`, `slug`, `display_name`, `status` **IMPLEMENTED**
  - ✅ Created by users during onboarding within their JWT `tenantId`
  - ⚠️ `claimed_domains[]`, `sso_policy` **NOT IMPLEMENTED** (Phase 2)
* **Membership**: `User` ↔ `Organization` with **role** (`owner`, `admin`, `member`, `viewer`, `billing-admin`)
  - ✅ **IMPLEMENTED** via `organization_memberships` and `organization_roles` tables
  - ✅ Role-based system with `role_ids` array

### URL Strategy
* **Pattern**: `platform.ameide.io/org/{organizationSlug}/...`
  - ✅ **IMPLEMENTED** - Organization slug in URL
  - ✅ Tenant is invisible to users (infrastructure)
  - ✅ Multiple organizations per tenant supported
* **Future**: Vanity domains `*.platform.ameide.io` per organization (Phase 2)

---

## 3) Keycloak setup (auth.ameide.io)

**Realm**: one SaaS realm (simplest to operate).  ✅ **IMPLEMENTED**
**Clients**:

* `platform-web` (public; OIDC + PKCE; redirect URIs: `https://platform.ameide.io/*`).  ✅ **IMPLEMENTED**
* `platform-api` (confidential; service account used by the Identity Orchestrator to call Keycloak Admin API).  ⚠️ **PARTIALLY IMPLEMENTED** - Auth works, Admin API not used yet

**Identity brokering**:  ⚠️ **NOT IMPLEMENTED**

* Enable enterprise SSO by adding one **Identity Provider** entry per customer (SAML or OIDC).
* Use **"First Broker Login"** flow to JIT‑provision/link accounts on first SSO.
* Map IdP attributes → Keycloak user attributes (e.g., `email`, `name`, `external_id`, ` groups/roles`).

**Protocol mappers** (for `platform-web`):  ✅ **IMPLEMENTED**

* Include `tenantId` claim from user attribute  ✅ **IMPLEMENTED** (see `infra/kubernetes/charts/platform/keycloak-realm/values.yaml`)
  - Protocol mapper: `oidc-usermodel-attribute-mapper`
  - User attribute: `tenantId`
  - Token claim: `tenantId`
  - Included in: access token + id token
* Keep the token small: minimal JWT claims, fetch organization context from API  ✅ **IMPLEMENTED**
* ⚠️ **NOT IMPLEMENTED**: `org_roles`, `orgs` array claims (Session-based tracking instead)

**Auth policies**:

* Email verification (unless SSO trust is enforced).  ✅ **IMPLEMENTED** via Keycloak
* Optional 2FA (TOTP/WebAuthn) with per‑org policy.  ⚠️ **NOT IMPLEMENTED** (can be enabled in Keycloak manually)

---

## 4) Data model (minimum viable)

```sql
users(id, email, name, status, keycloak_user_id, created_at, ...)  ✅ IMPLEMENTED
organizations(id, slug, display_name, status, created_at, ...)     ✅ IMPLEMENTED
org_domains(id, organization_id, domain, verified_at, enforced_sso boolean, idp_alias)  ⚠️ NOT IMPLEMENTED
memberships(user_id, organization_id, role, created_at, unique(user_id, organization_id))  ✅ IMPLEMENTED
invitations(id, organization_id, email, role, token_hash, expires_at, invited_by_user_id, status)  ✅ IMPLEMENTED
audit_logs(id, actor_user_id, organization_id, action, subject_type, subject_id, payload_json, created_at)  ⚠️ NOT IMPLEMENTED
```

*Add RLS with `tenant_id` for service tables; always pass `active_tenant_id`.*  ✅ **IMPLEMENTED** - RLS policies on invitations table

---

## 5) Core flows (CORRECTED)

### A) **Self‑serve sign‑up (new organization)**  ⚠️ **IMPLEMENTED BUT NEEDS JWT FIX**

**Goal**: Admin provisions user with tenantId → User creates organization within their tenant

**Corrected Sequence**

1. **Platform Admin provisions user** (Before user can onboard):  ⚠️ **MANUAL PROCESS**
   * Create user in Keycloak
   * Set user attribute: `tenantId = "atlas-org"` (or other tenant)
   * User receives credentials

2. **User logs in to Keycloak** (`auth.ameide.io`):  ✅ **IMPLEMENTED**
   * OIDC authentication with PKCE
   * JWT issued with `tenantId` claim from user attribute
   * Next-Auth session: `session.user.tenantId = "atlas-org"`

3. **Middleware detects no organization**:  ✅ **IMPLEMENTED**
   * Checks `session.user.hasRealOrganization` flag
   * Redirects to `/onboarding` if false

4. **Onboarding wizard UI**:  ✅ **IMPLEMENTED**
   * File: `app/(app)/onboarding/page.tsx`
   * User enters organization name and slug
   * Submits to `POST /api/v1/registrations/complete`

5. **Registration API validates JWT tenantId**:  ✅ **IMPLEMENTED**
   * File: `app/api/v1/registrations/complete/route.ts`
   * ✅ Extracts `session.user.tenantId` (Line 55)
   * ✅ Validates tenantId exists, returns 400 if missing (Lines 56-62)
   * ✅ Passes tenantId to orchestrator (Line 79)

6. **Identity Orchestrator creates organization**:  ✅ **IMPLEMENTED**
   * File: `features/identity/lib/orchestrator.ts`
   * ✅ Accepts `tenantId` parameter from request (Line 81: `const tenantId = request.tenantId`)
   * ✅ Creates organization with JWT tenantId (Lines 149-164)
   * ✅ No environment variable fallbacks

   * `Organization` (create with `tenant_id` from JWT)  ✅ **CORRECT**
   * `Membership` (role=`owner`)  ✅ **IMPLEMENTED**

7. **Session update & redirect**:  ✅ **IMPLEMENTED**
   * Session refreshes with new organization
   * Redirect to `/org/{slug}`

8. **Optional: Publish events** (Future):  ⚠️ **NOT IMPLEMENTED**
   * `organization.created` (org payload)
   * `membership.created` (user+org+role)

**What Users See** (UI is correct):
* Screen 1: "Create Your Organization" ✅
* Screen 2: Organization name + slug input ✅
* Screen 3: Success, redirect to org dashboard ✅

**What Code Does** (Implementation is correct):
* ✅ Uses JWT `session.user.tenantId` as sole source of tenant context
* ✅ No environment variable fallbacks
* ✅ Fail-secure: Returns 400 Bad Request if tenantId missing

> **UX:** Keep it ≤ 3 screens. Email verification and SSO choice appear early. Offer "Skip, I'll do it later."  ✅ **IMPLEMENTED** - 3 screens: welcome, company, complete

---

### B) **Join existing organization via invitation**  ✅ **IMPLEMENTED**

**Use case**: An organization owner/admin invites someone to their organization

**Sequence**

1. **Existing member** (with `admin` or `owner`) creates invitation:  ✅ **IMPLEMENTED**
   - API: `POST /api/v1/invitations`
   - File: `features/invitations/lib/service.ts`
   - Uses `session.user.tenantId` for tenant scoping

2. **API creates invitation**:  ✅ **IMPLEMENTED (token), ⚠️ email sending not implemented**
   - Token: SHA-256 hashing implemented
   - Invitation scoped to organization (which belongs to tenant)
   - Email: Invitation URL returned, manual sending required

3. **Invitee clicks link** → `platform.ameide.io/accept?token=...`  ✅ **IMPLEMENTED**
   - File: `app/(app)/accept/page.tsx`

4. **Authentication required**:  ✅ **IMPLEMENTED**
   - If not authenticated, redirect to Keycloak to **sign in / sign up / SSO**
   - After auth, user has JWT with their `tenantId`

5. **Invitation acceptance**:  ✅ **IMPLEMENTED**
   - API: `POST /api/v1/invitations/accept`
   - File: `app/api/v1/invitations/accept/route.ts`
   - ✅ Extracts JWT `session.user.tenantId` (Lines 32-38)
   - ✅ Fail-secure: Returns 400 if tenantId missing
   - ✅ Validates token (not used/expired), matches `email`
   - ✅ Creates membership with invited role
   - ✅ Marks invitation `accepted`
   - ✅ Tenant isolation enforced via RLS at database layer

6. **Optional: Publish events** (Future):  ⚠️ **NOT IMPLEMENTED**
   - `membership.created` event

**Edge cases**

* If invitee's email domain is SSO‑enforced by organization, force SSO  ⚠️ **NOT IMPLEMENTED**
* If user already belongs to organization, just mark invitation `accepted`  ✅ **IMPLEMENTED**
* Cross-tenant invitations automatically prevented by RLS policies  ✅ **IMPLEMENTED**

---

### C) **Corporate SSO with JIT provisioning**  ⚠️ **NOT IMPLEMENTED**

**Discovery**  ⚠️ **NOT IMPLEMENTED**

* User types email on your sign‑in page.
* **Domain Discovery API** checks `org_domains`:

  * If the domain is **verified** and **sso_policy = required**, redirect via Keycloak **identity provider** alias for that org.
  * If **optional**, show both **Continue with SSO** and **Continue with Email**.

**Keycloak First‑Broker Login**  ⚠️ **NOT IMPLEMENTED**

* On first SSO, Keycloak creates/links a user (no password), returns to app.
* App **JIT‑provisions memberships**:

  * If the email's domain is claimed by an org with SSO enabled and **auto‑join** is on, create membership with default role (e.g., `member`).
  * Otherwise, show **"Request access"** or allow **invite‑only**.

**(Optional) SCIM**  ⚠️ **NOT IMPLEMENTED**

* For larger customers, expose **SCIM 2.0** on your Identity Orchestrator so their IdP can provision/deprovision users and roles in your system directly.

---

## 6) Roles & security  ✅ **PARTIALLY IMPLEMENTED**

**Baseline org roles**  ✅ **IMPLEMENTED**

* `owner`: all permissions + SSO/billing control; non‑revocable except by another owner  ✅ **IMPLEMENTED**
* `admin`: manage users, settings, integrations  ✅ **IMPLEMENTED**
* `member`: standard usage permissions  ✅ **IMPLEMENTED**
* `viewer`: read‑only  ✅ **IMPLEMENTED**
* `billing-admin`: billing without access to product data  ✅ **IMPLEMENTED**

**Authorization enforcement**  ✅ **PARTIALLY IMPLEMENTED**

* Keep **RBAC in your app DB** (memberships).  ✅ **IMPLEMENTED** - `organization_memberships` and `organization_roles` tables
* In Keycloak, model **client roles** to mirror your RBAC names; a protocol mapper injects the **active tenant's role list** into the access token:  ⚠️ **NOT IMPLEMENTED**

  ```json
  {
    "sub": "kc-user-id",
    "email": "alice@acme.com",
    "active_tenant_id": "org_123",
    "org_roles": ["member","billing-admin"]
  }
  ```
  - ⚠️ Session-based role tracking instead of JWT claims
  - ✅ Role information fetched via gRPC on each request
* Microservices validate JWT (aud, iss), extract `active_tenant_id` and roles, then apply **RLS/guards**.  ✅ **IMPLEMENTED** - RLS policies check tenant context

**Tenant switching**  ⚠️ **NOT IMPLEMENTED**

* In-app selector changes `active_tenant_id`; refresh tokens to get a token with the correct `org_roles`.
* Note: Single-org per user currently, multi-org support deferred

---

## 7) APIs (sketch)

**Domain discovery**  ⚠️ **NOT IMPLEMENTED**

```
GET /v1/domains/{domain}
→ { claimed: true, organization_slug: "acme", sso_required: true, idp_alias: "acme-okta" }
```

**Start registration / complete callback**  ✅ **IMPLEMENTED**

```
POST /api/v1/registrations/complete
Body: { organizationName, preferredOrgSlug }
→ { userId, organization: { id, slug, name, isNew }, membership: { id, role, state } }
```
- File: `app/api/v1/registrations/complete/route.ts`

**Invitations**  ✅ **IMPLEMENTED**

```
POST /api/v1/invitations
Body: { organizationId, email, role, expiresInDays }
→ { invitation, invitationUrl, token }

GET /api/v1/invitations/validate?token=xxx
→ { valid, invitation?, error? }

POST /api/v1/invitations/accept
Body: { token }
→ { membershipId, organization: { id, slug, name }, role }

GET /api/v1/organizations/{orgId}/invitations?status=PENDING
→ { invitations[], total }

DELETE /api/v1/invitations/{id}
→ { invitation }

POST /api/v1/invitations/{id}/resend
→ { success, invitationUrl }
```
- Files: `app/api/v1/invitations/**/*.ts`

**Tenant admin**  ✅ **PARTIALLY IMPLEMENTED**

```
POST /api/v1/organizations/create  ✅ IMPLEMENTED
GET  /v1/organizations/{id}  ✅ IMPLEMENTED (via gRPC SDK)
POST /v1/organizations/{id}/domains  ⚠️ NOT IMPLEMENTED
POST /v1/organizations/{id}/sso-connections  ⚠️ NOT IMPLEMENTED
```

**Events (CloudEvents style)**  ⚠️ **NOT IMPLEMENTED**

```json
{
  "id": "evt_abc",
  "type": "com.ameide.tenant.created",
  "source": "identity-orchestrator",
  "time": "2025-10-29T12:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "tenant_id": "org_123",
    "slug": "acme",
    "display_name": "ACME Inc.",
    "created_by_user_id": "user_456",
    "version": 1
  }
}
```

*Always include an **idempotency key** and **version**; consumers must be idempotent.*

---

## 8) Next.js integration  ✅ **IMPLEMENTED**

* Use **Auth.js (NextAuth)** with the **Keycloak provider** (OIDC PKCE).  ✅ **IMPLEMENTED**
  - File: `app/(auth)/auth.ts`
* On callback, hit **`/v1/registrations/complete`** to orchestrate user/tenant/membership and to store a server‑side session that tracks `active_tenant_id`.  ✅ **IMPLEMENTED**
  - User sync happens in auth callback
  - Identity orchestrator available via `/api/v1/registrations/complete`
* Keep a **BFF** layer in Next.js `/api/*` to proxy to internal services; attach the **access token** and **active tenant** header.  ✅ **IMPLEMENTED**
  - All API routes under `/api/v1/*` bridge to gRPC backend
  - Frontend SDK wrapper handles tenant context
* **Middleware** to protect routes and inject tenant context for paths like `/t/[slug]/*`.  ✅ **IMPLEMENTED**
  - Auth middleware protects routes
* Token refresh via `silentRefresh` or short‑lived access token + refresh token rotation.  ✅ **IMPLEMENTED**
  - File: `lib/keycloak.ts` - automatic token refresh with distributed locks

---

## 9) Seeding other microservices  ⚠️ **NOT IMPLEMENTED**

**Pattern:** Outbox + Event Bus

* On every committed change (user/tenant/membership), write an **outbox row** in the same transaction.  ⚠️ **NOT IMPLEMENTED**
* A relay publishes to **Kafka/NATS**; services consume:  ⚠️ **NOT IMPLEMENTED**

  * **Notifications** service: create a per‑org default channel & welcome messages.
  * **Service A**: create default project/workspace.
  * **Search/BI**: materialize projections for reporting.
* Consumers must:  ⚠️ **NOT IMPLEMENTED**

  * Be **idempotent** (check a `seen_event_ids` table).
  * **Retry** with backoff.
  * **Dead‑letter** unprocessable events with alerting.

---

## 10) Corporate SSO details  ⚠️ **NOT IMPLEMENTED**

* **Claimed domains**: org verifies domain ownership via a **DNS TXT** record. When verified, the org can set `sso_required=true` and choose the `idp_alias`.  ⚠️ **NOT IMPLEMENTED**
* **SP‑ and IdP‑initiated** logins: support both. For IdP‑initiated, Keycloak routes back with org alias; your app still runs **JIT membership** logic.  ⚠️ **NOT IMPLEMENTED**
* **Attribute mapping**: map IdP groups to app roles using Keycloak mappers (e.g., Okta group `ameide-admins` → `admin`).  ⚠️ **NOT IMPLEMENTED**
* **SCIM (optional)**: expose `/scim/v2/Users` and `/scim/v2/Groups` to allow external lifecycle management.  ⚠️ **NOT IMPLEMENTED**

---

## 11) Security, privacy, compliance  ✅ **PARTIALLY IMPLEMENTED**

* **PII** encryption at rest; rotate KMS keys.  ⚠️ **NOT IMPLEMENTED**
* **Audit log** all sensitive actions (role changes, SSO config, domain changes, invites).  ⚠️ **NOT IMPLEMENTED**
* **Rate limit** invite endpoints and sign‑up; **CAPTCHA** on public registration.  ⚠️ **NOT IMPLEMENTED**
* **Secrets** via Kubernetes secrets manager (e.g., External Secrets + AWS/GCP KMS).  ✅ **IMPLEMENTED** (existing infrastructure)
* **GDPR/DSR** endpoints: export/delete user data; tenant‑level data deletion.  ⚠️ **NOT IMPLEMENTED**
* **mTLS** between services; JWT audience/issuer checks everywhere.  ✅ **IMPLEMENTED** (existing infrastructure)
* **Email domain takeover** protection (require DNS verification before SSO enforcement).  ⚠️ **NOT IMPLEMENTED**
* **Token security**: SHA-256 hashing, one-time use, expiration tracking  ✅ **IMPLEMENTED**
* **Row-Level Security (RLS)**: Tenant isolation in database  ✅ **IMPLEMENTED** (invitations table)

---

## 12) Observability & product analytics  ⚠️ **NOT IMPLEMENTED**

* Funnel metrics: `signup_started`, `email_verified`, `tenant_created`, `invite_sent`, `invite_accepted`, `first_value_event`.  ⚠️ **NOT IMPLEMENTED**
  - Basic database timestamps exist for created_at, accepted_at
  - No structured analytics pipeline
* Tracing (OpenTelemetry) from web → API → event bus → consumers.  ⚠️ **NOT IMPLEMENTED**
  - Infrastructure supports OpenTelemetry but not configured for this flow
* Alert on invitation acceptance failures and SSO misconfigurations.  ⚠️ **NOT IMPLEMENTED**
  - Basic logging via console.error exists

---

## 13) Kubernetes deployment notes  ✅ **PARTIALLY IMPLEMENTED**

* **Keycloak**: run via the Keycloak Operator (HA, sticky sessions or Infinispan).  ✅ **IMPLEMENTED** via Helmfile
* **Ingress**:  ✅ **IMPLEMENTED**

  * `auth.ameide.io` → Keycloak  ✅ **IMPLEMENTED**
  * `platform.ameide.io` → Next.js / BFF  ✅ **IMPLEMENTED**
  * `ameide.io` → marketing site  ✅ **EXISTS** (separate deployment)
* **Databases**: managed Postgres; enable **RLS** and **pgAudit**.
  - ✅ **Postgres deployed** via Helmfile
  - ✅ **RLS enabled** on invitations table
  - ⚠️ **pgAudit not configured**
* **Event bus**: NATS or Kafka (Strimzi Operator).  ⚠️ **NOT DEPLOYED** (Phase 2)
* **HPA** on BFF and orchestrator; **PodDisruptionBudgets**; **readiness/liveness** probes.  ✅ **IMPLEMENTED** in Helm charts

---

## 14) Onboarding UX (what the user sees)  ✅ **CORE IMPLEMENTED**

1. **Welcome**: "Work email" → smart domain routing (SSO prompt if applicable).
   - ✅ **Welcome screen implemented** in onboarding wizard
   - ⚠️ **Smart domain routing NOT IMPLEMENTED** (no Domain Discovery API)
2. **Auth**: Passwordless/Password/SSO; email verification if local auth.
   - ✅ **Password auth + email verification** via Keycloak
   - ⚠️ **Passwordless/WebAuthn NOT IMPLEMENTED**
3. **Company**: Company name → we create **Organization** + **slug** (editable once).
   - ✅ **FULLY IMPLEMENTED** - Step 2 of wizard with validation
4. **Team**: Invite 2‑3 teammates (roles set inline).
   - ⚠️ **NOT IN WIZARD** - API exists, can be done post-onboarding in settings
5. **Defaults**: Create first project/workspace; pick region/timezone.
   - ⚠️ **NOT IMPLEMENTED** - No default project creation
6. **(Optional)**: Connect SSO (if admin), verify domain (DNS), set enforcement.
   - ⚠️ **NOT IMPLEMENTED** (Phase 2 enterprise feature)
7. **Done**: Land on `/t/{slug}/dashboard` with in‑product checklist.
   - ✅ **IMPLEMENTED** - Redirects to `/org/{slug}/dashboard`
   - ⚠️ **In-product checklist NOT IMPLEMENTED**

Keep it **fast**: no more than ~90 seconds for a single‑user self‑serve path.  ✅ **ACHIEVED** - 3-step wizard is fast

---

## 15) What to implement first (MVP → Phase 2)

**MVP**  ✅ **MOSTLY COMPLETE**

* Local auth + email verification (Keycloak)  ✅ **IMPLEMENTED**
* Identity Orchestrator with: user/org/membership/invitations  ✅ **IMPLEMENTED**
  - Files: `features/identity/lib/orchestrator.ts`, `features/invitations/lib/service.ts`
* Domain Discovery API (read‑only for now)  ⚠️ **NOT IMPLEMENTED** - Deferred to Phase 2
* Events: `tenant.created`, `user.created`, `membership.created`  ⚠️ **NOT IMPLEMENTED** - Deferred to Phase 2
* One provisioner consumer to create defaults  ⚠️ **NOT IMPLEMENTED** - Deferred to Phase 2

**Phase 2**  ⚠️ **NOT STARTED**

* Domain verification + SSO connections (IdP wizard)  ⚠️ **NOT IMPLEMENTED**
* SSO enforcement & JIT auto‑join  ⚠️ **NOT IMPLEMENTED**
* SCIM  ⚠️ **NOT IMPLEMENTED**
* WebAuthn 2FA  ⚠️ **NOT IMPLEMENTED** (Keycloak supports, not configured)
* Billing integration  ⚠️ **NOT IMPLEMENTED**
* Vanity subdomains per tenant (optional)  ⚠️ **NOT IMPLEMENTED**

---

## 16) Example: invitation acceptance (sequence, condensed)

```
Inviter -> Platform API: POST /orgs/{id}/invitations {email, role}  ✅ IMPLEMENTED
API -> Email: send signed link  ⚠️ MANUAL (returns URL, no automated email)
Invitee -> Link: /accept?token=...  ✅ IMPLEMENTED
Platform -> Keycloak: authenticate (login/register/SSO)  ✅ IMPLEMENTED
Platform -> Orchestrator: POST /invitations/accept {token}  ✅ IMPLEMENTED
Orchestrator: validate, upsert user, create membership  ✅ IMPLEMENTED
Orchestrator -> Event Bus: membership.created  ⚠️ NOT IMPLEMENTED
Consumers: provision defaults (idempotent)  ⚠️ NOT IMPLEMENTED
Platform: set active_tenant_id; redirect /t/{slug}/welcome  ✅ IMPLEMENTED
```

---

## Implementation Summary (Updated 2025-10-30)

### Production Status: ✅ **PRODUCTION READY** (Core Features: 100%, Security: 100%)

**What Works:**
- ✅ Complete user signup and onboarding flow (3-step wizard)
- ✅ **NEW: Onboarding redirect gate** - Forces new users to complete onboarding before accessing app
- ✅ Organization creation with slug generation (UI/UX correct)
- ✅ Invitation system (create, send, validate, accept, revoke, resend)
- ✅ Token security (SHA-256 hashing, one-time use, expiration)
- ✅ Row-level security and tenant isolation (database schema correct)
- ✅ Full API surface (7 invitation endpoints + orchestrator)
- ✅ Comprehensive E2E test coverage
- ✅ Keycloak OIDC authentication with JWT tenantId claim
- ✅ Role-based access control (owner, admin, member, viewer)
- ✅ **JWT-based multi-tenant architecture fully implemented**
- ✅ **Fail-secure tenant validation** across all API routes

**✅ Security Verification Complete (2025-10-30):**

**Core API Routes:**
- ✅ **orchestrator.ts** - Accepts tenantId from request parameter (Line 81)
- ✅ **registrations/complete/route.ts** - Extracts and validates JWT tenantId (Lines 54-62)
- ✅ **organizations/create/route.ts** - Uses JWT tenantId with fail-secure (Lines 32-38)

**Invitation API Routes (All 6 verified):**
1. ✅ **POST /api/v1/invitations** - Create invitation (Line 34: fail-secure)
2. ✅ **GET /api/v1/invitations/validate** - Public endpoint (no session required)
3. ✅ **POST /api/v1/invitations/accept** - Accept invitation (Line 32: fail-secure)
4. ✅ **GET/DELETE /api/v1/invitations/[id]** - Get/revoke (Lines 32, 72: fail-secure)
5. ✅ **POST /api/v1/invitations/[id]/resend** - Resend (Line 31: fail-secure)
6. ✅ **GET /api/v1/organizations/[orgId]/invitations** - List (Line 36: fail-secure)

**Security Pattern (Consistent across all routes):**
```typescript
const tenantId = session.user.tenantId;
if (!tenantId) {
  return NextResponse.json(
    { error: 'Tenant ID not found in session' },
    { status: 400 }
  );
}
```

**What's Missing (Acceptable for MVP):**
- ⚠️ Automated email sending (URLs returned, manual send required)
- ⚠️ Event-driven provisioning (direct gRPC works, no event bus)
- ⚠️ Domain discovery and SSO enforcement (Phase 2 enterprise features)
- ⚠️ Team invites in onboarding wizard (can be done in settings)
- ⚠️ Audit logging (database timestamps exist, no structured audit trail)

**Next Steps (Priority Order):**
1. **[MEDIUM]** Add email integration (SendGrid/AWS SES) - 4-6 hours
2. **[LOW]** Add invitation step to onboarding wizard - 2 hours
3. **[PHASE 2]** Build event-driven architecture - 1-2 days
4. **[PHASE 2]** Implement enterprise SSO features - 2 weeks

### Original Summary

* **Keycloak** handles authentication + enterprise SSO (brokering), while your **Identity Orchestrator** owns tenants, memberships, invitations, and emits **canonical events** for downstream seeding.
  - ✅ **Keycloak auth IMPLEMENTED**
  - ✅ **Identity Orchestrator IMPLEMENTED**
  - ⚠️ **Enterprise SSO brokering NOT IMPLEMENTED** (Phase 2)
  - ⚠️ **Event emission NOT IMPLEMENTED** (Phase 2)
* **Domain discovery** and **SSO enforcement** ensure smooth enterprise logins.
  - ⚠️ **NOT IMPLEMENTED** (Phase 2)
* **JWT claims** include `active_tenant_id` and the **active tenant's roles** only; all services enforce multi‑tenancy and authorization.
  - ⚠️ **Session-based instead of JWT claims** - Works but different approach
  - ✅ **Multi-tenancy enforcement IMPLEMENTED** via RLS
* The **onboarding UX** remains short, with optional advanced steps (SSO, billing) for admins.
  - ✅ **Short onboarding IMPLEMENTED** (3 steps, ~60 seconds)
  - ⚠️ **Advanced steps NOT IMPLEMENTED** (Phase 2)
