# 597: Login/Logout vs Onboarding — Primitive Boundaries (Single‑Realm Keycloak + IdP‑per‑Tenant)

**Status:** Proposed  
**Audience:** Platform + Identity + Tenancy owners; `www_ameide_platform` owners; test‑infra  
**Scope:** Define the **minimal primitive set** required for login/logout, and the **additional primitives** required for onboarding (provisioning + recovery), using **one Keycloak realm (single issuer)** with **one brokered external IdP per tenant** as the recommended baseline.

---

## Executive Summary

Login/logout is an **access‑resolution concern** (authn + session + “what can this principal access?”). Onboarding is a **provisioning concern** (configure tenant auth routing + create org + membership + activate tenant). Mixing these concerns causes the exact failure chain we saw: degraded access resolution (or identity mismatches) gets misread as “no orgs”, which incorrectly routes users into onboarding and then fails on readiness invariants.

This backlog defines:

1) which primitives are needed to support **login/logout** without onboarding, and  
2) which additional primitives are needed to support **onboarding** safely and idempotently.

---

## Use With / Impacts

Recommended alignment targets:

- `backlog/431-auth-session-v2.md` (tenant binding, session/header shape, middleware expectations)
- `backlog/427-platform-login-stability.md` (login must not trigger onboarding; TIMEOUT≠EMPTY)
- `backlog/428-onboarding.md` (replace realm‑per‑tenant and `tenantId`‑claim assumptions with IdP‑binding baseline)
- `backlog/300-400/323-keycloak-realm-roles-v2.md` + `backlog/426-keycloak-config-map.md` (drop `tenantId` token claim as the tenancy signal; add `idp_alias` claim)
- `backlog/582-local-dev-seeding.md` (seed identity links + tenant IdP bindings; login never provisions)
- `backlog/583-gitops-tenant-env-alignment.md` (keep `AMEIDE_TENANT_ID` as bootstrap/system context only)

Reference design (optional future):

- `backlog/300-400/333-realms.md` (realm‑per‑tenant is a stronger isolation model, but not required to achieve “tenant SSO with their directory”)

---

## Definitions

### Login / Logout (authn + session + access resolution)

**Login**: authenticate principal via IdP, establish a session, and resolve an access state for routing.  
**Logout**: terminate session locally and (optionally) propagate logout to the IdP.

**Invariant:** Login/logout must never require provisioning. If access cannot be resolved reliably, surface a typed “degraded access” state and allow retry — do not route into onboarding.

### Onboarding (provisioning + recovery)

Onboarding is a workflow that creates or repairs durable tenant/org state, and ensures tenant auth routing invariants (tenant has a brokered IdP binding and a valid Keycloak IdP configuration). It is entered only under explicit lane semantics and authorization.

### “App shell” vs “access success”

For this backlog, “land in the app shell” means:
- the IdP callback succeeds and a session is established, and
- the web app renders a stable signed‑in route that shows an explicit access state view (`OK | EMPTY | TIMEOUT | ERROR` + typed issues), and
- it does **not** implicitly call onboarding/provisioning endpoints.

---

## Recommended Keycloak Model (Baseline)

**Goal:** “each tenant can SSO with their directory” without multi‑issuer complexity.

### 1) One Keycloak realm (single issuer)

- The app integrates with **one** OIDC issuer (the platform realm).
- Tenants authenticate through **identity brokering**: one external IdP (OIDC/SAML) configured in Keycloak per tenant.

### 2) One brokered IdP per tenant (stable `idp_alias`)

- Tenancy Catalog stores `tenant_id → idp_alias` (stable).
- Keycloak stores an Identity Provider configuration keyed by the same `idp_alias`.

### 3) Tenant binding authority = login attempt binding

Tenant selection happens **before** redirecting to Keycloak (subdomain/invite/tenant picker). The system:

1. resolves `{ expected_tenant_id, expected_idp_alias }` server‑side
2. persists it server‑side as the login attempt state (bound to `state`)
3. redirects to Keycloak with `kc_idp_hint=expected_idp_alias`
4. on callback, validates:
   - `state` (and `nonce` if used)
   - ID token standard claims
   - brokered IdP alias used for login (see below) matches `expected_idp_alias`

**Unknown tenant selection**

If the system cannot determine a trusted `{ expected_tenant_id, expected_idp_alias }` (no subdomain/invite/tenant‑picker selection), it must **not** start OIDC login. Render a tenant picker or return a typed “tenant required” error.

**Brokered IdP alias observation**

Keycloak stores the broker alias used for login as a user‑session note (`identity_provider`). Map it into tokens as a claim (recommended: `idp_alias`) using a protocol mapper, then validate it on callback.

**Non‑forgeable requirement:** `principal.idp_alias` must come from Keycloak (e.g. a claim mapped from the `identity_provider` user‑session note) and be validated server‑side. Never accept `idp_alias` from client‑controlled inputs (query params, headers, cookies, local storage).

---

## 1) Minimal primitives needed for login/logout

**Goal:** a seeded/admin (or any already‑assigned) user can log in and land in the app shell **without calling registrations endpoints**.

### 1.1 Integration primitive: `identity-oidc-adapter` (OIDC/Keycloak)

Responsibilities:
- Build the auth redirect (Authorization Code flow; PKCE where applicable)
- Pass `kc_idp_hint` (tenant broker routing)
- Validate callback and tokens (see Security baseline)
- Expose a `Principal` object: `{ issuer, subject, email?, idp_alias?, claims… }`

### 1.2 Domain primitive: `identity` (Identity Catalog)

Responsibilities:
- Canonical platform user ids (opaque IDs)
- External identity links: `(issuer, subject) → userId`

Key operation:
- `EnsurePlatformUser(principal)` links by `(issuer,subject)` only; email is an attribute, not an identifier.

**Policy (recommended)**
- Do not perform link‑by‑email during normal login.
- If link‑by‑email exists at all, keep it admin/support only, audited, and disabled by default.
- Do not expose caller‑supplied “linking policy” knobs on normal login APIs; any elevated linking must be a distinct admin/support command with authorization + audit.

### 1.3 Domain primitive: `tenancy` (Tenancy Catalog)

Responsibilities:
- Tenants, orgs, memberships (canonical ids)
- Tenant auth routing metadata: `tenant_id → idp_alias`
- Tenant aliases (legacy tenant id → canonical tenant id), when needed

Key operation:
- `ResolveTenantContext(loginAttempt, principal)` uses the state‑bound `{ expected_tenant_id, expected_idp_alias }` and validates `principal.idp_alias`.

### 1.4 Projection primitive: `tenancy-access` (UserAccessView)

Responsibilities:
- Compute explicit access state:
  - `status`: `OK | EMPTY | TIMEOUT | ERROR`
  - `userId` (canonical)
  - `tenantId` (canonical)
  - `memberships[]`
  - typed issues split by ownership:
    - identity resolution issues
    - tenant binding/resolution issues
    - tenant readiness issues (IdP binding/config)

Invariant:
- `TIMEOUT/ERROR` must never be treated as `EMPTY`.

### 1.5 UISurface primitive: `tenancy-session-gateway` + route guards

Responsibilities:
- After callback: establish session, query `UserAccessView`, render shell.
- Route based on access state:
  - `OK` → normal UX
  - `EMPTY` → “no access / request invite”
  - `TIMEOUT/ERROR` → degraded‑access UI + retry
  - readiness/binding issues → blocker view (in‑shell), no onboarding
- Logout clears session and uses RP‑initiated logout via discovery metadata (`end_session_endpoint`) when configured.

---

## 2) Additional primitives needed for onboarding

Onboarding runs only when lane semantics permit it.

### 2.1 Process primitive: `tenancy-onboarding` (orchestrator)

Responsibilities:
- Determine lane: `UNASSIGNED | ASSIGNED_NO_ORG | HAS_ORG | SEEDED_PERSONA | DEGRADED_ACCESS`
- Execute idempotent steps:
  1. `EnsureTenantIdpBinding` (validate → provision/repair or block)
  2. `EnsureOrgAndMembership`
  3. `ActivateTenant`

### 2.2 Integration primitive: `tenancy-idp-provisioner` (Keycloak Admin)

Responsibilities:
- Provision/validate a tenant’s brokered IdP configuration (OIDC/SAML) in the platform realm
- Persist/repair `tenant_id → idp_alias` only under explicit authority (onboarding/support), never during normal login

### 2.3 Projections + UISurface

- Projection: onboarding attempt timeline + typed blockers (for UI/support)
- UISurface: onboarding portal (UNASSIGNED + Create‑first‑org flows), driven only by projections and typed states

---

## 3) Proto contracts (naming conventions + minimal surfaces)

Target proto layout:

```text
packages/ameide_core_proto/src/ameide_core_proto/
  identity/
    core/v1/...
    integration/v1/...
  tenancy/
    core/v1/...
    process/v1/...
    integration/v1/...
```

### 3.1 Identity core (`ameide_core_proto.identity.core.v1`)

Outline:
- `ExternalIdentityKey { string issuer; string subject; }`
- `EnsurePlatformUser` links by `(issuer,subject)` only
- `LinkExternalIdentity` is admin/support only (if needed)

### 3.2 Tenancy core (`ameide_core_proto.tenancy.core.v1`)

Outline:
- `Tenant { string tenant_id; string tenant_slug; string idp_alias; TenantStatus status; ... }`
- `UserAccessView` with:
  - `status`: `OK|EMPTY|TIMEOUT|ERROR`
  - split issue lists:
    - `identity_issues[]`
    - `tenant_resolution_issues[]` (e.g., `TENANT_LOGIN_BINDING_MISMATCH`)
    - `tenant_readiness_issues[]` (e.g., `TENANT_IDP_ALIAS_MISSING`, `TENANT_IDP_CONFIG_INVALID`)

### 3.3 Tenancy process (`ameide_core_proto.tenancy.process.v1`)

Outline:
- `StartAttempt`, `SubmitOrgDetails`, `ResumeAttempt`, `Retry`
- `OnboardingAttemptView` for timeline + blockers

### 3.4 Tenancy integration (`ameide_core_proto.tenancy.integration.v1`)

Outline:
- `EnsureTenantIdp` (idempotent create/validate)
- typed error mapping for “not found / invalid / disabled / timeout”

---

## 4) Non‑negotiable invariants (recommended baseline)

1. **One Keycloak realm (one issuer) for the platform.**
2. **Tenant readiness includes IdP binding** (`tenant.idp_alias` present and valid in Keycloak).
3. **Tenant binding is state‑bound**: `{ expected_tenant_id, expected_idp_alias }` bound to `state` is the authority.
4. **Login does not provision.**
5. **Canonical identifiers**: memberships reference canonical `userId` and canonical `tenantId` (aliases are resolution aids).
6. **Unknown ≠ empty**: `TIMEOUT/ERROR` are first‑class states and never treated as `EMPTY`.

---

## 5) GitOps implication (No tenant‑specific GitOps requirement)

The broker model requires **tenant‑specific data** (`tenant_id → idp_alias`), but it does **not** require “tenant‑specific GitOps” (one overlay/app per tenant).

**Recommended default:** runtime‑managed tenants.
- onboarding/admin flows provision IdP configs and write `tenant_id → idp_alias` into the Tenancy Catalog (DB)
- no PRs to GitOps are required to create a tenant

`AMEIDE_TENANT_ID` (see `backlog/583-gitops-tenant-env-alignment.md`) remains a **system/bootstrap context** env var; it is not the tenant binding mechanism for user traffic.

---

## 6) Tests (repo‑root integration packs)

All cross‑primitive orchestration tests live in repo‑root packs: `tests/integration/<pack>/...` (not in a “capabilities” folder).

### 6.1 Login/logout tests

Goal: prove **login does not provision** and routing is driven by `UserAccessView`.

Core cases:
- seeded/admin logs in → `UserAccessView.status=OK` → shell; **no registrations endpoints invoked**
- identity link missing → blocker view; support‑only repair links identity (no login‑time email linking)
- org/membership timeout → `TIMEOUT` → degraded view + retry; no onboarding
- tenant readiness violated (IdP alias missing/invalid) → typed blocker; no onboarding
- logout clears session and (when configured) propagates IdP logout

### 6.2 Onboarding tests

Goal: prove provisioning is **lane‑scoped**, **idempotent**, and enforces **IdP binding** without 500s.

Core cases:
- `UNASSIGNED` provisions IdP binding + first org + membership + activation
- `ASSIGNED_NO_ORG` creates first org/membership without tenant bootstrap; permissions enforced
- repeated commands converge without duplicates
- IdP provision fails → attempt blocked with typed blocker; UI shows actionable state

### 6.3 Pack governance (recommended)

Add a lightweight manifest to each pack (example: `tests/integration/onboarding/pack.yaml`) containing:
- owners/team
- which backlogs it asserts (e.g., 597 + 430 runner contract)
- invariants enforced (“login does not provision”, “TIMEOUT ≠ EMPTY”, “tenant binding authority”)
- supported modes (`repo|cluster`)

---

## 7) Scaffolding primitives (instructions)

Create the proto files under the target packages (Section 3), then scaffold from the *service proto* that declares the RPCs.

Suggested “service proto” files to scaffold from:

- `identity` (domain): `packages/ameide_core_proto/src/ameide_core_proto/identity/core/v1/identity_command_service.proto`
- `tenancy` (domain): `packages/ameide_core_proto/src/ameide_core_proto/tenancy/core/v1/tenancy_command_service.proto`
- `tenancy-access` (projection/query): `packages/ameide_core_proto/src/ameide_core_proto/tenancy/core/v1/tenancy_access_query_service.proto`
- `tenancy-onboarding` (process): `packages/ameide_core_proto/src/ameide_core_proto/tenancy/process/v1/tenancy_onboarding_command_service.proto`
- `tenancy-idp-provisioner` (integration): `packages/ameide_core_proto/src/ameide_core_proto/tenancy/integration/v1/idp_provisioner_service.proto`

Example scaffold commands (adjust to your CLI):

```bash
ameide primitive scaffold --kind integration --name identity-oidc-adapter --proto-path packages/ameide_core_proto/src/ameide_core_proto/identity/integration/v1/identity_oidc_adapter_service.proto --include-test-harness
ameide primitive scaffold --kind domain --name identity --proto-path packages/ameide_core_proto/src/ameide_core_proto/identity/core/v1/identity_command_service.proto --include-test-harness
ameide primitive scaffold --kind domain --name tenancy --proto-path packages/ameide_core_proto/src/ameide_core_proto/tenancy/core/v1/tenancy_command_service.proto --include-test-harness
ameide primitive scaffold --kind projection --name tenancy-access --proto-path packages/ameide_core_proto/src/ameide_core_proto/tenancy/core/v1/tenancy_access_query_service.proto --include-test-harness
ameide primitive scaffold --kind process --name tenancy-onboarding --proto-path packages/ameide_core_proto/src/ameide_core_proto/tenancy/process/v1/tenancy_onboarding_command_service.proto --include-test-harness
ameide primitive scaffold --kind integration --name tenancy-idp-provisioner --proto-path packages/ameide_core_proto/src/ameide_core_proto/tenancy/integration/v1/idp_provisioner_service.proto --include-test-harness
```

---

## 8) Detailed implementation checklist (doc‑only; for the implementing agent)

### Phase 0 — Lock invariants + reproduce the current failure chain

- [ ] Bind tenant selection to login attempt state (`{expected_tenant_id, expected_idp_alias}`) and validate on callback.
- [ ] Ensure Keycloak emits `idp_alias` in the token/session context (mapper from `identity_provider` note).
- [ ] Verify access resolution surfaces `OK|EMPTY|TIMEOUT|ERROR` without folding TIMEOUT/ERROR into EMPTY.
- [ ] Seeded personas: ensure seeds include `tenant_id → idp_alias` and external identity links; login never provisions.

### Phase 1 — UI routing hardening

- [ ] Replace “empty org list ⇒ onboarding” with “render shell + access state”.
- [ ] Add guardrails/metrics proving registrations endpoints are never invoked during login flows.

### Phase 2 — Contracts + schema

- [ ] Add proto packages per Section 3.
- [ ] Add DB schema for `external_identities` and `tenant_auth_routing (tenant_id → idp_alias)`.

### Phase 3 — Onboarding orchestration

- [ ] Implement lane decisions + idempotent steps; add attempt timeline projection.
- [ ] Implement IdP provisioner with typed error mapping; no normal‑login side effects.

### Phase 4 — Integration pack gates

- [ ] Create `tests/integration/onboarding/` pack enforcing 597 invariants in `repo|cluster` modes.
