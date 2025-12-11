# Backlog Item 329: Authentication & Authorization Consolidation (2025-11-02 Snapshot)

**Owner**: Platform Authentication & Authorization  
**Status**: In Flight – Realm-per-tenant rollout blocked, single-realm hardened  
**Related**: `333-realms.md`, `331-tenant-resolution.md`, `322-rbac-v2.md`, `323-keycloak-realm-roles.md`

---

## 1. Architecture at a Glance

| Layer | Current State | Notes |
|-------|---------------|-------|
| Identity Provider | Single Keycloak realm (`ameide`) backed by KC groups | Realm-per-tenant ADR written but not live. |
| Session broker | NextAuth (Node runtime) with Redis session store | Custom JWT callback refreshes Keycloak access token + merges org context. |
| Tenant context | `tenantId` issued as Keycloak attribute → JWT claim → session | Eliminated `DEFAULT_TENANT_ID` fallbacks in API layer; still present in legacy SDK/helpers. |
| Organization context | JWT session carries `organization`, `organizations`, `defaultOrgId`, `hasRealOrganization` | `hasRealOrganization` gates onboarding; roles pulled via new membership lookup helper. |
| RBAC enforcement | Platform API enforces via gRPC `requireOrganizationRole`, UI reads `session.user.roles` | Role lookup now reconciles platform id ↔ Keycloak subject before populating session. |

---

## 2. Recent Hardening Work

- **Session ↔ Platform identity alignment**  
  - NextAuth keeps the platform-generated user id as the canonical `session.user.id`.  
  - New helper `fetchOrganizationMembershipRoles` resolves membership roles using the platform id and falls back to `kcSub` when required.  
  - On JWT refresh we now attach real membership roles (not just hints), guaranteeing `hasRealOrganization` survives logout/login sequences.
- **Keycloak infrastructure parity**  
  - Keycloak operator instance now references the CNPG-owned `postgres-ameide-auth` credentials; ExternalSecrets managed by the chart hydrate admin/master secrets only.  
  - Helm smoke tests exercise realm bootstrap and client patching end-to-end, preventing regressions during future realm-per-tenant work.

- **Onboarding flow reliability**  
  - `IdentityOrchestrator.completeRegistration` accepts both `platformUserId` and `keycloakUserId`, ensuring membership rows and subsequent role lookups align.  
  - `/onboarding` refresh now calls `updateSession` with `refreshOrganizations` + `organizationRoles` hints; middleware routes users back to the org home once `hasRealOrganization` is true.

- **Logout handling**  
  - `/logout` clears NextAuth cookies, optionally fans out to Keycloak logout; middleware routes authenticated users away from `/login`/`/register` to `/onboarding` or org home depending on `hasRealOrganization`.

---

## 3. Known Gaps / Risks

1. **Realm-per-tenant still aspirational**  
   - ADR-333 marks phases as “implemented”, but production still runs a single realm.  
   - No realm bootstrap migration (`V37__organization_realm_mapping.sql`) exists in repo; org catalog still keyed via legacy groups.

2. **SDK defaults leak legacy tenants**  
   - `packages/ameide_sdk_ts/src/platform/organization-client.ts` and graph service helpers continue to default to `DEFAULT_TENANT_ID`.  
   - Downstream scripts/tests assume atlas tenant; needs explicit tenant plumbing to avoid cross-tenant leakage during multi-tenant roll-out.

3. **Pre-onboarding session race**  
   - Immediately after logout/login, session refresh may miss the `hasRealOrganization` flip if the membership fetch fails (e.g., org created but membership not committed yet).  
   - Mitigated by the new membership-role lookup but still dependent on prompt platform-service consistency.

4. **Guest / multi-org membership**  
   - `collectCandidateMemberIds` checks both platform id and `kcSub`, but membership queries default to platform id only.  
   - Federation scenarios (guest users) will need explicit tests to ensure membership rows store platform id to avoid double lookups.

5. **Legacy tests / scaffolding**  
   - Jest aliases for onboarding (e.g., `/onboarding/page.tsx`) cause suites to fail unless moduleNameMapper is extended.  
   - Numerous legacy Flyway migrations were deleted locally; environment rebuilds rely on fresh baseline (requires documentation for a clean bootstrap).

---

## 4. Action Plan

| Priority | Task | Owner | Target |
|----------|------|-------|--------|
| P0 | Correct ADR-333 status + ship migration scripts (`realm_name`, realm provisioning CLI) | Platform Auth | Sprint +1 |
| P0 | Update SDK defaults (TS/Go) to require explicit tenantId, remove `DEFAULT_TENANT_ID` | SDK team | Sprint +1 |
| P1 | Add regression tests for logout/login ensuring `hasRealOrganization` persists | Platform QA | Sprint +2 |
| P1 | Document “drop all data” bootstrap procedure (Keycloak realms + platform seed) | Ops | Sprint +2 |
| P2 | Guest federation roadmap: confirm membership storage uses platform id, add mapping table if necessary | Platform Auth | Sprint +3 |
| P2 | Fix Jest moduleNameMapper for onboarding path to keep unit suite green | DX Tooling | Sprint +1 |

---

## 5. Reference Diagnostics

- **Logs to watch**  
  - `[auth] applyOrganizationContext - organizations fetched` (ensure `count` > 0)  
  - `[auth] JWT callback - organization refresh complete` (look for `newHasRealOrg: true`)  
  - Middleware `[MIDDLEWARE] ...` entries to verify redirects away from `/onboarding` for onboarded users.

- **Manual sanity checks**  
  1. Seed user (`admin66`) completes onboarding -> verify `/api/auth/session` shows `hasRealOrganization: true`, `roles` includes `admin`.  
  2. `/logout` then login -> user lands on `/org/<slug>`, not `/onboarding`.  
  3. Invited user accepts invitation -> middleware should bypass onboarding (hasRealOrganization true).  
  4. `listOrganizations` gRPC call includes platform id first; fallback to `kcSub` only if membership missing.

---

## 6. Glossary

- **Platform user id**: Primary key generated by the platform service; stored in session as `session.user.id`.  
- **Keycloak subject (`kcSub`)**: Stable per realm identifier; stored alongside the platform id for cross-realm lookups.  
- **`hasRealOrganization` flag**: Session marker telling middleware whether to gate `/onboarding`; set true once membership rows occupy the platform DB.  
- **Tenant slug**: Human-readable tenant identifier stored in tenant metadata (`metadata.slug`), used for future realm naming.

---

_Last updated: 2025-11-02 (post admin66 regression fix)._
