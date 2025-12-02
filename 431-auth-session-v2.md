# 431 – Auth & Session Architecture v2 (Middleware-First)

**Status:** Adopted  
**Intent:** Define the canonical pattern for authentication and session usage in the Ameide Platform after `safeAuth` removal. This backlog consolidates decisions from:
- [329-auth.md](./329-auth.md)
- [331-tenant-resolution.md](./331-tenant-resolution.md)
- [330-dynamic-tenant-resolution.md](./330-dynamic-tenant-resolution.md)
- [427-platform-login-stability.md](./427-platform-login-stability.md)
- [428-onboarding.md](./428-onboarding.md)

and replaces the older “server components call `safeAuth()`” model with a **middleware-first** architecture.

---

## 1. Motivation

We previously relied on `safeAuth()` for server-side session access in layouts, server components, and some utilities:

- RootLayout called `safeAuth()`, causing Redis and org-fetch work on every SSR request.
- Server utilities (e.g. `org-scope`, feature resolvers) could also trigger `safeAuth()`.
- This made `/login` and other routes fragile (connection resets, long compilation times).

Backlog [427](./427-platform-login-stability.md) implemented a safer pattern:

- Edge middleware (proxy) validates the session once and injects trusted headers.
- Public routes (like `/login`) no longer perform server-side auth.
- `safeAuth` has been removed from the codebase.

This backlog captures that **v2 auth/session architecture** so future work stays aligned.

---

## 2. High-Level Architecture

### 2.1 Flow Overview

```text
Keycloak (tenantId user attribute)
  ↓
JWT Token (tenantId + roles claims via protocol mappers)
  ↓
NextAuth Session (session.user.tenantId, roles, org context)
  ↓
Proxy Auth Gate (`proxy.ts`, using an edge-compatible NextAuth config, running in the Node runtime)
  ↓
Trusted headers for server components:
  - x-pathname
  - x-tenant-id
  - x-user-id
  - x-user-kc-sub
  - x-org-home
  ↓
App Routes
  - Public: (public)/login, (public)/register, healthz
  - Authenticated: (app)/..., (onboarding)/...
  - API routes: use getSession() only when strictly needed
```

### 2.2 Runtimes and Responsibilities

- **Proxy Auth Gate (`services/www_ameide_platform/proxy.ts`):**
  - Runs in the **Node runtime**, using an **edge-compatible** NextAuth configuration (JWT sessions, no DB adapter) to validate cookies and decode the session token.
  - Enforces optimistic access control for protected routes (redirects unauthenticated users to `/login` with `callbackUrl` and applies onboarding rules like `hasRealOrganization` → `/onboarding`).
  - Injects `x-pathname`, `x-tenant-id`, `x-user-id`, `x-user-kc-sub`, `x-org-home` for downstream server components and layouts.
  - Does **not** perform Redis or org-fetch operations; all heavy/authoritative checks happen in Node-only API routes and backend services.

- **Root Layout (`app/layout.tsx`):**
  - Performs **no server-side auth or session lookup**.
  - Hydrates global providers with `session = null`.
  - Delegates auth state to:
    - Middleware-injected headers for server components.
    - Client-side session hooks in the browser.

- **Public Layout / Routes (`app/(public)/...`):**
  - `PublicLayout` performs no auth() / `getSession()` calls.
  - `/login` is a client-only page that:
    - Sanitizes `callbackUrl` and `realm` params.
    - Prepares a secure redirect to `/api/auth/signin/keycloak`.
  - `/register` is implemented as an HTML-only route that forwards into Auth.js/Keycloak.
  - `/healthz` is a standalone route (`dynamic = 'force-static'`), excluded from middleware.

- **Authenticated App Routes (`app/(app)/...`, `app/(onboarding)/...`):**
  - Assume middleware has already enforced auth and onboarding rules.
  - Use headers:
    - `x-pathname` to resolve navigation descriptors.
    - `x-tenant-id`, `x-user-id`, `x-user-kc-sub`, `x-org-home` for org/tenant context.
  - Must **not** call `auth()` / `getSession()` in layouts for gating.

> **Security note:** The middleware-injected headers are a **convenience mechanism** for layouts and server components, not a security boundary. All sensitive operations (e.g. tenant/org membership, role checks, data reads/writes) must still be enforced in Node-only API routes and backend services using authenticated session/JWT context.

---

## 3. Allowed Session Access Patterns

### 3.1 Preferred: Middleware Headers

Server components and layouts should preferentially use headers injected by middleware:

- `x-tenant-id` – primary tenant identifier.
- `x-user-id` / `x-user-kc-sub` – user identity, realm-aware.
- `x-org-home` – resolved home org route for the current user.
- `x-pathname` – canonical request pathname for navigation.

Examples:

- `app/(app)/layout.tsx`:
  - Reads `x-pathname`, `x-tenant-id` and calls `getNavigationDescriptor(pathname, tenantId)`.
- `app/(app)/org/[orgId]/layout.tsx`:
  - Reads `x-tenant-id`, `x-user-id`, `x-user-kc-sub` to resolve the organization via gRPC.

**Rule:** If the information you need is already in headers, do not call `getSession()`.

### 3.2 Direct `getSession()` (Node runtime only)

Direct calls to `getSession()` from `app/(auth)/auth.ts` are allowed in **narrow, explicit contexts**:

- **Navigation access control**:
  - `features/navigation/server/access.ts` uses `getSession()` to:
    - Read roles (`admin`, `contributor`, `viewer`, etc.).
    - Compute `UserAccess` booleans for navigation.
- **Debug and test endpoints** (guarded by env flags):
  - `/api/auth/debug` – surfaces session and token wiring details.
  - `/api/test/userinfo` – demonstrates use of access tokens against Keycloak.

**Constraints for `getSession()` usage:**

- Node runtime only (`runtime = 'nodejs'`).
- Never in:
  - Root layout.
  - Public routes.
  - Health checks.
  - General-purpose server components that can leverage headers instead.

### 3.3 Disallowed: New Auth Helpers

The following patterns are **not allowed** going forward:

- Reintroducing `safeAuth()` or any wrapper around `auth()` / `getSession()` that:
  - Hides JWT/Redis exceptions.
  - Encourages implicit session access from arbitrary server components.
- Calling `auth()` / `getSession()` from:
  - `app/layout.tsx`.
  - `(public)` layout and routes.
  - `(app)` layout for gating basic navigation.

---

## 4. safeAuth Deprecation (Final)

### 4.1 Historical Role

`safeAuth()` was introduced as a Node-only helper that:

- Wrapped NextAuth `auth()` / `getSession()`.
- Swallowed JWT/JWE decode/verify errors and returned `null`.
- Prevented crashes when stale/invalid cookies or secret mismatches occurred.

It was used by:

- RootLayout for server-side session resolution.
- Some server utilities (`org-scope`, feature helpers).
- Debug and test routes.

### 4.2 Current Status

As of the work captured in [427](./427-platform-login-stability.md):

- `safeAuth` has been **fully removed** from the codebase.
- `app/(auth)/safe-auth.ts` no longer exists.
- All prior call sites have been migrated to either:
  - Middleware header consumption, or
  - Explicit `getSession()` calls in Node-only, tightly scoped modules.

Any remaining references to `safeAuth` in backlog documents are:

- Annotated as **historical**.
- Accompanied by notes pointing to this v2 architecture.

**Rule:** Do not reintroduce `safeAuth` or similar “catch-all” auth helpers.

---

## 5. Patterns for New Work

### 5.1 New Public Routes (e.g. marketing, auth entry points)

When adding routes under `app/(public)/`:

- Use `PublicLayout` or another non-auth layout.
- Do **not** import `auth()`, `getSession()`, or any auth helper.
- If a route needs to kick off an OAuth flow:
  - Prepare safe redirect URLs client-side (like `/login`).
  - Call into `/api/auth/signin` or provider-specific Auth.js endpoints.

Public routes should never block on Redis or org-fetch logic.

### 5.2 New Authenticated Pages

For new routes under `app/(app)/` or `app/(onboarding)/`:

- Assume middleware has gated access.
- Use:
  - `x-tenant-id` to parameterize SDK clients and RPC calls.
  - `x-user-id` / `x-user-kc-sub` for user- or org-specific data fetches.
- Avoid calling `getSession()` unless:
  - You need detailed session fields that are not represented in headers (and those fields cannot reasonably be added to middleware headers).

If you find yourself needing session fields frequently, consider extending middleware and header wiring rather than adding more `getSession()` usage.

### 5.3 New API Routes

API routes may:

- Use `getSession()` from `app/(auth)/auth.ts` to:
  - Verify authentication.
  - Read tenant and role information.
- Optionally trust middleware headers where appropriate (e.g. behind internal proxies).

However:

- Do not introduce new `safeAuth`-like wrappers.
- Prefer explicit error handling around `getSession()` and token refresh flows.

---

## 6. Migration Checklist for Legacy Code

When touching legacy code that still mentions `safeAuth` in backlogs or comments:

1. **Identify usage pattern:**
   - Is it a layout/page/server component?
   - Is it a helper or API route?
2. **Replace with headers when possible:**
   - For layouts and server components, use `headers()` and the middleware-injected headers.
3. **Otherwise use `getSession()` directly:**
   - For Node-only API routes and targeted helpers.
4. **Update documentation:**
   - Mark `safeAuth` references as historical and point to this backlog.
5. **Verify behavior:**
   - `/login` and `/healthz` must not regress (no Redis/org fetch).
   - Authenticated routes continue to respect tenant/org boundaries.

---

## 7. References

- [329-auth.md](./329-auth.md) – Auth baseline and earlier patterns.
- [331-tenant-resolution.md](./331-tenant-resolution.md) – TenantId through JWT/session (now updated with middleware notes).
- [330-dynamic-tenant-resolution.md](./330-dynamic-tenant-resolution.md) – Org/tenant resolution (org-scope deprecation guidance).
- [427-platform-login-stability.md](./427-platform-login-stability.md) – RootLayout/safeAuth removal, Redis/org-fetch hardening.
- [428-onboarding.md](./428-onboarding.md) – Onboarding flows and their dependencies on login stability.
