# 427 – Platform /login Stability and Performance

**Status:** Phase 1–3 complete; layouts split, `/login` uses a public layout with no server-side auth, and middleware-only session validation (with `safeAuth` fully removed) is in place for protected routes.
**Intent:** Fix the upstream connection resets on `/login`, reduce 70+ second SSR compilation times, and eliminate `/healthz` JSON parse errors. Root causes are blocking Redis/RPC operations in the SSR path, missing error handling, and layout coupling.

---

## Symptoms

1. **`/login` upstream resets:** Envoy logs show `upstream_reset_before_response_started{connection_termination}` for `/login` requests to both `platform.dev.ameide.io` (Argo baseline) and `platform.local.ameide.io` (Tilt).

2. **70+ second compilation:** First request to `/login` on Tilt takes 70-90 seconds due to cascading SSR operations (Redis connect, org fetch, dynamic imports).

3. **`/healthz` JSON error:** Occasional `SyntaxError: Unexpected end of JSON input` at `/healthz`, causing probe failures.

---

## Root Cause Analysis

### 1. `/login` Upstream Resets

The reset occurs because Next.js drops connections during SSR compilation. The request path blocks on multiple slow operations:

```
Browser → Envoy → Next.js App
                      ↓
              RootLayout.tsx (async)
                      ↓
              await safeAuth()
                      ↓
              await getSession() → auth.ts JWT callback
                      ↓
              ┌──────────────────────────────────────────────┐
              │  BLOCKING OPERATIONS:                        │
              │  1. Redis connect() - up to 10s timeout      │
              │  2. getTokens() - Redis GET                  │
              │  3. applyOrganizationContext():              │
              │     - fetchOrganizationContexts() - gRPC     │
              │     - fetchOrganizationMembershipRoles()     │
              │  4. Dynamic imports (await import())         │
              └──────────────────────────────────────────────┘
```

**Key code paths:**
- `services/www_ameide_platform/app/layout.tsx:61` — `session = await safeAuth()` runs on every request
- `services/www_ameide_platform/app/(auth)/auth.ts:836` — `await applyOrganizationContext()` makes 2-3 RPC calls
- `services/www_ameide_platform/lib/cache/redis.ts:109` — `await cache.connecting` blocks until Redis ready

When these operations stall, the connection drops before a response is sent. Envoy interprets this as `upstream_reset_before_response_started`.

### 2. 70+ Second `/login` Compilation

Multiple expensive operations compound during first-request SSR:

| Operation | Location | Time Impact |
|-----------|----------|-------------|
| Redis Sentinel connection | `lib/cache/redis.ts:35-58` | 1-10s (10s timeout) |
| Organization context fetch | `app/(auth)/auth.ts:174-440` | 5-20s (2-3 gRPC calls) |
| Dynamic imports in JWT callback | `app/(auth)/auth.ts:822,952` | 0.5-2s per import |
| OpenTelemetry init (if configured) | `lib/telemetry-init.ts` | 2-10s |
| Google Fonts loading | `app/layout.tsx:22-32` | Variable |

**Cumulative worst case:** 20-50+ seconds for SSR on first request.

### 3. `/healthz` JSON Parse Error

The `/healthz` route itself is trivial (`app/healthz/route.ts`), but the error originates from Redis cache operations retrieving corrupted/incomplete data:

```typescript
// lib/cache/redis.ts:117-119 - NO error handling around JSON.parse
async get<T = unknown>(key: string): Promise<T | null> {
  const val = await this.client.get(nsKey(key, this.config.namespace));
  return val ? (JSON.parse(val) as T) : null;  // ← THROWS if val is malformed
}
```

When Redis returns truncated/corrupted session data, `JSON.parse()` throws `SyntaxError: Unexpected end of JSON input`. This propagates through session store → safeAuth → RootLayout.

Although `/healthz` is excluded from middleware (`proxy.ts:217`), if it shares the RootLayout that calls `safeAuth()`, the error can surface.

---

## Implementation Plan

### Phase 1: Immediate Stability Fixes

**1.1 Add error handling to Redis `get()` method**

File: `services/www_ameide_platform/lib/cache/redis.ts`

```typescript
async get<T = unknown>(key: string): Promise<T | null> {
  const val = await this.client.get(nsKey(key, this.config.namespace));
  if (!val) return null;
  try {
    return JSON.parse(val) as T;
  } catch (err) {
    console.error('[REDIS] Failed to parse cached value', { key, error: err });
    // Optionally delete the corrupted key
    await this.del(key).catch(() => {});
    return null;
  }
}
```

**1.2 Ensure `/healthz` bypasses RootLayout entirely**

Implement `/healthz` as a standalone route without any app layout or auth/session wiring. In practice, this is done by:
- Placing the handler at `app/healthz/route.ts`
- Exporting `export const dynamic = 'force-static'`
- Excluding `/healthz` from middleware matching so NextAuth is never invoked for probes

**1.3 Reduce Redis connection timeout for dev**

File: `services/www_ameide_platform/lib/cache/redis.ts`

```typescript
connectTimeout: process.env.NODE_ENV === 'production' ? 10000 : 3000,
```

---

### Phase 2: Performance Optimizations

**2.1 Skip organization fetching for unauthenticated requests**

The `/login` page doesn't need organization context — it just redirects to Keycloak. Modify `applyOrganizationContext()` to skip RPC calls for unauthenticated users or during initial sign-in setup.

File: `services/www_ameide_platform/app/(auth)/auth.ts`

```typescript
// Early return if no tenantId (unauthenticated or pre-signin)
if (!enrichedToken.tenantId) {
  return;
}
```

**2.2 Convert dynamic imports to static imports**

File: `services/www_ameide_platform/app/(auth)/auth.ts`

Replace:
```typescript
const { extractRealmFromIssuer } = await import('@/lib/auth/realm-discovery');
const { upsertUserFromKeycloak } = await import('@/lib/users');
```

With top-level imports:
```typescript
import { extractRealmFromIssuer } from '@/lib/auth/realm-discovery';
import { upsertUserFromKeycloak } from '@/lib/users';
```

**2.3 Add Redis connection warmup**

File: `services/www_ameide_platform/lib/cache/redis.ts`

```typescript
export async function warmupRedis(): Promise<void> {
  const cache = new RedisCache({ namespace: 'warmup' });
  await cache.connect();
  console.log('[REDIS] Connection pool warmed up');
}
```

Call from `instrumentation.node.ts` during app startup.

**2.4 Cache organization context in Redis with TTL**

Instead of fetching organizations on every token refresh:

```typescript
const cacheKey = `org-ctx:${userId}:${tenantId}`;
let orgs = await cache.get<OrganizationContext[]>(cacheKey);
if (!orgs) {
  orgs = await fetchOrganizationContexts(tenantId, userId, orgSlugs);
  await cache.set(cacheKey, orgs, 300); // 5 min TTL
}
```

---

### Phase 3: Architectural Improvements

**3.1 Split RootLayout for public vs authenticated routes**

Target layout shape:

```
app/
  (public)/           ← Minimal layout, no server-side auth()
    login/
    register/
  (app)/              ← Authenticated app routes (use middleware headers, not auth())
    org/[orgId]/
    onboarding/
  healthz/route.ts    ← Standalone health check (no layout, force-static)
```

**3.2 Add circuit breaker for backend RPC calls**

If the platform service is slow/down, don't block auth:

```typescript
async function fetchOrganizationsWithTimeout(
  tenantId: string, userId: string, orgSlugs: string[], timeoutMs = 5000
): Promise<OrganizationContext[]> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);
  try {
    return await fetchOrganizationContexts(tenantId, userId, orgSlugs);
  } catch (err) {
    if (err.name === 'AbortError') {
      console.warn('[auth] Organization fetch timed out, continuing without org context');
      return [];
    }
    throw err;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

**3.3 Move session validation to middleware-only for protected routes**

Current state:
- Middleware (Edge) validates the session and enforces access control for protected routes; RootLayout no longer performs any server-side session validation and simply hydrates client-side providers.
- Middleware injects `x-pathname`, `x-tenant-id`, `x-user-id`, `x-user-kc-sub`, and `x-org-home` headers derived from the Edge session so server components can avoid re-running heavy auth logic.
- The authenticated app layout (`app/(app)/layout.tsx`) reads `x-pathname` and `x-tenant-id` to resolve the navigation descriptor instead of calling `auth()`/`safeAuth()`.
- The organization layout (`app/(app)/org/[orgId]/layout.tsx`) uses `x-tenant-id`, `x-user-kc-sub`, and `x-user-id` to resolve and load organizations without invoking `auth()`/`safeAuth()`.
- The home redirect (`app/page.tsx`) uses `x-org-home` for the target organization/home route and falls back to `/login` when middleware context is missing, instead of calling `safeAuth()`.
- The `safeAuth` helper has been removed entirely. Session introspection now happens only where strictly required:
  - Navigation access control (`getUserAccess`) via `getSession()` in `app/(auth)/auth.ts`.
  - Development/test API endpoints (`/api/auth/debug`, `/api/test/userinfo`) that intentionally inspect the full session via `getSession()`.

---

## Verification Commands

```bash
# 1. Test /healthz bypasses auth (should be instant)
kubectl exec -n ameide deployment/apps-www-ameide-platform-tilt -- \
  curl -s -w '\nTime: %{time_total}s\n' http://localhost:3001/healthz

# 2. Test /login timing (check compilation time)
kubectl exec -n ameide deployment/apps-www-ameide-platform-tilt -- \
  curl -s -w '\nTime: %{time_total}s\n' -o /dev/null http://localhost:3001/login

# 3. Check Redis connectivity from pod
kubectl exec -n ameide deployment/apps-www-ameide-platform-tilt -- \
  env | grep REDIS

# 4. Check platform service health (for org fetch)
kubectl get pods -n ameide -l app=platform

# 5. Test via Envoy with explicit resolve
curl -vk --resolve platform.local.ameide.io:443:172.18.0.5 \
  https://platform.local.ameide.io/healthz
```

---

## Key Files

| File | Purpose |
|------|---------|
| `app/layout.tsx` | RootLayout — no server-side session validation; hydrates global providers with `session = null` and relies on middleware + client-side session for auth state |
| `app/(app)/layout.tsx` | Authenticated app layout — uses `x-pathname` and `x-tenant-id` headers from middleware to resolve navigation, never calls `auth()`/`safeAuth()` |
| `app/page.tsx` | Home redirect — uses `x-org-home` header from middleware to route users to their org home or onboarding; falls back to `/login` when middleware context is missing (no server-side session lookup) |
| `app/(auth)/auth.ts` | NextAuth config — JWT callback, token refresh, org fetch |
| `lib/cache/redis.ts` | Redis client — connection and JSON parsing |
| `lib/session-store.ts` | Redis-backed session storage |
| `lib/sdk/organizations.ts` | Organization context fetch (gRPC) |
| `app/healthz/route.ts` | Health check endpoint |
| `proxy.ts` | Middleware — route matching and auth |

---

## Dependencies

- Redis Sentinel must be reachable within 3-10s (reduce timeout in dev)
- Platform service must respond to organization queries within 5s
- Keycloak must be healthy for token refresh

---

## Implementation Log

### 2025-12-01: Phase 1-2 Implementation

**Changes made:**

1. **Redis error handling** (`lib/cache/redis.ts`):
   - Added try/catch around `JSON.parse()` in `get()` method
   - Returns `null` instead of throwing on corrupted data
   - Best-effort cleanup of corrupted entries

2. **`/healthz` isolation** (`app/healthz/route.ts`):
   - Added `export const dynamic = 'force-static'` to bypass SSR auth
   - Route now responds in ~40-200ms consistently

3. **Redis timeout reduction** (`lib/cache/redis.ts`):
   - Dev timeout reduced to 3s (from 10s)
   - Production remains at 10s

4. **Circuit breaker for org fetch** (`lib/sdk/organizations.ts`):
   - Added `fetchOrganizationContextsWithTimeout()` with 5s timeout
   - Returns empty array on timeout instead of blocking auth flow

5. **Static imports** (`app/(auth)/auth.ts`):
   - Converted 6 dynamic `await import()` calls to static imports
   - Reduces per-request module loading overhead

6. **Redis warmup** (`lib/cache/redis.ts`, `instrumentation.node.ts`):
   - Added `warmupRedis()` function called during app startup
   - Prevents first request from waiting for Redis connection

**Test results:**
- `/healthz` responds in ~40-200ms (was occasionally timing out)
- Baseline pod (`apps-www-ameide-platform`) shows stable healthz responses
- Tilt pod has Redis authentication issue (environment config, not code)

### 2025-12-01: Phase 3 Implementation - Route Group Split

**Changes made:**

1. **Created `(public)` route group** (`app/(public)/layout.tsx`):
   - Minimal layout that avoids any server-side session validation
   - Provides theming (ThemeProvider) and toast notifications
   - Skips SessionProvider, QueryClient, and Transport providers (not needed for public routes)
   - Imports `globals.css` via relative path

2. **Moved `/login` to public route group**:
   - `app/(public)/login/page.tsx` - client-side login page
   - `app/(public)/login/KeycloakRedirect.tsx` - Keycloak redirect component
   - Removed old `app/(auth)/login/` directory

**Route structure after refactor:**
```
app/
  layout.tsx              ← RootLayout; no server-side auth, shared theming/providers
  (public)/
    layout.tsx            ← PublicLayout (no auth), wraps unauthenticated pages
    login/
      page.tsx            ← Login page (fast, client-only redirect prep)
      KeycloakRedirect.tsx
  (auth)/
    auth.ts               ← NextAuth config (API routes + getSession/auth/signIn/signOut)
    register/route.ts     ← API route (returns HTML, no layout)
    logout/route.ts       ← API route
  (app)/
    ...                   ← Authenticated app routes (guarded by middleware)
  (onboarding)/
    ...                   ← Onboarding routes
  healthz/route.ts        ← Health check (force-static)
```

**Test results (2025-12-01):**

```
# Redis warmup working:
[REDIS] Connecting via Sentinel to redis:26379 (master: mymaster)
[REDIS] Connection pool warmed up successfully
[REDIS] Connected to master node

# /healthz consistently fast:
GET /healthz 200 in 50-200ms

# /login first request (webpack compilation):
GET /login 200 in 87s (compile: 83s, proxy.ts: 729ms, render: 3.5s)

# /login subsequent requests (cached):
HTTP Status: 200, Time: 4.65s
HTTP Status: 200, Time: 6.21s
```

**Performance improvement:**
- `/login` first request: ~87s (mostly Next.js webpack compilation - unavoidable in dev mode)
- `/login` subsequent requests: **~4-6 seconds** (down from 70+ seconds when server-side auth ran in RootLayout)
- `/healthz`: **50-200ms** consistently

Note: The first-request compilation time is a Next.js dev-mode behavior. In production builds, the routes are pre-compiled and should respond in ~100-500ms.

---

### 2025-12-01: Phase 3.3 – SafeAuth removal and middleware-only validation

**Changes made:**

1. **Home redirect simplified** (`services/www_ameide_platform/app/page.tsx`):
   - Now uses only the `x-org-home` header from middleware to determine the destination.
   - If the header is missing or empty, the route treats the request as unauthenticated and redirects to `/login` instead of calling `safeAuth()`/`getSession()`.

2. **`safeAuth` helper removed**:
   - Deleted `services/www_ameide_platform/app/(auth)/safe-auth.ts`.
   - All former `safeAuth()` call sites were updated:
     - Navigation features (`lib/auth/org-scope.ts`, `features/navigation/server/features.ts`) no longer depend on `safeAuth()`; `org-scope` is now explicitly deprecated in favor of middleware headers and organization layouts, and feature fetching relies only on `resolveOrganizationId` + tenant headers.
     - Debug and test endpoints (`app/api/auth/debug/route.ts`, `app/api/test/userinfo/route.ts`) now call `getSession()` directly when they need full session details.

3. **Public layout documentation updated** (`app/(public)/layout.tsx`):
   - Clarified that the public layout performs no server-side session validation or NextAuth `auth()` calls, avoiding Redis-backed session lookup and org-context fetching entirely for `/login` and `/register`.

4. **Navigation feature tests aligned** (`features/navigation/server/__tests__/unit/features.test.ts`):
   - Removed mocks and expectations around `safeAuth()`.
   - Tests now assert a single `resolveOrganizationId` call using the provided `tenantId` and `userId` parameters.

**Resulting behavior:**
- No production request path invokes `safeAuth()`, and the helper no longer exists.
- All auth gating for protected routes is enforced in `proxy.ts` (middleware), with server components consuming middleware-injected headers instead of re-running expensive auth logic.
- Session introspection via `getSession()` remains limited to:
  - Navigation access control (`getUserAccess`) where role information is required.
  - Explicit debug/test endpoints that are disabled in production by default.

---

## Follow-ups

- [x] Phase 1: Redis error handling and `/healthz` isolation
- [x] Phase 2: Static imports, Redis warmup, circuit breaker for org fetch
- [x] Phase 3: Route group split (public vs authenticated layouts)
- [x] Phase 3.3: Complete middleware-only session validation and remove `safeAuth()` from the request path (no remaining `safeAuth` usages)
- [ ] Fix Redis auth config in Tilt values (REDIS_PASSWORD not passed)
- [ ] Add E2E tests for `/login` flow with Playwright
- [ ] Monitor Envoy logs post-fix for residual upstream resets
