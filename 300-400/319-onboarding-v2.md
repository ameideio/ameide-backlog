> **DEPRECATED**: This backlog has been superseded by [428-onboarding.md](./428-onboarding.md).
>
> The content below is retained for historical reference. All new work should follow backlog 428.

---

## Backlog 319 – Realm-Per-Tenant Onboarding (v2)

This document defines the **end-to-end onboarding flow** for the target architecture where each customer tenant receives its own Keycloak realm (no fallbacks, no partial provisioning). It supersedes the interim guidance in `backlog/319-onboarding.md` and pulls together decisions from:

- `backlog/333-realms.md` – Realm-per-tenant architecture ADR  
- `backlog/331-tenant-resolution.md` – How tenant IDs flow through auth/session layers  
- `backlog/324-user-org-settings.md` – Post-onboarding organization & membership management

---

### 1. Two-Level Model Recap

| Layer           | Purpose                                                                 | Provisioned By            | Backlog reference                 |
|-----------------|-------------------------------------------------------------------------|---------------------------|-----------------------------------|
| **Tenant**      | Infrastructure container (DB schema, billing account, data residency); powers realm-per-tenant isolation. | Platform ops or automated lead intake. | `331-tenant-resolution.md`        |
| **Organization**| Workspace created during onboarding: holds repositories, transformations, members. | End user via onboarding wizard. | `324-user-org-settings.md`        |

During onboarding the system first **activates the tenant** (if still pending) and then **creates the first organization** inside that tenant, attaching the user as owner.

---

### 2. Target Onboarding Flow (no workarounds)

1. **Bootstrap**  
   - Option A (ops/CRM): marketing flags a new lead and creates a pending catalog record (`status = pending_onboarding`).  
   - Option B (self-serve): the landing page calls `POST /api/v1/registrations/bootstrap` with company name + optional tenant slug. The API creates the pending catalog entry via `tenantService.createTenant`, stamps the Keycloak `tenantId` attribute, and returns the new identifier.  
   - The bootstrap handler always uses the Keycloak subject (`kcSub`) as `ownerUserId` **and** for the gRPC request context; this keeps the tenant owner aligned with the identifier the orchestrator passes back into platform services during activation.  
   - In both cases the tenant catalog owns the UUID `tenantId`; human-friendly slugs remain metadata.  
   - The `tenantId` is **reserved during this stage** so billing/CRM systems, data residency routing, and Ops tooling can reference a stable identifier, and so the eventual Keycloak registration can set the `tenantId` attribute without fallbacks.

2. **User self-registration**  
   - User visits `platform.ameide.io/register` (CTA, invite, or self-serve).  
   - Registration page redirects to the **landing realm** (shared, e.g. `ameide`).  
   - After the user completes sign-up the client checks `session.user.tenantId`. If it is missing, the wizard calls the bootstrap endpoint (Option B above), refreshes the session (`updateSession({ refreshTenant: true })`), and only then proceeds.  
   - Auth.js never invents tenant IDs; the bootstrap step must succeed or onboarding returns HTTP 409.

3. **Onboarding gate**  
   - Middleware sees `session.user.hasRealOrganization === false` ⇒ redirect to `/onboarding`.  
   - Wizard asks the user to supply:  
     1. **Tenant slug** (optional, defaults to slugified company name; if the bootstrap step already ran, this value refines the existing tenant slug).  
     2. **Organization name** (required).  
     3. **Organization slug** (optional; persisted for vanity URLs and marketing collateral, still defaults to slugified name). The application now routes with the canonical organization ID (`/org/{id}`) immediately after onboarding to avoid stale links when slugs change.
   - UUIDs remain the canonical keys (`tenantId`, `organizationId`); slugs are for navigation and display.

4. **POST `/api/v1/registrations/complete`**  
   - Request payload: `{ organizationName, preferredOrgSlug?, tenantSlug? }`.  
   - Handler fetches the session, validates the pending tenant (no bootstrapping at this stage), and calls `IdentityOrchestrator.completeRegistration`.  
   - Response returns tenant, organization, membership.

5. **`IdentityOrchestrator.completeRegistration` responsibilities**  
   1. Ensure user has no existing org memberships.  
   2. Generate org slug/ID.  
   3. **Create Keycloak realm for the tenant** via service account with master-realm `realm-admin` + `create-realm`:  
      - Realm attributes store `organizationId` and `tenantId`.  
      - Seed realm roles (`org-admin`, `org-member`, `org-guest`).  
      - Create `platform-app-{slug}` client, configure redirect URIs, upload secret in Vault.  
      - **Immediately refresh the Keycloak admin token cache after realm creation.** Tokens minted before the realm existed will not include the new realm's `realm-management` grants; the orchestrator must invalidate the cached token so subsequent client/user provisioning runs with a freshly issued token that can access the tenant realm.  
      - In environments where Azure credentials are unavailable (local dev, CI sandboxes), set `AUTH_SKIP_KEYVAULT_PERSISTENCE=true` (or `KEYVAULT_WRITE_OPTIONAL=true`) so onboarding logs a warning and continues when Key Vault writes fail. Production must keep secret persistence mandatory.
      - Provision the current user inside the new realm, assign `org-admin`.  
  4. Call platform gRPC `organizationService.createOrganization` to persist org + membership (including `realmName`). The TypeScript handler inserts the initial membership inside the same transaction; the running `@ameide/platform-service` build must include this logic (i.e., `pnpm build` before deploy) otherwise onboarding will fail when it cannot resolve the membership.  
  5. Update tenant catalog entry to `active` and (once the event bus contract is available) emit `tenant.created`/`organization.created` events.
  6. Fail fast if any upstream call returns an error—surface the failure to the client and log it; do not mint placeholder orgs, skip catalog updates, or silently continue.

> **Platform schema dependency**  
> The platform service must expose a `realm_name` (text) column on the `organizations` table (and corresponding protobuf/ORM fields). Without this migration the gRPC call in step 5.4 will fail with `column "realm_name" ... does not exist`, aborting onboarding after the realm has already been provisioned. Keep the database migration in lock-step with onboarding rollout.

6. **Finalize onboarding**  
   - API responds with the structure below.  
   - Front-end calls `updateSession({ refreshOrganizations: true, refreshTenant: true, preferredOrgId: orgSlugOrId, organizationIds: [orgSlugOrId], hasRealOrganization: true })`, both immediately after the API resolves and again before navigating away from the success screen (`orgSlugOrId` = newly created organization slug when present, otherwise the GUID).  
   - Auth cookies are scoped to `.dev.ameide.io` (via `AUTH_COOKIE_DOMAIN`) so the session ID survives redirects across `platform.dev.ameide.io`, Envoy, and dev hosts. Without this, the middleware cannot see the refreshed session and onboarding loops.  
   - Auth.js JWT callback reloads tenant/org lists from the platform API and sets `hasRealOrganization = true`. `ensureUserTenant` receives the Keycloak subject (`kcSub`), and `applyOrganizationContext` now unions the slug hints from `updateSession` with the Keycloak group claims so the freshly created membership is recognised even before Keycloak propagates realm groups (hints also flip `hasRealOrganization` true if the fetch is still empty, preventing the wizard loop). As a final guard, if the refreshed token resolves an organization context it forces `hasRealOrganization = true` before returning so the middleware never regresses.  
   - Middleware now allows the organization dashboard (`/org/{organizationId}`) to load; user lands on the workspace overview and can pivot into `/org/{organizationId}/settings` directly from the success state.

```jsonc
{
  "tenantId": "tenant-xyz",
  "organization": {
    "id": "org-123",
    "slug": "acme",
    "name": "ACME",
    "realmName": "acme",
    "isNew": true
  },
  "membership": {
    "id": "membership-abc",
    "role": "org-admin",
    "state": "ACTIVE"
  }
}
```

7. **Subsequent logins**  
   - User still authenticates with the landing realm, but the session refresh always pulls the active organization list; `hasRealOrganization` stays true so the wizard never reappears.

8. **Inviting additional users**  
   - Uses the invitation flows documented in `324-user-org-settings.md`.  
   - Invitees authenticate against the **organization realm** (`platform-app-{slug}`) once the realm provisioning is complete.

---

### 3. Hard Requirements (no fallbacks allowed)

| Capability | Description | Responsible backlog |
|------------|-------------|---------------------|
| Keycloak admin client | Dedicated service account (e.g., `platform-admin-client`) with master-realm `realm-admin` + `create-realm`. | `333-realms.md` |
| Tenant catalog + API | `organizationService.createOrganization`, tenant status transitions, event emission. | `331-tenant-resolution.md` |
| Onboarding wizard UX | Validates org slug/name, handles success/error messaging, refreshes session flags. | `324-user-org-settings.md` |
| Session refresh | `session.user.hasRealOrganization` must rely solely on platform API responses—no “atlas” fallback. | `331-tenant-resolution.md` |
| Cookie domain alignment | `AUTH_COOKIE_DOMAIN` / `NEXTAUTH_URL` must scope cookies to `.dev.ameide.io` so NextAuth sessions persist when proxied through ingress; localhost (`0.0.0.0:3001`) requests break the gating loop. | `331-tenant-resolution.md` |
| Platform build pipeline | Deployments must ship a freshly built `services/platform/dist` bundle so organization membership creation runs during onboarding. | Ops release checklist |
| Error handling | Provisioning failures must bubble to the caller; no silent retries or substitute records. | `333-realms.md` |
| Platform org listing | `organizationService.listOrganizations` falls back to membership lookup when Keycloak groups are absent (new tenant realms). | `319-onboarding-v2.md` |

---

### 4. Outstanding Implementation Items

1. **Redeploy telemetry + web stack** – `helmfile -e local --file infra/kubernetes/helmfile.yaml sync` now needs to run end-to-end (without skipping Alloy) so Alloy gateway/logs, `platform`, and `www-ameide-platform` are re-installed with the new config.  
2. **Platform service availability** – Confirm the `platform` release is restored (Helm removed it during the last sync); without it the gRPC orchestrator calls still fail.  
3. **Tenant catalog telemetry** – `/api/v1/registrations/bootstrap` provisions pending tenants; add rate limiting, analytics, and abuse monitoring.  
4. **Catalog events** – Wire `tenant.created` / `organization.created` emission into the orchestrator once the downstream event bus is ready.  
5. **Metrics & auditing** – Capture onboarding completion events to feed analytics pipeline (`backlog/324-user-org-settings.md`).
6. **Cookie domain config rollout** – Update Helm/Tilt values for `AUTH_COOKIE_DOMAIN`, `NEXTAUTH_URL`, and `allowedDevOrigins`, then redeploy `www-ameide-platform` so browser cookies target `.dev.ameide.io` and the middleware stops redirecting post-success sessions back to `/onboarding`.

---

### 5. Status Snapshot (2025-10-31)

| Area | Status | Notes |
|------|--------|-------|
| UI & gating | ✅ | `/onboarding` wizard + middleware ready. |
| Tenant provisioning | ⏳ | API/bootstrap path works with catalog + BigInt sanitisation; needs redeploy before E2E validation. |
| Realm-per-tenant | ✅ | App now uses the master realm service account to create tenant realms (no more 403). |
| Platform gRPC | ⏳ | `platform` Helm release must be reinstalled after the last sync removed it. |
| Automation | ⚠️ | Hook job disabled; manual DB changes currently required. |

When the dependencies above are met, onboarding will deliver the full “single realm → per-tenant realm” migration path without workarounds; until then, the flow must fail fast and surface the blocking error to the user.

---

### 6. Recent Progress & Validation

- Self-serve bootstrap routed through `/api/v1/registrations/bootstrap`; onboarding wizard now fronts a tenant review step before organization creation and refuses to continue without a session tenant.  
- Bootstrap handler now uses the Keycloak subject (`kcSub`) for both the tenant owner and request context when calling `tenantService.createTenant`, preventing the “tenant not found” activation failure we observed when session IDs differed.  
- `POST /api/v1/registrations/complete` no longer imports `ensureUserTenant`; session tenants are mandatory and return HTTP 409 when absent.  
- JWT refresh path calls `ensureUserTenant` with the Keycloak subject so the tenant attribute is re-synced immediately after onboarding (fixes the “Tenant not provisioned for user” toast).  
- Onboarding success flow now passes `preferredOrgId` / `organizationIds` + `hasRealOrganization` during `updateSession`, and `applyOrganizationContext` merges those hints with group claims so the new membership is visible before Keycloak issues updated tokens—`hasRealOrganization` flips to `true` (even if the initial platform lookup is empty) and navigation escapes the wizard on first try when the app is served from `https://platform.dev.ameide.io`; requests against `0.0.0.0:3001` still loop because cookies never scope to `.dev.ameide.io`.  
- `organizationService.listOrganizations` now derives memberships from `platform.organization_memberships` when no Keycloak group keys are supplied, so freshly provisioned admins can refresh or open a new tab without falling back into the onboarding wizard while realm groups propagate.  
- `organizationMembershipService.listActiveMemberships` now queries by platform user ID first and falls back to the Keycloak subject (`kcSub`) when needed; integration coverage for both branches is still pending (target tests in `services/platform/src/organizations/__tests__`).  
- Success call-to-action now routes with the canonical organization ID, so “Open organization settings” / “Continue to workspace” land on real pages even when the slug changes.  
- Structured logging helper (`@/lib/logging`) adopted by both onboarding API routes for consistent diagnostics.  
- `/register` now mirrors `/login`, initiating a Keycloak registration flow with `screen_hint=signup`/`kc_action=register` so brand-new visitors can create an account without hitting the old login redirect loop.  
- Tenant summary endpoint sanitises BigInts before returning JSON, unblocking the wizard on tenants with bigint metadata.  
- `createOrganizationRealm` now authenticates against the `master` realm using the existing service account secret and supports username/password overrides.  
- Tenant realm provisioning fetches the generated `platform-app-{slug}` client secret from Keycloak and stores it in Azure Key Vault (`platform-app-tenant-{slug}-client-secret`) so downstream services can consume per-tenant credentials without manual rotation.  
- Tests:  
  - `pnpm --filter www-ameide-platform exec jest --runTestsByPath features/onboarding/__tests__/unit/onboarding-wizard.test.tsx`  
  - `pnpm --filter www-ameide-platform exec jest --runTestsByPath features/onboarding/__tests__/integration/registration-api.test.ts`  
  - `pnpm --filter www-ameide-platform exec playwright test onboarding` *(cluster-only; requires `INTEGRATION_MODE=cluster` and a seeded Keycloak test realm with appropriate fixtures).*
