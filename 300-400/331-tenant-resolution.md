> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Tenant Resolution - Implementation Status

## ⚠️ ARCHITECTURE CHANGE: Realm-Per-Tenant (2025-10-30)

**🚨 THIS DOCUMENT DESCRIBES THE INTERIM SINGLE-REALM ARCHITECTURE**

The long-term architecture is **realm-per-tenant** where each organization gets its own Keycloak realm. See **[backlog/333-realms.md](./333-realms.md)** for the target architecture.

**Why the change?**
- Enterprise customers need to configure their own SSO (Okta, Azure AD, Google Workspace)
- Complete tenant isolation required for compliance
- Organization admins should manage their own identity provider settings
- Aligns with industry standards (Microsoft Entra ID, Okta, Auth0)

**Migration status**:
- ✅ Phase 1 complete (this document) - Single realm with JWT tenantId
- 🔄 Phase 2 planned - Realm-per-tenant implementation (333-realms.md)
- ⏳ Migration timeline - 3-4 weeks for full realm-per-tenant rollout
- 📄 Onboarding flow built on this target model: see [backlog/319-onboarding-v2.md](./319-onboarding-v2.md)

**What changes with realm-per-tenant**:
- ❌ No more single "ameide" realm - each org gets dedicated realm
- ❌ No more Keycloak groups for organization membership (`/orgs/{slug}`)
- ❌ No more `org_groups` JWT claim
- ✅ Realm name IS the organization: `iss: "https://keycloak/realms/acme"`
- ✅ Each org configures own IdP, password policies, branding
- ✅ Guest users via realm-to-realm federation

**What remains from Phase 1**:
- ✅ JWT-based authentication (now multi-realm)
- ✅ No hardcoded tenant defaults
- ✅ Session-based tenant validation
- ✅ Security patterns (all still applicable)

---

## ✅ COMPLETED: JWT-Based Multi-Tenant Authentication (Phase 1)

**Completed Date**: 2025-10-30
**Status**: All DEFAULT_TENANT constants eliminated from production code (single-realm architecture)
**Remediation Date**: 2025-10-30 (13 additional files hardened)
**Security Fix**: Critical tenant forgery vulnerability patched in agents API
**⚠️ Note**: This implementation uses single realm + groups. Will be superseded by realm-per-tenant (333-realms.md).

### Implementation Summary

We have successfully implemented **maximum security multi-tenant architecture** where `tenantId` flows exclusively from Keycloak JWT tokens through the entire application stack with **zero hardcoded fallbacks**.

### Architecture Flow (Implemented – original, before safeAuth deprecation)

```
Keycloak (tenantId user attribute)
  ↓
JWT Token (tenantId claim via protocol mapper)
  ↓
Next-Auth Session (session.user.tenantId)
  ↓
├─ Server Components (via safeAuth())
├─ API Routes (via getSession())
├─ Client Components (via useSession())
└─ Server Actions (via safeAuth())
```

> **Update (2025‑12‑01):** `safeAuth()` has been removed from the platform. The current flow is:
>
> - Middleware validates the session and injects `x-tenant-id`, `x-user-id`, `x-user-kc-sub`, and `x-org-home` headers.
> - Server components and layouts use these headers instead of calling `safeAuth()` to derive `tenantId`.
> - API routes and server actions that need full session details call `getSession()` from `app/(auth)/auth.ts` directly.

### Files Refactored (33+ total: 20 Phase 1 + 13 Remediation)

#### Authentication Core
- ✅ `app/(auth)/auth.ts` - JWT callback extracts tenantId, session callback exposes it
- ✅ `types/auth.d.ts` - Added tenantId to Session, JWT, KeycloakAccessToken interfaces
- ✅ `types/next-auth.d.ts` - Type augmentation for Next-Auth

#### API Routes (7 files)
- ✅ `app/api/elements/route.ts`
- ✅ `app/api/transformations/route.ts`
- ✅ `app/api/transformations/[transformationId]/route.ts`
- ✅ `app/api/repositories/route.ts`
- ✅ `app/api/repositories/[graphId]/route.ts`
- ✅ `app/api/threads/messages/route.ts`
- ✅ `app/api/threads/stream/route.ts`
- ✅ `app/api/agents/instances/route.ts`
- ✅ `app/api/agents/instances/[agentId]/route.ts`
- ✅ `app/api/agents/invoke/route.ts`

#### Server Components & Layouts (3 files)
- ✅ `app/(app)/layout.tsx` - Passes tenantId to navigation descriptor
- ✅ `app/(app)/org/[orgId]/layout.tsx` - Validates tenantId before org lookup
- ✅ `app/(app)/org/[orgId]/settings/actions.ts` - Server actions use session tenantId

#### Client Hooks & Features (4 files)
- ✅ `lib/api/hooks.ts` - All workflows hooks use session.user.tenantId
- ✅ `lib/auth/org-scope.ts` - Prioritizes session.user.tenantId
- ✅ `features/threads/hooks/useChatContext.ts` - Uses session tenantId for threads context
- ✅ `features/agents/hooks/useAgentMutations.ts` - Requires valid tenantId
- ✅ `features/agents/api.ts` - Validates tenantId parameter

#### Server-Side Navigation (2 files)
- ✅ `features/navigation/server/features.ts` - Accepts tenantId parameter
- ✅ `features/navigation/server/descriptor.ts` - Passes tenantId for feature fetching

#### Feature Components (6 files)
- ✅ `features/editor-agent/AgentDesignerCanvas.tsx`
- ✅ `features/transformations/components/InitiativeSectionShell.tsx`
- ✅ `app/(app)/org/[orgId]/transformations/[transformationId]/page.tsx`
- ✅ `app/(app)/org/[orgId]/settings/workflows/[workflowsId]/page.tsx`
- ✅ `lib/sdk/organizations.ts` - SDK accepts tenantId parameter

### Keycloak Configuration

**Realm**: `ameide` (configured in `infra/kubernetes/charts/platform/keycloak-realm/values.yaml`)

**Client Scope**: `tenant` with protocol mapper:
```yaml
clientScopes:
  - name: tenant
    description: Tenant ID claim for multi-tenancy
    protocol: openid-connect
    protocolMappers:
      - name: tenantId
        protocol: openid-connect
        protocolMapper: oidc-usermodel-attribute-mapper
        config:
          user.attribute: tenantId
          claim.name: tenantId
          access.token.claim: "true"
          id.token.claim: "true"
```

**Default Scopes**: Added `tenant` to `defaultDefaultClientScopes` for all clients

### Security Improvements

✅ **Session-based tenant validation** - All API routes extract tenantId from JWT session
✅ **JWT-based tenant isolation** - tenantId flows from Keycloak → Session → All services
✅ **Fail-secure by default** - Missing tenantId returns 400/401 errors
✅ **Request body protection** - Agent routes and all endpoints ignore client-provided tenantId
✅ **Type-safe implementation** - All changes pass TypeScript validation

#### Post-Verification Remediation (2025-10-30)

Following comprehensive security audit, **13 additional files** were hardened:

**V1 API Routes (9 files)** - Eliminated 'atlas-org' hardcoded fallbacks:
- `/api/v1/invitations/*` - 4 routes (create, accept, get, resend)
- `/api/v1/organizations/create` - Organization creation
- `/api/v1/organizations/[orgId]/invitations` - List invitations
- `/api/user/settings` - User settings endpoint
- `/api/user/profile` - User profile endpoint

**Registration Flow (3 files)** - Session-based tenant propagation:
- `/api/v1/registrations/complete` - Now extracts & validates session tenantId
- `features/identity/lib/orchestrator.ts` - Updated to accept tenantId parameter
- `features/identity/types/index.ts` - Added tenantId to request type

**Security Vulnerabilities Fixed (2 files)**:
- `/api/agents/instances` - **CRITICAL**: POST/PUT handlers were not validating session, allowing client to forge tenantId
- `features/editor-agent/session/AgentSession.ts` - Removed hardcoded 'atlas-org'

**Appropriate Environment Fallbacks Documented (1 file)**:
- `lib/auth/organization.ts` - Clarified environment fallbacks are ONLY for UI/navigation, NOT API routes

### Validation Results

```bash
✅ TypeScript compilation: PASSED (0 errors in modified files)
✅ Production code scan: 0 DEFAULT_TENANT instances found
✅ Hardcoded fallbacks: Eliminated from 13 additional API routes
✅ Security audit: Fixed critical tenant forgery vulnerability in agents API
✅ Architecture pattern: All API routes use session.user.tenantId from JWT
✅ 33+ total files refactored: 20 (Phase 1) + 13 (Remediation)
```

### Code Pattern Example

**Before (hardcoded fallback):**
```typescript
const DEFAULT_TENANT_ID = process.env.NEXT_PUBLIC_TENANT_ID ?? 'atlas-org';
const context = create(common.RequestContextSchema, {
  tenantId: DEFAULT_TENANT_ID,
});
```

**After (session-based):**
```typescript
const session = await getSession();
if (!session?.user) {
  return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
}

const tenantId = session.user.tenantId;
if (!tenantId) {
  return NextResponse.json({ error: 'Tenant ID not available' }, { status: 400 });
}

const context = create(common.RequestContextSchema, {
  tenantId,
});
```

---

## 🔄 NEXT PHASE: ~~Organization-as-Subtenant Pattern~~ → REALM-PER-TENANT

**⚠️ DEPRECATED**: The "organization as subtenant" pattern described below is being **replaced** with **realm-per-tenant** architecture.

**See [backlog/333-realms.md](./333-realms.md) for the new architecture.**

**Why deprecated?**
- Single-realm approach doesn't support per-organization SSO configuration
- Enterprise customers need control over their identity provider settings
- Realm-per-tenant provides stronger security boundaries
- Aligns with Microsoft Entra ID multi-tenant model

**The content below is kept for historical reference only.**

---

## ~~Organization-as-Subtenant Pattern (DEPRECATED)~~

Below is a practical, battle‑tested tenant‑resolution pattern for a **single Next.js app** running in **Kubernetes** behind **Envoy**, using **Keycloak** for auth and **backend microservices** for data. It explicitly models **"organization as subtenant."**

**⚠️ This approach is superseded by realm-per-tenant (333-realms.md).**

---

## 0) Vocabulary (so we’re precise)

* **Deployment tenant (tenant_id)**: your platform install (e.g., `atlas-org`). This is mostly static per cluster/region and is helpful for global isolation, ops, and telemetry.
* **Organization (org / subtenant)**: a customer space inside the tenant (`ACME Corp`, `Competitor Inc`). This is what you’ll resolve on each request.
* **Org slug/id**: stable identifiers (slug: `acme`; id: UUID `org_123…`) used in URLs, tokens, headers, and DB. Prefer UUID for storage, slug for routing/UX.

> Example hierarchy (your sample)

```
Tenant: atlas-org
├─ Organization: ACME Corp         (slug: acme, id: org_acme)
└─ Organization: Competitor Inc    (slug: competitor, id: org_comp)
```

---

## 1) End‑to‑end request flow (high level)

1. **Envoy** (edge) identifies the target **organization** based on host/subdomain or vanity domain and adds **trusted headers**:

   * `x-tenant-id: atlas-org`
   * `x-org-slug: acme` (or sets a canonical path prefix)
   * It must **strip any inbound x-org-* headers** to prevent header injection.

2. **Next.js middleware** derives the **org context** from trusted headers and/or URL, canonicalizes the URL shape (e.g., rewrites to `/o/:org/...`), and sets/reads a secure **active‑org** cookie for convenience.

3. **Auth**: the user authenticates with **Keycloak**. Their JWT includes the organizations they belong to (and per‑org roles). On each request, Next.js **verifies** the user is a member of the resolved org. If not: 403 or org switcher.

4. **Next.js → microservices**: forward the **user’s access token**, and include **org scope** (e.g., `X-Org-Id` or `X-Org-Slug`) to each backend. Backends enforce org isolation (preferably **DB Row Level Security** or explicit filters).

5. **Data**: all multitenant data is keyed by `org_id` (and optionally `tenant_id`). Backends set the DB session variable (e.g., `SET LOCAL atlas.org_id = $org_id`) so **RLS** can do the heavy lifting.

---

## 2) Canonical URL patterns (pick 1 primary + 1 fallback)

**Primary (recommended)**: subdomain/vanity‑domain, canonicalized to a path:

* Public URL: `https://acme.atlas.example.com/transformations`
* **Middleware rewrite** to internal canonical path: `/o/acme/transformations`
* Vanity domains: `https://app.acme.com` → `/o/acme/...` the same way

**Fallback**: explicit path

* `https://atlas.example.com/o/acme/transformations`

> **Why canonicalize to `/o/:org`?**
> It makes server components, routing, and breadcrumbs predictable and simplifies static asset/caching rules. Subdomains are for UX; path prefix drives your app logic.

---

## 3) Envoy (edge) responsibilities

* Terminate TLS, parse `:authority`.
* Determine `org_slug` from host/subdomain/vanity mapping (ideally via a small **org‑resolver** service or RDS/xDS).
* **Never** forward user‑supplied `x-org-*` headers; explicitly **remove** them.
* Inject trusted context for downstream:

  * `x-tenant-id: atlas-org` (constant per deployment)
  * `x-org-slug: <resolved-slug>`

**Sketch (conceptual)**

```yaml
# Key ideas; adapt to your Envoy/Istio/Gateway API flavor
http_filters:
  # Optional: ext_authz calling an org-resolver that returns headers
  - name: envoy.filters.http.ext_authz
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
      http_service:
        server_uri:
          uri: http://org-resolver.svc.cluster.local:8080
        authorization_request:
          allowed_headers:
            patterns:
              - exact: ":authority"
              - exact: "x-forwarded-host"
        authorization_response:
          allowed_upstream_headers:
            patterns:
              - exact: "x-tenant-id"
              - exact: "x-org-slug"

# Also ensure request_headers_to_remove removes any inbound x-org-* headers
# and request_headers_to_add sets x-tenant-id if ext_authz is not used.
```

> The **org‑resolver** is a tiny service (or even a Lua filter) that maps hostnames → slugs using your Org Registry. It lets you support:
> `acme.atlas.example.com` and `app.acme.com` → `x-org-slug: acme`.

---

## 4) Next.js middleware: resolve org, canonicalize URL, set active‑org cookie

> Works with the **App Router** (Next.js 13+). Runs at the **Edge runtime**, so keep it lean.

**`middleware.ts`**

```ts
import { NextRequest, NextResponse } from "next/server";

const RESERVED = new Set(["www", "app", "id", "api"]);
const TENANT_ID = "atlas-org";                      // your deployment

function findOrgSlug(req: NextRequest): string | null {
  // 1) From trusted Envoy header (preferred)
  const fromEnvoy = req.headers.get("x-org-slug");
  if (fromEnvoy && /^[a-z0-9-]+$/.test(fromEnvoy)) return fromEnvoy;

  // 2) From /o/:org path
  const m = req.nextUrl.pathname.match(/^\/o\/([a-z0-9-]+)(?:\/|$)/);
  if (m) return m[1];

  // 3) Best-effort from host (only if you cannot use Envoy header)
  const host = (req.headers.get("x-forwarded-host") ?? req.headers.get("host") ?? "").toLowerCase();
  const parts = host.split(".");
  if (parts.length >= 3) {
    const sub = parts[0];
    if (sub && !RESERVED.has(sub) && /^[a-z0-9-]+$/.test(sub)) return sub;
  }
  return null;
}

export function middleware(req: NextRequest) {
  const url = req.nextUrl;
  const orgSlug = findOrgSlug(req);
  const headers = new Headers(req.headers);

  // Always carry a trusted tenant id (either from Envoy or constant per deployment)
  headers.set("x-tenant-id", TENANT_ID);

  if (orgSlug) {
    headers.set("x-org-slug", orgSlug); // normalize if it came from path/host
    // Keep a secure “active org” cookie for convenience (non-authoritative)
    const existing = req.cookies.get("__Host-act_org")?.value;
    if (existing !== orgSlug) {
      const res = NextResponse.next({ request: { headers } });
      res.cookies.set("__Host-act_org", orgSlug, {
        httpOnly: true,
        sameSite: "lax",
        secure: true,
        path: "/",
        maxAge: 60 * 60 * 24 * 30, // 30 days
      });
      // Canonicalize to /o/:org if not already there
      if (!url.pathname.startsWith(`/o/${orgSlug}`)) {
        url.pathname = `/o/${orgSlug}${url.pathname === "/" ? "" : url.pathname}`;
        return NextResponse.rewrite(url, { request: { headers } });
      }
      return res;
    }

    // Canonicalize path shape
    if (!url.pathname.startsWith(`/o/${orgSlug}`)) {
      url.pathname = `/o/${orgSlug}${url.pathname === "/" ? "" : url.pathname}`;
      return NextResponse.rewrite(url, { request: { headers } });
    }
    return NextResponse.next({ request: { headers } });
  }

  // No org discovered: send to org-picker (public) or a helpful 404
  url.pathname = "/pick-organization";
  return NextResponse.rewrite(url, { request: { headers } });
}

export const config = {
  // Don’t run on static assets; adjust as needed
  matcher: ["/((?!_next|favicon.ico|robots.txt|sitemap.xml).*)"],
};
```

> **Security note**: the cookie is **not** authoritative. You must still validate org membership against the **JWT** on every request that touches data.

---

## 5) Keycloak modeling (single realm; organizations as groups)

**Recommended** for “orgs are subtenants”:

* **Realm**: `atlas-org` (one realm per deployment/region).
* **Clients**:

  * `web` (public/confidential) for the Next.js app.
  * One **bearer-only** client per microservice (e.g., `repo-service`, `transformation-service`), or a shared audience if you prefer.
* **Groups** (organizations):

  ```
  /orgs/acme         (attribute: org_id = org_acme)
  /orgs/competitor   (attribute: org_id = org_comp)
  ```
* **Roles**:

  * Client roles on `web` and/or realm roles like `org_admin`, `org_member`, `repo_write`, `repo_read`…
  * Assign roles at the **group** level so membership yields per‑org roles.
* **Protocol mappers** (added to a client scope attached to `web`):

  * **Group Membership** → claim `org_slugs: ["acme","competitor"]` (strip `/orgs/` prefix).
  * **Group Attribute** mapper → claim `org_map` (array of objects or parallel arrays), e.g.

    ```json
    {
      "orgs": [
        {"slug":"acme","id":"org_acme","roles":["org_admin","repo_write"]},
        {"slug":"competitor","id":"org_comp","roles":["org_member"]}
      ],
      "tenant": "atlas-org"
    }
    ```
  * Include `tenant` as a static hardcoded claim or via realm attribute mapper.

> You can also add an **“active_org”** app‑level cookie/setting; if you must put it in the token, Keycloak’s client session notes mapper can do it, but it complicates flows. Most teams keep **active org** out of the token and let the **URL + header** define it.

---

## 6) Next.js server utilities (guard rails)

**`lib/orgContext.ts`**

```ts
import { headers } from "next/headers";

export function getOrgContext() {
  const h = headers();
  const tenantId = h.get("x-tenant-id") || "atlas-org";
  const orgSlug = h.get("x-org-slug");                // from middleware/envoy canonicalization
  if (!orgSlug) throw new Error("Organization not resolved.");
  return { tenantId, orgSlug };
}
```

**Authorize per‑org (example with NextAuth + Keycloak)**

```ts
// app/api/_auth.ts
import { getServerSession } from "next-auth";
import { authOptions } from "@/auth"; // your NextAuth config
import { getOrgContext } from "@/lib/orgContext";

export async function requireOrgMembership() {
  const session = await getServerSession(authOptions);
  if (!session?.user) throw new Response("Unauthorized", { status: 401 });

  const { orgSlug } = getOrgContext();
  // Your Keycloak JWT should be on the session (e.g., session.accessToken)
  // and contain orgs; or decode the token server-side.
  const orgs: Array<{ slug: string; roles?: string[] }> =
    (session as any)?.orgs || (session as any)?.token?.orgs || [];

  const membership = orgs.find(o => o.slug === orgSlug);
  if (!membership) throw new Response("Forbidden (org)", { status: 403 });

  return { session, orgSlug, roles: membership.roles ?? [] };
}
```

**Route handler using the guard**

```ts
// app/o/[org]/api/repositories/route.ts
import { NextRequest } from "next/server";
import { requireOrgMembership } from "@/app/api/_auth";
import { getOrgContext } from "@/lib/orgContext";

export async function GET(req: NextRequest) {
  const { session } = await requireOrgMembership();
  const { orgSlug } = getOrgContext();

  const res = await fetch(
    `http://repo-service.svc.cluster.local:8080/orgs/${orgSlug}/repositories`,
    {
      headers: {
        // Forward the user’s KC access token; or do token exchange if needed
        Authorization: `Bearer ${(session as any).accessToken}`,
        "X-Org-Slug": orgSlug,
      },
      cache: "no-store",
    }
  );

  if (!res.ok) return new Response("Backend error", { status: 502 });
  return new Response(await res.text(), { status: 200 });
}
```

---

## 7) Microservices: enforce org isolation (defense in depth)

* **Trust boundary**: only trust `X-Org-*` from **Envoy/Next.js** inside the cluster (mTLS or network policy required).

* **AuthZ**: validate the **Keycloak JWT** (`iss`, `aud`, `exp`) and **ensure the token’s org list includes the resolved org**. Do **not** rely on headers alone.

* **DB isolation**:

  * Schema option A (simple): one logical DB with `tenant_id`, `org_id` columns + **RLS**.
  * Schema option B (heavyweight): schema‑per‑org or DB‑per‑org.
  * With Postgres RLS:

    ```sql
    -- Example: repositories(org_id uuid, name text, ...)
    ALTER TABLE repositories ENABLE ROW LEVEL SECURITY;

    CREATE POLICY org_isolation ON repositories
      USING (org_id = current_setting('atlas.org_id', true)::uuid);

    -- At request start (per connection/tx):
    -- SET LOCAL atlas.org_id = 'org_acme';
    ```

* **Service checks** (pseudo):

  ```ts
  const token = verifyJwt(authzHeader);
  const orgFromHeader = req.get("x-org-id") || req.get("x-org-slug");
  assertUserBelongsToOrg(token, orgFromHeader); // 403 if not
  setDbOrgContext(orgFromHeader);               // SET LOCAL for RLS
  ```

---

## 8) Caching & ISR/SSR

* **Vary** any cache by **Host + `/o/:org`** (or `x-org-slug`) to avoid cross‑org leaks.
* Default to **`cache: "no-store"`** for org‑scoped API responses; selectively enable ISR where responses are org‑public and safe.
* Disable prefetch for org‑sensitive links if needed: `<Link prefetch={false} />`.

---

## 9) Bootstrap & lifecycle

1. **Create org** (Org Registry service): generate `org_id`, `slug`, and optional vanity domain.
2. **Keycloak**: create group `/orgs/<slug>`, set `org_id` attribute; assign roles; add users (Alice → ACME; Charlie → Competitor).
3. **Routing**: update Envoy/xDS mapping (or org‑resolver’s cache) for the new domain/subdomain.
4. **Audit**: emit an org‑scoped event (`tenant_id`, `org_id`, `user_id`, `action`).

---

## 10) Observability & safety rails

* Put `tenant_id`, `org_id/slug`, and `user_id` in your **structured logs** and tracing spans at **Envoy, Next.js, and each microservice**.
* **Explicitly remove** any inbound `x-tenant-id`, `x-org-*` headers at Envoy. Only Envoy sets them.
* Rate‑limit and WAF rules: scope by org where possible to protect noisy neighbors.

---

## 11) Worked example with your hierarchy

* **ACME Corp (Alice, Bob)** uses `acme.atlas.example.com`.

  * Envoy resolves → `x-org-slug: acme`, injects `x-tenant-id: atlas-org`.
  * Alice’s JWT includes `orgs: [{slug:"acme", roles:["org_admin"]}]`.
  * Next.js middleware rewrites `/transformations` → `/o/acme/transformations`.
  * Backend enforces `org_id = org_acme` via RLS.
* **Competitor Inc (Charlie)** uses `competitor.atlas.example.com`.

  * Same flow with `competitor`. Alice **cannot** access `competitor` because her token lacks that org.

---

## 12) Implementation Status Checklist

### Phase 1: Tenant ID Resolution ✅ COMPLETE
* [x] **JWT Token**: Keycloak protocol mapper adds `tenantId` claim from user attribute
* [x] **Session Management**: Next-Auth extracts and exposes `session.user.tenantId`
* [x] **API Routes**: All routes validate session and extract `tenantId` (no fallbacks)
* [x] **Server Components**: Layouts and pages use middleware-injected headers (e.g. `x-tenant-id`) to get `tenantId` instead of calling `safeAuth()`
* [x] **Client Components**: Hooks use `useSession()` to access `tenantId`
* [x] **Server Actions**: Actions use `getSession()` when they need explicit session context; `safeAuth()` is no longer used
* [x] **Type Safety**: TypeScript interfaces updated across the stack
* [x] **Security**: Zero hardcoded `DEFAULT_TENANT_ID` constants in production code

### Phase 2: Organization-as-Subtenant (Planned)
* [ ] **Edge**: Envoy maps host → `x-org-slug`; strips user‑supplied org headers; sets `x-tenant-id`.
* [ ] **Middleware**: canonicalize to `/o/:org`, set a convenience cookie, and forward trusted headers.
* [ ] **Auth**: Keycloak single realm; orgs as groups; roles assigned per group; mappers add `orgs` claims.
* [ ] **Guard**: verify org membership server‑side on every org‑scoped route/API.
* [ ] **Backends**: verify JWT; check org; enforce RLS/filters; pass org via header and DB session.
* [ ] **Caching**: vary by host+org; conservative defaults.
* [ ] **Observability**: include tenant/org/user in logs & traces.

---

### Alternatives & notes

* **Realms per customer** (stronger isolation) add operational overhead and complicate multi‑org users. Since you want **“organization as subtenant,”** one realm per deployment is the smoother path.
* If you must support **anonymous org‑scoped pages** (e.g., public repo listings), still resolve org at the edge and vary caches appropriately.

---

If you want, I can tailor the code to your exact Next.js version (Pages vs App Router), or sketch a tiny **org‑resolver** service (Go/Node) that Envoy can call to populate `x-org-slug`.
