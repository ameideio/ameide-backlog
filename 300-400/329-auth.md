# Backlog Item 329: Authentication & Authorization Consolidation (2025-11-02 Snapshot)

**Owner**: Platform Authentication & Authorization  
**Status**: In Flight – Realm-per-tenant rollout blocked, single-realm hardened  
**Related**: `333-realms.md`, `331-tenant-resolution.md`, `322-rbac-v2.md`, `323-keycloak-realm-roles.md`

> **Update (597):** The recommended target state is issuer-first tenant routing (OIDC `iss` issuer URL → server-side `issuer → tenant_id` mapping) and explicit access states (`OK | EMPTY | TIMEOUT | ERROR`). Do not extend legacy patterns like `tenantId` token claims, `x-tenant-id`, or `hasRealOrganization` onboarding gates.

---

## 1. Architecture at a Glance

| Layer | Current State | Notes |
|-------|---------------|-------|
| Identity Provider | Single Keycloak realm (`ameide`) backed by KC groups | Realm-per-tenant ADR written but not live. |
| Session broker | NextAuth (Node runtime) with Redis session store | Custom JWT callback refreshes Keycloak access token + merges org context. |
| Tenant context | Issuer-first routing (`iss` issuer URL → server-side mapping) | Avoid tenantId claims and issuer parsing; treat issuer as a URL. |
| Organization context | Explicit access view (`OK | EMPTY | TIMEOUT | ERROR` + typed issues) | TIMEOUT/ERROR must never be treated as EMPTY; onboarding is not gated by `hasRealOrganization`. |
| RBAC enforcement | Platform API enforces via gRPC `requireOrganizationRole`, UI reads `session.user.roles` | Role lookup now reconciles platform id ↔ Keycloak subject before populating session. |

---

## 2. Recent Hardening Work

- **Session ↔ Platform identity alignment**  
  - NextAuth keeps the platform-generated user id as the canonical `session.user.id`.  
  - New helper `fetchOrganizationMembershipRoles` resolves membership roles using the platform id and falls back to `kcSub` when required.  
  - On refresh we attach real membership roles (not just hints) and preserve explicit access-state semantics (TIMEOUT/ERROR ≠ EMPTY).
- **Keycloak infrastructure parity**  
  - Keycloak operator instance now references the CNPG-owned `postgres-ameide-auth` credentials; ExternalSecrets managed by the chart hydrate admin/master secrets only.  
  - Helm smoke tests exercise realm bootstrap and client patching end-to-end, preventing regressions during future realm-per-tenant work.

- **Onboarding flow reliability**  
  - `IdentityOrchestrator.completeRegistration` accepts both `platformUserId` and `keycloakUserId`, ensuring membership rows and subsequent role lookups align.  
  - UI routing should be driven by access states and lane semantics (see `backlog/597-login-onboarding-primitives.md`), not session flags.

- **Logout handling**  
  - `/logout` clears session cookies and may optionally perform RP-initiated logout to Keycloak; post-logout routing should not depend on org-fetch side effects.

---

## 3. Known Gaps / Risks

1. **Realm-per-tenant still aspirational**  
   - ADR-333 marks phases as “implemented”, but production still runs a single realm.  
   - No realm bootstrap migration (`V37__organization_realm_mapping.sql`) exists in repo; org catalog still keyed via legacy groups.

2. **SDK defaults leak legacy tenants**  
   - `packages/ameide_sdk_ts/src/platform/organization-client.ts` and graph service helpers continue to default to `DEFAULT_TENANT_ID`.  
   - Downstream scripts/tests assume atlas tenant; needs explicit tenant plumbing to avoid cross-tenant leakage during multi-tenant roll-out.

3. **Access resolution degraded vs empty**  
   - Any access timeout/error must surface explicitly as `TIMEOUT`/`ERROR` and must never be interpreted as “no orgs” for routing decisions (see `backlog/597-login-onboarding-primitives.md`).

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
| P0 | Update SDK defaults (TS/Go) to remove `DEFAULT_TENANT_ID` and derive tenant context from issuer-first routing | SDK team | Sprint +1 |
| P1 | Add regression tests ensuring TIMEOUT/ERROR never route into onboarding | Platform QA | Sprint +2 |
| P1 | Document “drop all data” bootstrap procedure (Keycloak realms + platform seed) | Ops | Sprint +2 |
| P2 | Guest federation roadmap: confirm membership storage uses platform id, add mapping table if necessary | Platform Auth | Sprint +3 |
| P2 | Fix Jest moduleNameMapper for onboarding path to keep unit suite green | DX Tooling | Sprint +1 |

---

## 5. Reference Diagnostics

- **Logs to watch**  
  - `[auth] applyOrganizationContext - organizations fetched` (ensure `count` > 0)  
  - Middleware routing logs to verify access-state handling (TIMEOUT/ERROR ≠ EMPTY) and that onboarding endpoints are not invoked during login.

- **Manual sanity checks**  
  1. Seeded user logs in → lands in app shell without hitting registrations endpoints (see `backlog/597-login-onboarding-primitives.md`).  
  2. Access TIMEOUT/ERROR shows a blocker/retry view and does not route into onboarding.  
  3. `/logout` then login returns to the shell and routes via access state, not session flags.

---

## 6. Glossary

- **Platform user id**: Primary key generated by the platform service; stored in session as `session.user.id`.  
- **Keycloak subject (`kcSub`)**: Stable per realm identifier; stored alongside the platform id for cross-realm lookups.  
- **Tenant slug**: Human-readable tenant identifier stored in tenant metadata (`metadata.slug`), used for future realm naming.

---

_Last updated: 2025-11-02 (post admin66 regression fix)._
