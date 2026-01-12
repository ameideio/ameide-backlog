# 597b: Login + Onboarding — UI Routes, Guards, and Wireframes

**Status:** Proposed  
**Audience:** `www_ameide_platform` owners; Identity/Onboarding owners; UX/design  
**Scope:** UI behavior only: route expectations, guard rules, and wireframes that prevent “login/access resolution” from being misinterpreted as “needs onboarding”.

Companion to `backlog/597-login-onboarding-primitives.md` (which defines the primitive boundaries and typed access states). This doc translates those primitives into **concrete UI route behavior** for `services/www_ameide_platform`.

---

## Goals

- Always land post-auth in a **stable signed-in shell route** that renders a clear access state (`OK | EMPTY | TIMEOUT | ERROR` + typed issues) and never “auto-onboards”.
- Make `/onboarding` **explicit and intentional**, not a fallback for unknown/degraded access.
- Provide predictable “return-to” behavior for deep links, invitations, and onboarding completion.
- Keep UI routes and guards consistent with Next.js route groups (`(public)`, `(app)`, `(onboarding)`).

## Non-goals

- Defining backend contracts/primitives (see 597).
- Designing the full product IA beyond the login/access/onboarding boundaries.

---

## Canonical route map (UI contract)

## Shell boundaries (what counts as “site wireframe”)

Auth/join/onboarding screens should **not** render inside the authenticated “site/app shell” (header/nav + app chrome). They are separate shells with their own minimal layouts.

### Shell map (ASCII)

```
Public shell (no session SSR, no app chrome)
  /login
  /register
  /accept?token=...   (invitation/join flow; may link to /login)

Signed-in access hub (still not the app chrome)
  /                (portal/access state; chooses next)

Onboarding shell (explicit provisioning UI)
  /onboarding

App shell (authenticated product chrome)
  /org/:orgId/...
```

### Codebase anchors (current)

- Public shell: `services/www_ameide_platform/app/(public)/layout.tsx`
- Root layout (global Providers + optional session prefetch): `services/www_ameide_platform/app/layout.tsx`
- App shell + auth gate: `services/www_ameide_platform/app/(app)/layout.tsx` (redirects unauthenticated → `/login`)
- Portal/access hub route: `services/www_ameide_platform/app/page.tsx` + `services/www_ameide_platform/app/(app)/_components/PlatformPortalClient.tsx`
- Login UI: `services/www_ameide_platform/app/(public)/login/page.tsx` + `services/www_ameide_platform/app/(public)/login/KeycloakRedirect.tsx`
- Register entry (server-rendered HTML POST to Auth.js): `services/www_ameide_platform/app/(auth)/register/route.ts`
- Logout route (clears session cookies; optional IdP logout): `services/www_ameide_platform/app/(auth)/logout/route.ts`
- Header sign-out entry: `services/www_ameide_platform/features/header/components/HeaderUserMenu.tsx`
- Onboarding shell: `services/www_ameide_platform/app/(onboarding)/layout.tsx` + `services/www_ameide_platform/app/(onboarding)/onboarding/page.tsx` (server guard) + `services/www_ameide_platform/app/(onboarding)/onboarding/OnboardingClientPage.tsx` (client UI)
- Invitation acceptance (public/join shell): `services/www_ameide_platform/app/accept/page.tsx`

#### Addendum (2025-12-30): Local cluster gRPC base URL must be functional

Even though this is a UI doc, several of the UI states here (notably `TIMEOUT/ERROR` vs `EMPTY` and the onboarding bootstrap failure banner) become impossible to validate if the server-side SDK can’t reach the platform APIs.

Observed in local k3d:
- If `www-ameide-platform` is not running (or is crashlooping), Envoy shows `upstream request timeout` for `/login`, `/accept`, and `/api/auth/*` even while unit tests are green.
- We also saw gRPC `UNIMPLEMENTED` when probing `envoy-grpc:9000` **from outside the cluster** (e.g. via port-forward). In practice this can be a protocol/authority mismatch and does **not** mean `envoy-grpc` is broken for in-cluster server-to-server calls.

Policy:
- Keep server-to-server gRPC pointed at `AMEIDE_GRPC_BASE_URL=http://envoy-grpc:9000` (per `backlog/589-rpc-transport-determinism.md`).
- Do **not** switch the Next.js server to `platform:8082` as a “workaround”; fix the determinism/sync issue instead.

Verification (recommended):
- `GET /api/health/grpc` (server-side gRPC reachability check; returns 200/503).
- In-cluster probe (example): `kubectl -n ameide-local exec <www-pod> -- node -e '...'` calling the SDK against `http://envoy-grpc:9000`.

Code anchors:
- Base URL resolution: `services/www_ameide_platform/lib/sdk/server-client.ts`
- Health endpoint: `services/www_ameide_platform/app/api/health/grpc/route.ts`

### Public (no session SSR, no app shell)

- `GET /login`
  - Purpose: begin SSO (auto-POST to Auth.js sign-in + optional IdP hint).
  - Inputs:
    - `callbackUrl` (untrusted; sanitized to same-origin relative path; never `/login`).
    - `realm` (optional; used only as a UI hint selector; never treated as authority).
  - Functional invariants:
    - Initiates the login chain **exactly once per navigation**, even under React 18 dev Strict Mode.
    - Allows manual retry without double-submitting concurrent attempts.
  - Output: renders a provider-neutral “Signing you in…” state with manual retry.

#### Addendum (2025-12-30): UI copy must not expose provider/technical details

Preserve the implementation detail (we use Keycloak today), but avoid putting it in end-user copy.

UI copy guidelines:
- Avoid “Keycloak” / “IdP” branding in the login/register redirect screens; use provider-neutral language (“Signing you in”, “Continue”).
- Avoid surfacing low-level details (CSRF/PKCE/state, “invalid_grant”, upstream timeouts, markdown/layout artifacts) to end users.
- Put technical details in logs and attach a correlation/request id where possible; the UI can offer a generic support CTA.

#### Addendum (2025-12-30): Portal UI must be user-facing by default

The access hub `/` should not default to a “debug dashboard” that prints internal states (`tenantId`, `organizationAccessStatus`, `TenantNotProvisioned`, etc.) to end users. Those details belong in logs/telemetry and (optionally) a developer-only view.

Recommended:
- Hide diagnostics by default; allow explicit opt-in via `?debug=1`.

Code anchors:
- Portal route: `services/www_ameide_platform/app/page.tsx`
- Portal view-state logic: `services/www_ameide_platform/app/(app)/_components/PlatformPortalClient.tsx`

- `GET /register`
  - Purpose: begin SSO with “signup/register” hint.
  - Input: `callbackUrl` (untrusted; sanitized; never `/login` or `/register`).
  - Output: same as `/login` but with sign-up intent.

### Signed-in shell (must never auto-provision)

- `GET /` (Portal / Access hub)
  - Purpose: post-auth landing; render access state; route into app or user actions.
  - Required behavior:
    - `OK` + “has real org” → client redirect to org home (`/org/:slug`).
    - `EMPTY` → show “no access” actions (invites/support), not onboarding by default.
    - `TIMEOUT/ERROR` → show degraded access + retry; never onboarding.
    - Typed tenant issues → show blocker view in-shell; never onboarding.

#### Addendum (2025-12-31): Seeded persona must not “fall into onboarding” on API failures

If a seeded persona cannot resolve tenant/org due to a transient platform outage (or protocol mismatch), the portal must render a degraded/error view and offer retry/sign-out. It must **not** switch into “needs onboarding” unless we are confident the user is truly unassigned.

Code anchors:
- Tenant resolution helper: `services/www_ameide_platform/lib/auth/auto-tenant.ts`
- E2E contract: `services/www_ameide_platform/features/onboarding/__tests__/e2e/seeded-admin-no-onboarding.spec.ts`

### Onboarding (explicit provisioning lane)

- `GET /onboarding`
  - Purpose: guided provisioning/recovery workflow.
  - Guard:
    - If unauthenticated → redirect to `/login?callbackUrl=/onboarding`.
    - If access is degraded (`TIMEOUT/ERROR`) → block entry and route back to `/` (with reason).
    - If user is not in an onboarding-allowed lane → route back to `/` (with explanation).

### Invitation acceptance (join flow)

- `GET /accept?token=…`
  - Purpose: validate invitation and (if needed) send user through login, then accept invite.
  - Guard:
    - If unauthenticated → link to `/login?callbackUrl=/accept?token=…` (+ optional `realm` hint).

---

## Guard rules (what the UI must do)

### Rule 0 — Public/join routes must not be behind the app shell redirect

Any route that needs to render unauthenticated UI (notably `/accept`) must **not** live under a layout that does `redirect('/login')` when unauthenticated.

Concretely, if `/accept` stays under `services/www_ameide_platform/app/(app)/layout.tsx`, the invitation details wireframe will never show to signed-out users because the layout will redirect immediately.

### Rule 0b — Login initiation uses Auth.js provider endpoint + CSRF

UI must initiate OAuth via the provider-specific Auth.js endpoint and CSRF flow:

- `POST /api/auth/signin/keycloak` with `csrfToken` and `callbackUrl` (and optional `kc_idp_hint`)
- `POST /api/auth/signin?provider=keycloak` is **not supported in this stack** and must not be used (it can bounce/loop via `GET /api/auth/signin` + `pages.signIn`).

Codebase anchors (current):
- `/login` prepares `postPath=/api/auth/signin/keycloak`: `services/www_ameide_platform/app/(public)/login/page.tsx`
- `/login` clears transients → fetches CSRF → POSTs to provider sign-in: `services/www_ameide_platform/app/(public)/login/KeycloakRedirect.tsx`
- `/register` uses the same provider endpoint: `services/www_ameide_platform/app/(auth)/register/route.ts`

### Rule A — Never interpret “unknown/degraded” as “empty”

If access resolution returns `TIMEOUT` or `ERROR`, the UI must:
- stay in the signed-in shell (`/`) or show a dedicated degraded view,
- offer **Retry** and **Sign out**,
- avoid showing onboarding CTAs and avoid calling provisioning endpoints.

This includes “tenant not resolved due to transport failure” cases: if the platform can’t be reached, treat it as degraded access (retry), not “needs onboarding”.

### Rule B — `/onboarding` is not a “fallback page”

The only legitimate entry to onboarding is one of:
- Explicit user intent (“Create a new workspace”) from the portal, **and** lane is onboarding-allowed, or
- Post-registration flow that is explicitly routed to onboarding (`/register?callbackUrl=/onboarding`), **and** lane is onboarding-allowed.

### Rule C — “No orgs” is not the same as “needs onboarding”

`EMPTY` must be treated as “no organization access found” (invite/join flow), not “tenant bootstrap required”.

If product wants to support “create first org” when `EMPTY`, the UI must require an explicit capability/permission flag (e.g., `canCreateFirstOrg` or `onboardingLane == ASSIGNED_NO_ORG`) before offering “Create workspace”.

### Rule D — Invitations always win

When the user lands on `/accept?token=…`, the UI should:
- never send them to onboarding as a next step,
- complete the invitation acceptance and then route to the new org home.

---

## Wireframes (text-only)

### 0) Shell overview (what should/shouldn’t be drawn as “site wireframe”)

```
┌───────────────────────────────┐      ┌───────────────────────────────────┐
│ Public shell                  │      │ App shell (site wireframe)        │
│ (no header/nav/app chrome)    │      │ (header/nav/app chrome)           │
│  /login /register /accept     │  vs  │  /org/...                         │
└───────────────────────────────┘      └───────────────────────────────────┘
                 │
                 ▼
       ┌──────────────────┐
       │ Access hub “/”    │  (signed-in, but still not org/app chrome)
       │ OK/EMPTY/ERROR…   │
       └──────────────────┘
                 │
                 ▼
       ┌──────────────────┐
       │ Onboarding “/…”   │  (explicit; minimal shell)
       └──────────────────┘
```

### 1) `/login`

```
┌──────────────────────────────────────────────┐
│ Signing you in                               │
│ We’re redirecting you to finish signing in.  │
│                                              │
│ [spinner] Preparing secure sign-in…          │
│                                              │
│ (error) Unable to start sign-in…             │
│                                              │
│ [ Continue ]  (disabled while…)              │
│ Having trouble? support@ameide.io            │
└──────────────────────────────────────────────┘
```

### 2) `/` (Portal / Access hub)

**2a) Access = OK**
```
Signed in as alice@corp.com
Access: OK
Tenant: <id>
[ Redirecting to your workspace… ] (auto)
(Fallback link) Open /org/acme
```

**2b) Access = EMPTY (no org access)**
```
No organization access found
- If you have an invite: [ Accept invitation ]
- If you need access: [ Contact admin/support ]
- If you’re creating the first workspace: [ Create a workspace ] (only if allowed)
[ Sign out ]
```

**2c) Access = TIMEOUT/ERROR (degraded)**
```
Unable to verify access right now
This can happen when the platform API is slow/unavailable.
[ Retry access ]  [ Sign out ]
(Optional) Show correlation id / timestamp / last error code
```

**2d) Tenant issue blocker (issuer missing / tenant not resolved)**
```
Your tenant isn’t ready yet
We can’t route you to a workspace because tenant routing is incomplete.
[ Retry ]  [ Contact support ]  [ Sign out ]
```

### 3) `/onboarding`

**3a) Not signed in**
```
Sign in required to continue onboarding.
[ Continue to sign in ] (→ /login?callbackUrl=/onboarding)
```

**3b) Signed in but degraded access**
```
Unable to verify access; onboarding is paused
Retry access resolution before provisioning anything.
[ Back to portal ] [ Retry access ]
```

**3c) Signed in, onboarding allowed**
```
Set up your workspace
Step 1: Review tenant
Step 2: Organization details
Step 3: Finish
```

### 4) `/accept?token=…` (invitation/join flow)

**4a) Signed out (should render invitation details + sign-in CTA)**
```
You’re invited to join: ACME
Role: Viewer
Expires: 2025-01-01

[ Sign in to accept ]  (→ /login?callbackUrl=/accept?token=…)
```

**4b) Signed in (can accept directly)**
```
You’re invited to join: ACME
[ Accept invitation ]  [ Decline ]
After accept → /org/acme
```

---

## Best-practice lock-downs (UI-adjacent guardrails)

These are implementation constraints that directly affect route/shell safety and should remain true as UI evolves.

### A) `clear-transients` must be non-abusable

Constraints:
- **POST-only**, same-origin checked.
- Clears **only** transient auth cookies (pkce/state/csrf/callback-url), never session cookies.
- Optional hardening: basic rate limit + structured logging to spot loops/abuse.

Codebase anchor: `services/www_ameide_platform/app/api/auth/clear-transients/route.ts`

### B) Keep Auth.js redirect safety as a defense-in-depth layer

Constraints:
- Treat UI `callbackUrl` as untrusted input (client sanitization is necessary but not sufficient).
- Ensure Auth.js `redirect` callback enforces same-origin (or tighter), so server-side redirects can’t be abused.

Codebase anchor: `services/www_ameide_platform/app/(auth)/auth.ts`

### C) Proxy config must be consistent with browser-visible origin

Constraints:
- Set canonical URL (`AUTH_URL`) consistently across environments.
- Trust forwarded host/proto when behind ingress/telepresence so “expected origin” matches browser reality.

Codebase anchors:
- Canonical URL usage for origin expectations: `services/www_ameide_platform/app/api/auth/clear-transients/route.ts`
- Auth.js host trust: `services/www_ameide_platform/app/(auth)/auth.shared.ts`
- Environment docs: `services/www_ameide_platform/README.md`

### D) Prevent duplicate login attempts in React Strict Mode

Constraint:
- `/login` must avoid double-firing the `clear-transients → csrf → signin` chain in React 18 dev Strict Mode to prevent “invalid_grant / code already used” storms.

Codebase anchor: `services/www_ameide_platform/app/(public)/login/KeycloakRedirect.tsx`

Validation (recommended):
- Add an E2E assertion that a single `/login` visit triggers at most one of each:
  - `POST /api/auth/clear-transients`
  - `GET /api/auth/csrf`
  - `POST /api/auth/signin/keycloak`

## What to address in current `www_ameide_platform` implementation (UI-only)

1) **Do not offer onboarding from generic “no access”**
   - Current behavior: portal treats `EMPTY` as “no access” (invite/support) and does not offer onboarding by default.
   - Desired: only show onboarding entry when the UI has an explicit onboarding-allowed signal (e.g., `session.error === 'TenantNotProvisioned'` or a dedicated lane/capability flag); `EMPTY` alone is insufficient.

2) **Guard `/onboarding` for unauthenticated sessions**
   - Current behavior: `/onboarding` does not redirect when `useSession()` is unauthenticated, which can render unauthenticated bootstrap UI that will fail on submit.
   - Desired: unauthenticated users see a sign-in gate (or client redirect) instead of a provisioning form.

3) **Keep `/accept` public/join (never behind app shell)**
   - Current behavior: `/accept` renders from `services/www_ameide_platform/app/accept/page.tsx` (outside the app shell redirect).
   - Desired: middleware must allow `/accept` and its validation call (`GET /api/v1/invitations/validate?token=…`) without requiring a session cookie; invite acceptance itself remains authenticated.

4) **Treat `realm` as a hint, not a requirement**
   - Current behavior: `/accept` appends a `realm` query param to `/login`, but `/login` only uses a static `REALM_HINT_MAP`.
   - Desired: either (a) remove the `realm` param from UI flows until there is a dynamic mapping strategy, or (b) explicitly define a dynamic tenant/realm hint lookup UI pattern (e.g., server-provided mapping) so invitations can reliably pre-select the correct IdP.

5) **Keep “degraded access” UX consistent across `/` and `/onboarding`**
   - Current behavior: both surfaces implement “timeout is not empty”, but they should share the same wording + CTA set: `Retry`, `Sign out`, and optional diagnostics.

---

## Acceptance criteria (UI)

- Post-login always lands on `/` and renders exactly one of: `OK`, `EMPTY`, `TIMEOUT/ERROR`, or “typed blocker”, with no onboarding side effects.
- `/onboarding` cannot be entered when unauthenticated or when access is degraded.
- “Create workspace”/onboarding CTAs are only shown when onboarding is explicitly allowed; `EMPTY` alone is insufficient.
- Invitation acceptance flow (`/accept`) never routes into onboarding.
