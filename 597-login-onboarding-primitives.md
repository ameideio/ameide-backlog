# 597: Login/Logout vs Onboarding — Primitive Boundaries (Single Realm + IdP-per-Tenant Baseline)

**Status:** Proposed  
**Audience:** Platform + Identity + Tenancy owners; www_ameide_platform owners; test-infra  
**Scope:** Define the **minimal primitive set** required for login/logout, and the **additional primitives** required for onboarding (provisioning + recovery), assuming the **baseline deployment model** is **one platform Keycloak realm** with **IdP-per-tenant** (brokered IdPs in that realm).

---

## Executive Summary

Login/logout is an **access-resolution concern** (authn + session + “what can this principal access?”). Onboarding is a **provisioning concern** (create/repair the tenant routing mapping, create org + membership, activate tenant). Mixing these concerns causes a fragile failure mode: degraded access resolution (or identity/key mismatches) gets misinterpreted as “no orgs”, which incorrectly routes users into onboarding and can hard-fail on tenant readiness invariants (e.g., missing `idp_alias → tenant_id` mapping).

This backlog defines:

1) which primitives are needed to support **login/logout** without onboarding, and  
2) which additional primitives are needed to support **onboarding** safely and idempotently.

---

## Use With

- `backlog/428-onboarding.md` (current onboarding + identity provisioning)
- `backlog/322-rbac.md` (roles, enforcement, membership semantics)
- `backlog/300-400/333-realms.md` (optional future: realm-per-tenant variant)
- `backlog/430-unified-test-infrastructure.md` (integration runner contract)
- `backlog/596-onboarding-capability.md` (onboarding hardening + test pack target)

---

## GitOps implications (ameide-gitops)

This backlog is mostly **runtime primitive + contract** work, but it has a few concrete **cluster/GitOps** implications:

- **Recommended GitOps model (baseline):** GitOps owns **platform Keycloak infrastructure** + **exactly one platform realm** import (`KeycloakRealmImport/ameide-realm`). Tenant SSO configuration is expressed as **N Identity Provider configs in one realm**, written by runtime onboarding/support (not GitOps overlays).
- **No tenant-specific GitOps overlays/apps:** GitOps must not manage per-tenant Keycloak IdPs or tenant secrets via PR flow; tenants are runtime state.
- **Secrets posture:** Git stores only `ExternalSecret` references; Keycloak client secrets (and any tenant IdP secrets) are never committed.
- **Smokes as enforcement:** `platform-auth-smoke` is the GitOps gate. It must fail fast if the realm import breaks *or* if the realm is missing the minimum knobs needed for login routing safety:
  - OIDC discovery serves for the realm
  - `kc_idp_hint` plumbing exists (Identity Provider Redirector is enabled in the Browser flow)
  - `idp_alias` is emitted in tokens (mapper from session note `identity_provider` → claim `idp_alias`)

### GitOps implementation progress (current)

Implemented in `ameide-gitops` (local + dev parity):

- [x] **Realm config baseline:** one platform realm import; `kc_idp_hint` redirector and `idp_alias` mapper are present and validated by `platform-auth-smoke`.
- [x] **No `tenantId` claim:** legacy `tenant` client scope removed from desired realm import and purged from Keycloak (no compatibility shims).
- [x] **Seed correctness (597 identity):** dev/local seeds create canonical platform user ids (opaque ids) and external identity links `(issuer, subject) → user_id`, so login never depends on provisioning.
- [x] **Tenant routing (idp-alias-first):** dev/local seeds write `idp_alias → tenant_id` into platform DB, so access resolution has an authoritative routing key in a single-realm brokered-IdP model.

---

## Deployment model baseline vs variants

This backlog previously mixed two models. The **baseline** (and the one aligned with current GitOps + UI work) is:

### Baseline: Single realm + IdP-per-tenant (brokered IdPs)

- There is **one platform Keycloak realm** (e.g. `ameide`).
- Each tenant’s directory/SSO is configured as a **brokered Identity Provider** (IdP) inside that realm.
- The IdP alias (`idp_alias`) is emitted in tokens and is the authoritative tenant-routing key:
  - `principal.idp_alias` → `tenant_id` via `TenantRoutingView`.
- `kc_idp_hint` is a *hint* to choose an IdP at login initiation; it is not tenant authority on its own.
- `principal.issuer` is still the OIDC issuer URL, but in this model it is typically **constant** (the platform realm issuer) and must **not** be used for tenant routing.

### Optional future variant: Realm-per-tenant

Realm-per-tenant can still be pursued later, but it changes the routing and callback validation model (multi-issuer mix-up defenses, issuer-driven routing, realm provisioning). Treat any realm-per-tenant-specific guidance in this doc as **future/optional** unless explicitly called out.

---

## Definitions

### Login / Logout (authn + session + access resolution)

**Login**: authenticate principal via IdP, establish a session, and resolve access state for routing.  
**Logout**: terminate session locally and (optionally) propagate logout to IdP.

Login/logout must never require provisioning. If access cannot be resolved reliably, surface a typed “degraded access” state and allow retry — do not route into onboarding.

### Onboarding (provisioning + recovery)

Onboarding is a workflow that creates or repairs durable tenant/org state. In the baseline model, it ensures:
- a durable **tenant routing mapping** (`idp_alias → tenant_id`), and
- org + membership creation/repair.

It is only entered under explicit lane semantics (e.g., `UNASSIGNED` or explicit “create-first-org” intent with authorization).

### “App shell” vs “access success”

For this backlog, “land in the app shell” means:
- the IdP callback succeeds and a session is established, and
- the web app renders a stable signed-in route that shows an explicit access state view (`OK | EMPTY | TIMEOUT | ERROR` + typed issues), and
- it does **not** implicitly call onboarding/provisioning endpoints.

A user can “land in the shell” even when there are typed blockers (e.g., tenant readiness violations). In that case the shell renders a blocker view and prevents navigation to normal app routes until an explicit admin/support action (or corrected seed) resolves the issue.

**Implementation note (platform.ameide.io)**: the app shell entrypoint is `GET /` (portal). Do not introduce alternate “shell” escape hatches (e.g. `/app`) that bypass the portal’s explicit access state semantics.

---

## 1) Minimal primitives needed for login/logout

The goal is: **a seeded/admin (or any already-assigned) user can log in and land in the app shell without calling registrations endpoints** (either normal app UX when access resolves, or a typed blocker/degraded view when it does not).

### 1.1 Integration primitive: Identity Adapter (OIDC/Keycloak)

**Responsibilities**
- Pre-login IdP selection (optional `kc_idp_hint`) and auth redirect construction.
- Use OIDC discovery for the platform realm issuer and cache it.
- Authorization Code flow token exchange (and refresh, when applicable).
- User identity claims retrieval (OIDC issuer URL + subject + email).
- Optional: RP-initiated logout / front-channel logout hooks.

**Outputs**
- `Principal` = `{ issuer, subject, idp_alias?, email?, claims… }`
  - `issuer` is the OIDC issuer **URL** (do not treat email as an ID)
  - `idp_alias` identifies which brokered IdP was used (required for tenant routing in the baseline model)

#### Pre-login IdP selection (single realm + IdP-per-tenant baseline)

Because tenant routing is IdP-alias-driven, the system must (optionally) select the **expected IdP alias** *before* redirecting to Keycloak.

Minimum viable rule:
- derive the expected IdP alias from an explicit pre-login input (invitation link, tenant-picker UI, etc.)
- persist `expected_idp_alias` inside the login attempt state (server-side), and bind it to `state` (CSRF)

#### Callback validation (baseline)

The adapter must validate the callback using the login attempt state:
- verify `state` (and `nonce` if used)
- verify PKCE `code_verifier` when applicable
- verify the ID token (signature and standard claims: `iss`, `aud`, `exp`, `nonce`, etc.)
- if an `expected_idp_alias` was set, verify the resulting `idp_alias` claim matches it (hard fail if wrong)

#### Realm-per-tenant callback validation (optional future)

If realm-per-tenant is adopted later, add multi-issuer mix-up defenses (RFC 9207 `iss` validation, expected issuer binding, etc.). This is not baseline for the single-realm brokered-IdP model.

### 1.2 Domain primitive: Identity Catalog (canonical user)

**Responsibilities**
- Maintain canonical platform user ids (opaque IDs).
- Maintain external identity links: `(issuer, subject) → userId`.
- Optional guarded legacy bootstrap: “link-by-email” only under explicit policy.

**Key operation**
- `EnsurePlatformUser(principal)` → canonical `userId` + link `(issuer, subject)` idempotently.

**Guardrails for “link-by-email” (recommended defaults)**
- Disabled by default; enable only for controlled migrations / dev seeding recovery.
- Require `email_verified=true` (or equivalent) when available from the IdP.
- Never link across issuers; allow-list which issuers are allowed for link-by-email.
- Prefer admin/support-initiated linking; treat self-serve linking as high risk unless you have additional proof-of-control.
- Emit an audit fact/event for any link operation (old→new identity linkage + correlation id).
  - Recommended placement: do not implement link-by-email inside `EnsurePlatformUser`; keep it in admin/support tooling or a support-only command.

### 1.3 Domain primitive: Tenancy Catalog (read path for access)

**Responsibilities**
- Tenants, orgs, memberships (canonical ids).
- Tenant metadata: tenant routing (`idp_alias → tenant_id`) in the single-realm brokered-IdP baseline.
- Tenant aliases (legacy tenant id → canonical tenant id), when needed.

**Key operations**
- `ResolveTenantContext(principal)` → canonical tenant id using deterministic precedence (see below).
- `ListMemberships(userId, tenantId)` (canonical ids only).

**Tenant resolution precedence (single realm + IdP-per-tenant baseline)**

1) **Authority:** `principal.idp_alias` is authoritative (it comes from Keycloak broker session; do not accept user-controlled tenant hints).  
2) **Resolve by idp alias:** map `idp_alias` → canonical `tenant_id` via `TenantRoutingView`.  
3) **If not found:** return a typed tenant resolution issue (`TENANT_NOT_FOUND_FOR_IDP_ALIAS`) and block in-shell.

**Tenant hints (recommended posture)**
- Do not accept user-controlled tenant-id hints for post-login access resolution (avoid tenant enumeration side-channels).
- Tenant selection happens pre-login (host/invite/tenant-picker) and is bound to `state` via `expected_idp_alias`.

### 1.4 Projection primitive: UserAccessView (routing input)

**Responsibilities**
- Compute an explicit access state from canonical identifiers:
  - `status`: `OK | EMPTY | TIMEOUT | ERROR`
  - `userId` (canonical)
  - `tenantId` (canonical)
  - `memberships[]` (org + role)
  - typed issues split by ownership (avoid mixing “identity” with “tenant readiness”):
    - identity resolution issues (e.g., external identity link missing)
    - tenant resolution issues (e.g., issuer→tenant not found, alias ambiguous, tenant context mismatch)
    - tenant readiness issues (e.g., issuer mapping missing/mismatch)

**Invariant**
- `TIMEOUT/ERROR` must never be treated as `EMPTY`.

### 1.5 UISurface primitive: Session Gateway + Route Guards

**Responsibilities**
- Login callback establishes session and then queries `UserAccessView`.
- Route based on access state:
  - `OK` → app shell (normal UX *only if* there are no blocking issues)
  - `EMPTY` → “no access / request invite / contact admin”
  - `TIMEOUT/ERROR` → degraded-access banner with retry diagnostics
  - tenant resolution / tenant readiness issues (e.g. `TENANT_IDP_ALIAS_MISSING`) → show blocker in-shell; do not route into provisioning
- Logout clears session and invokes RP-initiated IdP logout (front-channel redirect) using OIDC discovery metadata (`end_session_endpoint`) when configured; avoid non-standard “token logout” flows unless explicitly required.
- Prefer standards-based RP-initiated logout via discovery (`end_session_endpoint`) and include `id_token_hint` when available; treat any legacy Keycloak logout query formats as migration-only fallbacks.

**Session security baseline (minimum)**
- Use Authorization Code flow; use PKCE for public/browser clients; avoid implicit flow.
- Require and verify `state` (CSRF) and `nonce` (ID token replay protection) when applicable.
- Keep tokens server-side; do not store access/refresh tokens in browser storage.
- Use secure session cookies (`Secure`, `HttpOnly`, appropriate `SameSite`) and rotate on login.
- Enforce idle + absolute session timeouts; invalidate sessions on logout.
- We intentionally do **not** use `keycloak-js` in the browser; the OIDC client is server-side and the browser holds only a session cookie.

### 1.6 Target primitive names (repo layout)

Naming convention: `<bounded-context>-<thing>`; avoid a bare `onboarding` primitive name. For this approach the bounded contexts are `identity` and `tenancy`.

- Integration: `identity-oidc-adapter`
- Domain: `identity`
- Domain: `tenancy`
- Projection: `tenancy-access` (exports `UserAccessView` + typed issues)
- Integration: `tenancy-idp-provisioner` (Keycloak Admin / brokered-IdP side effects)
- UISurface: `tenancy-session-gateway` (or keep inside `services/www_ameide_platform` until extraction is worthwhile)

---

## 2) Additional primitives needed for onboarding

Onboarding is the provisioning/recovery workflow that runs when (and only when) lane semantics permit it.

### 2.1 Process primitive: Onboarding Orchestrator (resumable, idempotent)

**Responsibilities**
- Determine lane: `UNASSIGNED | ASSIGNED_NO_ORG | HAS_ORG | SEEDED_PERSONA | DEGRADED_ACCESS`.
- Execute steps with idempotency keys and resumability:
  1. `EnsureTenantReady` (baseline: `idp_alias → tenant_id` routing present; brokered IdP exists in platform realm; may repair only if policy allows)
  2. `EnsureOrgAndMembership`
  3. `ActivateTenant`

### 2.2 Domain primitive: Tenancy (single writer for canonical state)

**Responsibilities**
- Own canonical tenant/org/membership state (single-writer).
- Provide idempotent domain commands used by onboarding (`create-or-get` semantics).
- Validate and persist tenant readiness transitions (pending → active) and tenant routing metadata (`idp_alias`, aliases when applicable).
- Avoid splitting “catalog” vs “provisioning” into separate writers for the same aggregates/tables.

### 2.3 Integration primitive: IdP Provisioner (Keycloak Admin)

**Responsibilities**
- Create/update brokered Identity Providers (IdPs) inside the **single platform realm** and validate required knobs (alias, discovery config, mappers).
- Write/update tenant routing metadata to the tenancy domain (`idp_alias → tenant_id`), only within allowed lanes/authority.

### 2.4 Projection primitives: OnboardingAttemptView + TenantReadinessView

**Responsibilities**
- Expose step timeline + typed blockers for UI/support.
- Make tenant readiness violations observable without stack traces.

### 2.5 UISurface primitive: Onboarding Portal

**Responsibilities**
- Collect org details and invoke orchestrator commands.
- Present blockers with retry/escalation actions (admin/support vs end user).

### 2.6 Optional Agent primitive: Onboarding Support

**Responsibilities**
- Admin-only repair assistant (alias mapping, IdP routing readiness repair) with audit trail.

### 2.7 Target primitive names (repo layout)

- Process: `tenancy-onboarding` (lane resolution + idempotent orchestration)
- Integration: `tenancy-idp-provisioner` (Keycloak Admin / brokered-IdP side effects)
- Projection: `tenancy-onboarding` (attempt timeline + typed blockers views)
- UISurface: `tenancy-onboarding-portal` (or keep inside `services/www_ameide_platform` until extraction is worthwhile)
- Agent (optional): `tenancy-onboarding-support` (admin-only repairs with audit trail)

---

## 3) Proto contracts (naming conventions + minimal surfaces)

This section proposes the **proto package layout** and the **minimum RPC surfaces** needed to implement the primitive boundaries above, following the repo convention used by e.g. `ameide_core_proto.sre.{core|process|integration}.v1`.

> Note: today, many tenancy/identity APIs exist under `ameide_core_proto.platform.v1` (e.g. `TenantService`, `OrganizationService`, `UserService`). This backlog defines the **target** package layout for the primitives we want (identity canonicalization, access view, onboarding process). Migration can be staged (thin wrappers / parallel services) but contracts should land in the target namespaces.

### 3.1 Package + file layout (target)

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

### 3.2 Identity core (canonical user + external identity link)

**Package:** `ameide_core_proto.identity.core.v1`  
**Goal:** make `(issuer, subject)` the stable key, where `issuer` is the OIDC issuer **URL**, and map it to a canonical platform user id.

**Key messages (outline)**
- `ExternalIdentityKey { string issuer; string subject; }`
- `ExternalIdentityLink { ExternalIdentityKey key; string user_id; string email; }`

**Services (outline)**
- `service IdentityCommandService`
  - `rpc EnsurePlatformUser(EnsurePlatformUserRequest) returns (EnsurePlatformUserResponse);`
  - `rpc LinkExternalIdentity(LinkExternalIdentityRequest) returns (LinkExternalIdentityResponse);` (admin/support only)
- `service IdentityQueryService`
  - `rpc GetUserByExternalIdentity(GetUserByExternalIdentityRequest) returns (GetUserByExternalIdentityResponse);`

**Requests/responses (outline)**
- `EnsurePlatformUserRequest { ameide_core_proto.common.v1.RequestContext context; ExternalIdentityKey external; string email; }` (email is an attribute, not an identifier; linking is by `(issuer,subject)` only)
- `EnsurePlatformUserResponse { string user_id; bool created; bool link_created; }`

### 3.3 Tenancy core (tenants/orgs/memberships + access view)

**Package:** `ameide_core_proto.tenancy.core.v1`  
**Goal:** represent readiness + access explicitly; avoid “unknown ⇒ empty”; make `idp_alias` the authoritative tenant-routing key (single realm brokered-IdP baseline).

**Key domain types (outline)**
- `Tenant { string tenant_id; string tenant_slug; string idp_alias; TenantStatus status; map<string,string> attributes; }` (`idp_alias` selects the brokered IdP used for this tenant)
- `Organization { string organization_id; string tenant_id; string slug; string display_name; }`
- `Membership { string membership_id; string tenant_id; string organization_id; string user_id; string role; }`
- `TenantAlias { string legacy_tenant_id; string canonical_tenant_id; string source; }`
- `TenantRoutingViewEntry { string idp_alias; string tenant_id; string tenant_slug; }` (unique by `idp_alias`)

**Access projection (outline)**
- `enum UserAccessStatus { USER_ACCESS_STATUS_UNSPECIFIED = 0; USER_ACCESS_STATUS_OK = 1; USER_ACCESS_STATUS_EMPTY = 2; USER_ACCESS_STATUS_TIMEOUT = 3; USER_ACCESS_STATUS_ERROR = 4; }`
- `enum IdentityResolutionIssueCode { IDENTITY_RESOLUTION_ISSUE_CODE_UNSPECIFIED = 0; IDENTITY_RESOLUTION_ISSUE_CODE_IDENTITY_LINK_MISSING = 1; IDENTITY_RESOLUTION_ISSUE_CODE_EMAIL_LINK_FORBIDDEN = 2; IDENTITY_RESOLUTION_ISSUE_CODE_EMAIL_LINK_AMBIGUOUS = 3; }`
- `enum TenantResolutionIssueCode { TENANT_RESOLUTION_ISSUE_CODE_UNSPECIFIED = 0; TENANT_RESOLUTION_ISSUE_CODE_TENANT_NOT_FOUND_FOR_IDP_ALIAS = 1; TENANT_RESOLUTION_ISSUE_CODE_TENANT_ALIAS_AMBIGUOUS = 2; TENANT_RESOLUTION_ISSUE_CODE_TENANT_CONTEXT_MISMATCH = 3; }`
- `enum TenantReadinessIssueCode { TENANT_READINESS_ISSUE_CODE_UNSPECIFIED = 0; TENANT_READINESS_ISSUE_CODE_TENANT_IDP_ALIAS_MISSING = 1; TENANT_READINESS_ISSUE_CODE_TENANT_IDP_ALIAS_MISMATCH = 2; }`
- `message IdentityResolutionIssue { IdentityResolutionIssueCode code = 1; string message = 2; map<string,string> details = 3; }`
- `message TenantResolutionIssue { TenantResolutionIssueCode code = 1; string message = 2; map<string,string> details = 3; }`
- `message TenantReadinessIssue { TenantReadinessIssueCode code = 1; string message = 2; map<string,string> details = 3; }`
- `message UserAccessView { UserAccessStatus status = 1; string user_id = 2; string tenant_id = 3; repeated Membership memberships = 4; repeated IdentityResolutionIssue identity_issues = 5; repeated TenantResolutionIssue tenant_resolution_issues = 6; repeated TenantReadinessIssue tenant_readiness_issues = 7; }`

**Services (outline)**
- `service TenancyAccessQueryService`
  - `rpc GetUserAccess(GetUserAccessRequest) returns (GetUserAccessResponse);`

**Requests/responses (outline)**
- `GetUserAccessRequest { ameide_core_proto.common.v1.RequestContext context; }`
- `GetUserAccessResponse { UserAccessView access = 1; }`

### 3.4 Tenancy onboarding process (orchestration evidence + commands)

**Package:** `ameide_core_proto.tenancy.process.v1`  
**Goal:** express lane semantics and onboarding outcomes without throwing 500s.

**Key enums/messages (outline)**
- `enum OnboardingLane { ONBOARDING_LANE_UNSPECIFIED = 0; UNASSIGNED = 1; ASSIGNED_NO_ORG = 2; HAS_ORG = 3; SEEDED_PERSONA = 4; DEGRADED_ACCESS = 5; }`
- `enum OnboardingStep { ONBOARDING_STEP_UNSPECIFIED = 0; PREFLIGHT = 1; TENANT_READY = 2; ORG_MEMBERSHIP = 3; ACTIVATION = 4; }`
- `message OnboardingAttemptView { string attempt_id; string tenant_id; string user_id; OnboardingLane lane; OnboardingStep step; repeated tenancy.core.v1.TenantReadinessIssue tenant_readiness_issues; repeated tenancy.core.v1.TenantResolutionIssue tenant_resolution_issues; }`

**Services (outline)**
- `service TenancyOnboardingCommandService`
  - `rpc StartAttempt(StartAttemptRequest) returns (StartAttemptResponse);`
  - `rpc SubmitOrgDetails(SubmitOrgDetailsRequest) returns (SubmitOrgDetailsResponse);`
  - `rpc RetryAttempt(RetryAttemptRequest) returns (RetryAttemptResponse);`
- `service TenancyOnboardingQueryService`
  - `rpc GetAttempt(GetAttemptRequest) returns (GetAttemptResponse);`

**Process facts (outline)**
- `message TenancyProcessFact { ... oneof fact { AttemptStarted ... TenantReadyValidated ... OrganizationCreated ... MembershipCreated ... AttemptBlocked ... AttemptCompleted ... } }`

### 3.5 IdP provisioner integration (Keycloak Admin)

**Package:** `ameide_core_proto.tenancy.integration.v1` (or `ameide_core_proto.identity.integration.v1`)  
**Goal:** explicit boundary for brokered-IdP side effects in a single realm.

**Service (outline)**
- `service IdpProvisionerService`
  - `rpc EnsureIdentityProvider(EnsureIdentityProviderRequest) returns (EnsureIdentityProviderResponse);`
  - `rpc ValidateIdentityProvider(ValidateIdentityProviderRequest) returns (ValidateIdentityProviderResponse);`

**Requests/responses (outline)**
- `EnsureIdentityProviderRequest { ameide_core_proto.common.v1.RequestContext context; string tenant_id; string idp_alias; }`
- `EnsureIdentityProviderResponse { string idp_alias; bool created; }`

### 3.6 How to scaffold primitives (repo commands)

Scaffolding is proto-driven and writes into `primitives/<kind>/<name>/…`. Run from the repo root.

**0) Create the proto files first (recommended)**

Create the proto files under the target packages from section 3.1, then scaffold from the *service* proto file that declares the RPCs.

Suggested “service proto” files to scaffold from:

- `identity` (domain): `packages/ameide_core_proto/src/ameide_core_proto/identity/core/v1/identity_command_service.proto`
- `tenancy` (domain): `packages/ameide_core_proto/src/ameide_core_proto/tenancy/core/v1/tenancy_command_service.proto`
- `tenancy-access` (projection/query): `packages/ameide_core_proto/src/ameide_core_proto/tenancy/core/v1/tenancy_access_query_service.proto`
- `tenancy-onboarding` (process): `packages/ameide_core_proto/src/ameide_core_proto/tenancy/process/v1/tenancy_onboarding_command_service.proto`
- `tenancy-idp-provisioner` (integration): `packages/ameide_core_proto/src/ameide_core_proto/tenancy/integration/v1/idp_provisioner_service.proto`

**1) Dry-run scaffolds (preview)**

```bash
ameide primitive scaffold --kind domain --name identity --proto-path packages/ameide_core_proto/src/ameide_core_proto/identity/core/v1/identity_command_service.proto --dry-run
```

**2) Scaffold the primitives (idempotent by default)**

Go primitives (domain/process/projection/integration) require `--proto-path`:

```bash
ameide primitive scaffold --kind integration --name identity-oidc-adapter --proto-path packages/ameide_core_proto/src/ameide_core_proto/identity/integration/v1/identity_oidc_adapter_service.proto --include-test-harness
ameide primitive scaffold --kind domain --name identity --proto-path packages/ameide_core_proto/src/ameide_core_proto/identity/core/v1/identity_command_service.proto --include-test-harness
ameide primitive scaffold --kind domain --name tenancy --proto-path packages/ameide_core_proto/src/ameide_core_proto/tenancy/core/v1/tenancy_command_service.proto --include-test-harness
ameide primitive scaffold --kind projection --name tenancy-access --proto-path packages/ameide_core_proto/src/ameide_core_proto/tenancy/core/v1/tenancy_access_query_service.proto --include-test-harness
ameide primitive scaffold --kind process --name tenancy-onboarding --proto-path packages/ameide_core_proto/src/ameide_core_proto/tenancy/process/v1/tenancy_onboarding_command_service.proto --include-test-harness
ameide primitive scaffold --kind integration --name tenancy-idp-provisioner --proto-path packages/ameide_core_proto/src/ameide_core_proto/tenancy/integration/v1/idp_provisioner_service.proto --include-test-harness
```

Optional: add GitOps scaffolding (writes under `gitops/ameide-gitops/...` in this repo’s submodule):

```bash
ameide primitive scaffold --kind domain --name tenancy --proto-path packages/ameide_core_proto/src/ameide_core_proto/tenancy/core/v1/tenancy_command_service.proto --include-gitops --include-test-harness
```

UISurface / Agent primitives do not require `--proto-path`:

```bash
ameide primitive scaffold --kind uisurface --name tenancy-onboarding-portal
ameide primitive scaffold --kind agent --name tenancy-onboarding-support --lang python
```

**3) Add cross-primitive tests in the repo-root pack**

Place orchestration/E2E tests in `tests/integration/onboarding/` (repo-root pack), not inside any one primitive.

---

## 4) Non-negotiable invariants (single realm + IdP-per-tenant baseline)

1. **There is exactly one platform realm** for the web app.
2. **Every tenant has exactly one `idp_alias` routing key**, and it is unique.
3. **Login does not provision.** Provisioning requires explicit lane/intent and authority.
4. **Canonical identifiers**: memberships are keyed by canonical `userId` and canonical `tenantId` (aliases are resolution aids, not the persistent truth).
5. **Tenant routing is idp-alias-driven.** `principal.idp_alias` → `tenant_id` via `TenantRoutingView` is authoritative; user-controlled tenant hints are validated inputs, not authority.

### 4.1 Operational caveat (single realm + many IdPs)

Single-realm + many brokered IdPs is a strong operational baseline, but it still has cost:
- Plan and test for your expected IdP count (admin API throughput, broker login latency, caching).
- Treat `idp_alias` mapping completeness as an SLO: if a tenant is expected to work, `TenantRoutingView` must have an `idp_alias → tenant_id` entry; otherwise expect an in-shell blocker state.
- Add reconciliation (periodic in non-dev) to detect drift between Keycloak IdP configs and `TenantRoutingView`.
- Keep canonical URL/host configuration stable so Auth.js and Keycloak agree on browser-visible origins.

### 4.2 GitOps implication (no tenant-specific GitOps requirement)

Single realm + idp-alias-driven routing requires **tenant-specific data** (`idp_alias → tenant_id`), but it does **not** require “tenant-specific GitOps” (one GitOps overlay/app per tenant).

Two supported patterns:
- **Runtime-managed tenants (default):** onboarding/admin flows provision IdPs and write tenant routing into the tenancy catalog (DB). No PRs to GitOps are required to create a tenant.
- **GitOps-managed tenant registry (optional):** GitOps stores a declarative tenant registry (tenant id, idp alias) that a controller reconciles into Keycloak + DB. Use when tenants are pre-provisioned or need infra-style change control.

`AMEIDE_TENANT_ID` (see `backlog/583-gitops-tenant-env-alignment.md`) remains a **system/bootstrap context** env var; it does not replace idp-alias-based tenant resolution for user traffic and should not be interpreted as a per-tenant GitOps requirement.

---

## 5) Acceptance Criteria

### Login/logout

1. After IdP login, the app always lands in a stable signed-in shell route (normal UX or typed blocker/degraded view) without calling registrations endpoints.
2. When the user has canonical identity + membership + an idp-alias-routable tenant, `UserAccessView` resolves to `OK` and normal app routes are reachable (no onboarding).
3. If access resolution fails (`TIMEOUT/ERROR`), UI shows degraded access with retry; no onboarding.
4. If tenant readiness is violated (e.g., `TENANT_IDP_ALIAS_MISSING`), UI shows typed blocker; no onboarding.
5. Seeded persona usability requires ready seed data (tenant routing + identity links) and never relies on login-time provisioning.

### Onboarding

1. `UNASSIGNED` lane provisions tenant routing (`idp_alias`) + first org + membership idempotently.
2. `ASSIGNED_NO_ORG` lane can create first org/membership without tenant bootstrap; permissions enforced.
3. Replays of onboarding commands converge without duplicates.

---

## 6) Test plan (repo-root integration packs)

### 6.0 Pack governance (recommended)

Repo-root packs should be curated, not a dumping ground. Add a lightweight manifest to each pack (example: `tests/integration/onboarding/pack.yaml`) containing:
- owners/team
- which ADR/backlogs it asserts (e.g., 597 + 430)
- invariants enforced (“login does not provision”, “TIMEOUT ≠ EMPTY”, “idp_alias routing authority”)
- supported modes (`repo|cluster`)

### 6.1 Login/logout (access resolution) tests

Goal: prove **login does not provision** and routing is driven by `UserAccessView` (with “unknown ≠ empty”).

**Core cases**
- Seeded/admin persona logs in → `UserAccessView.status=OK` → app shell; no onboarding endpoints invoked.
- External identity link missing → `UserAccessView.identity_issues` contains `IDENTITY_LINK_MISSING` → blocker view; support-only repair links the identity (no login-time linking).
- Org/membership query timeout → `UserAccessView.status=TIMEOUT` → degraded-access UI with retry; no onboarding.
- Tenant readiness violated (`idp_alias` mapping missing/mismatch) → `UserAccessView.tenant_readiness_issues` contains typed code → blocker UI; no onboarding.
- Logout clears session and (when configured) performs IdP logout.

**Where these tests live**
- Primitive tests (fast, deterministic):
  - `primitives/domain/identity/tests`: idempotency and support-only identity linking guardrails (including any email-based recovery flow, if implemented).
  - `primitives/projection/tenancy-access/tests`: `OK|EMPTY|TIMEOUT|ERROR` correctness and blocker typing.
  - `primitives/integration/identity-oidc-adapter/tests`: error mapping (invalid code/refresh, IdP unavailable).
- Cross-primitive E2E (routing invariants): include in `tests/integration/onboarding/` as “login lane” scenarios until/unless a dedicated pack is created.

### 6.2 Onboarding (provisioning + recovery) tests

Goal: prove provisioning is **lane-scoped**, **idempotent**, and maintains **idp_alias-driven routing** without 500s.

**Core cases**
- `UNASSIGNED` lane: provisions IdP config + tenant routing + first org + membership + activation (idempotent under retries).
- `ASSIGNED_NO_ORG` lane: creates first org + membership without tenant bootstrap; permissions enforced.
- Idempotency: repeated `StartAttempt` / `SubmitOrgDetails` converge to exactly one org + one membership.
- Failure typing: Keycloak IdP provision fails → attempt blocked with typed blocker; UI shows actionable state.
- Canonicalization: legacy tenant id resolves via alias mapping; principal mismatch is repaired or blocked with explicit code (never “0 orgs” by accident).

**Where these tests live**
- Primitive tests:
  - `primitives/process/tenancy-onboarding/tests`: replay/resume, lane decisions, dedupe keys.
  - `primitives/domain/tenancy/tests`: create-or-get org/membership and readiness validator.
  - `primitives/integration/tenancy-idp-provisioner/tests`: IdP create/validate behavior and error mapping.
- Cross-primitive E2E: `tests/integration/onboarding/` runs the end-to-end suite under `INTEGRATION_MODE=repo|cluster` using the 430 runner contract.

---

## 7) Detailed implementation checklist (phased)

The checklist below is ordered to deliver value early (login stability) while incrementally moving toward the target primitive boundaries.

### Phase 0 — Baseline + guardrails (today’s failure chain)

- [ ] **Lock vendor-aligned invariants (document + enforce in code paths)**:
  - idp-alias-driven routing authority: `principal.idp_alias` is authoritative; do not accept user-controlled tenant hints for access resolution
  - login establishes session + access state only; it must never “fall into” provisioning/onboarding
  - `TIMEOUT/ERROR` are first-class and must never be treated as `EMPTY`
  - BFF/server-session stance: browser holds only a session cookie; do not use `keycloak-js` in-browser
- [ ] Reproduce the current failure chain end-to-end (seeded persona → login → org discovery → onboarding redirect → realm-missing failure) and record:
  - principal fields: `issuer`, `subject`, `idp_alias`, `email`
  - expected IdP selection input (invite/tenant-picker)
  - membership lookup key(s) used vs what exists in the DB
  - where “unknown became empty” (timeout/error folded into `[]`)
- [ ] Confirm and document the authoritative tenant-routing rule: `principal.idp_alias` → canonical `tenant_id` via `TenantRoutingView`, else emit `TENANT_RESOLUTION_ISSUE_CODE_TENANT_NOT_FOUND_FOR_IDP_ALIAS` and block in-shell.
- [ ] **Pre-login IdP selection (recommended)**: define one supported selector (invite/tenant-picker), persist `expected_idp_alias` server-side bound to `state` for the login attempt.
- [ ] **Callback validation**: validate `state` + ID token (`iss`,`aud`,`exp`, `nonce` when used) and validate `idp_alias` matches `expected_idp_alias` (hard fail if wrong).
- [ ] **Remove login-path footguns**:
  - disallow caller-controlled email-linking policy in login path (`EnsurePlatformUser` links only by `(issuer,subject)`; email is an attribute)
  - remove/ignore user-controlled tenant-id hints in access resolution APIs (avoid tenant enumeration side-channels)
- [ ] Make dev/local seeds deterministic and “ready”: tenant routing present (`idp_alias → tenant_id`) + external identity links present (login never provisions).
- [ ] Add repo-root integration pack governance manifest for onboarding (`tests/integration/onboarding/pack.yaml`) so the pack is curated, not a dumping ground.
- [ ] Introduce a single feature flag for routing cutover (example): `ACCESS_VIEW_ROUTING_ENABLED` (default off until tests pass).

### Phase 1 — Login must not route into onboarding (UI gate hardening)

- [ ] Replace “hasRealOrganization ⇒ /onboarding” logic with “render shell + access state”:
  - shell always renders after successful IdP callback
  - normal routes are enabled only when `UserAccessView.status=OK` and there are no blocking issues
  - `EMPTY`, `TIMEOUT`, `ERROR`, and readiness/resolution issues render explicit in-shell views
- [ ] Ensure `TIMEOUT/ERROR` are preserved as first-class states (never folded into `EMPTY`).
- [ ] Add “no provisioning on login” guardrails:
  - remove/avoid any implicit calls to registrations/onboarding endpoints from login callbacks
  - add request tracing/metrics so it’s visible if registrations endpoints are hit during login flows
- [ ] Add minimal UI acceptance tests (or route/middleware unit tests) asserting:
  - login does not redirect to `/onboarding` for already-assigned users
  - `TIMEOUT/ERROR` show degraded view (not onboarding)
  - readiness issues show blocker view (not onboarding)

### Phase 2 — Contracts: proto surfaces + domain schema (identity + tenancy)

- [ ] Add proto packages per section 3.1:
  - `ameide_core_proto.identity.core.v1`
  - `ameide_core_proto.tenancy.core.v1`
  - `ameide_core_proto.tenancy.process.v1`
  - `ameide_core_proto.tenancy.integration.v1`
- [ ] Add proto messages for:
  - external identity key/link (`issuer`,`subject` → `user_id`)
  - `UserAccessView` with split issue lists (`identity_issues`, `tenant_resolution_issues`, `tenant_readiness_issues`)
  - `TenantRoutingViewEntry` (`idp_alias → tenant_id`)
- [ ] Add DB schema/migrations to support the contracts (minimum viable):
  - `external_identities(issuer, subject, user_id, email, created_at, ...)` with unique `(issuer,subject)`
  - `tenant_routing(idp_alias, tenant_id, tenant_slug, updated_at, ...)` with unique `idp_alias`
  - (optional) `tenant_aliases(legacy_tenant_id, canonical_tenant_id, source, ...)` if aliasing is required
- [ ] Define timeouts/error mapping policy for access queries so `TIMEOUT` is testable and deterministic (repo-mode).

### Phase 3 — Scaffold primitives (repo layout)

- [ ] Create the “service proto” files referenced in section 3.6.
- [ ] Scaffold primitives (dry-run first, then apply):
  - [ ] `integration/identity-oidc-adapter`
  - [ ] `domain/identity`
  - [ ] `domain/tenancy`
  - [ ] `projection/tenancy-access`
  - [ ] `process/tenancy-onboarding`
  - [ ] `integration/tenancy-idp-provisioner`
- [ ] Wire build/test harnesses for each primitive (per scaffolder output) and ensure `go test ./...` passes in those modules.

### Phase 4 — Implement login/logout primitives (no provisioning)

- [ ] Implement `identity.EnsurePlatformUser(principal)`:
  - [ ] lookup by `(issuer,subject)`
  - [ ] if not found, create new user + link external identity
  - [ ] do not link by email in this operation (email is an attribute, not an identifier)
- [ ] Implement support-only identity recovery (never invoked during login):
  - [ ] link an external identity to an existing user (admin/support authorized)
  - [ ] if an email-based recovery path exists, keep it support-only, disabled by default, guarded (`email_verified`, issuer allow-list, ambiguity checks), and audited
- [ ] Implement idp-alias-driven tenant resolution:
  - [ ] use `principal.idp_alias` as the routing key
  - [ ] resolve `tenant_id` via `TenantRoutingView` (authoritative)
  - [ ] do not accept user-controlled tenant-id hints for access resolution
  - [ ] emit typed `TenantResolutionIssue` (e.g., `TENANT_NOT_FOUND_FOR_IDP_ALIAS`) when routing fails
- [ ] Implement `tenancy-access.GetUserAccess`:
  - [ ] returns `OK` with memberships when resolvable
  - [ ] returns `EMPTY` only when memberships are definitively empty for the canonical user+tenant
  - [ ] returns `TIMEOUT/ERROR` when queries fail (and includes typed issues)
  - [ ] emits `TenantReadinessIssue` when tenant idp alias mapping is missing/mismatched
- [ ] Integrate UI shell routing with `GetUserAccess` (feature-flagged):
  - [ ] no onboarding endpoints invoked during login/session establishment
  - [ ] blocker views include clear next actions (retry vs contact admin/support)

### Phase 5 — Implement onboarding primitives (explicit provisioning lanes)

- [ ] Implement lane evaluation in `tenancy-onboarding` using `UserAccessView` + explicit intent:
  - [ ] `UNASSIGNED` → allow provisioning
  - [ ] `ASSIGNED_NO_ORG` → allow create-first-org only (no tenant bootstrap)
  - [ ] `HAS_ORG` → no-op / return lane result
  - [ ] `DEGRADED_ACCESS` → block with retry (no provisioning)
- [ ] Implement idempotent step execution + attempt tracking:
  - [ ] attempt id + idempotency keys per step
  - [ ] replay-safe “create or get” semantics for org/membership
  - [ ] evidence facts emitted for each step transition
- [ ] Implement `tenancy-idp-provisioner` integration:
  - [ ] `EnsureIdentityProvider` (idempotent)
  - [ ] `ValidateIdentityProvider` (detect mismatch)
  - [ ] explicit error mapping to typed issues (no 500s)
- [ ] Implement authorization rules for `ASSIGNED_NO_ORG` (must be explicit; pick one):
  - [ ] tenant-admin claim
  - [ ] “first org creator” rule
  - [ ] support/admin only

### Phase 6 — Seeds + repair (make the seeded persona story deterministic)

- [x] Dev/local seeding must satisfy (no login-time provisioning):
  - [x] every seeded tenant has `idp_alias`
  - [x] tenant routing has `idp_alias → tenant_id`
  - [x] seeded personas exist in Keycloak and in platform DB
  - [x] seeded personas have external identity links `(issuer,subject) → user_id`
  - [x] seeded memberships reference canonical `user_id`
  - [x] Seed sources are versioned:
    - Keycloak + platform dev data values: `gitops/ameide-gitops/sources/values/env/local/platform/platform-dev-data.yaml`
    - Playwright persona secrets sync: `gitops/ameide-gitops/sources/values/env/local/platform/platform-playwright-int-tests-secret.yaml`
    - Vault bootstrap (where seeded passwords originate): `gitops/ameide-gitops/sources/values/_shared/foundation/foundation-vault-bootstrap.yaml`
- [ ] Provide support-only repair commands (for drift/migrations; never on login):
  - [ ] upsert/repair tenant routing (`TenantRoutingView`)
  - [ ] link external identity to existing user
  - [ ] all repairs produce an audit fact with old→new values

### Phase 7 — Tests (repo-root packs + primitive tests)

- [ ] Primitive tests:
  - [ ] identity: EnsurePlatformUser idempotency + support-only recovery guardrails
  - [ ] tenancy-access: `OK|EMPTY|TIMEOUT|ERROR` + issue typing + idp-alias-driven routing
  - [ ] onboarding: lane selection + idempotent steps + error typing
  - [ ] idp provisioner: create/validate + error mapping
- [ ] Repo-root integration pack `tests/integration/onboarding/`:
  - [ ] Login lane scenarios:
    - [ ] seeded admin lands in shell without registrations endpoints
    - [ ] `TIMEOUT` renders degraded view and allows retry
    - [ ] `TENANT_IDP_ALIAS_MISSING` renders blocker view and does not provision
  - [ ] Onboarding lane scenarios:
    - [ ] `UNASSIGNED` provisions idp routing + org + membership idempotently
    - [ ] `ASSIGNED_NO_ORG` creates first org/membership only
    - [ ] retries converge without duplicates
- [ ] Validate integration runner contract (430):
  - [ ] `node tools/integration-runner/bin/integration-runner.js -- --list` shows the pack
  - [ ] `node tools/integration-runner/check-packs.js` passes

### Phase 8 — Cutover + rollout

- [ ] Deploy new access query and enable `ACCESS_VIEW_ROUTING_ENABLED` progressively.
- [ ] Backfill (one-time) data needed for tenant routing (`idp_alias`) and identity links.
- [ ] Add dashboards/alerts for:
  - percent `TIMEOUT/ERROR` in `GetUserAccess`
  - counts of typed issues (idp alias missing/mismatch, tenant mismatch, identity link missing)
  - onboarding attempts per lane and failure codes
