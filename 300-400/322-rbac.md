# Backlog Item 322: RBAC Testing Strategy and Implementation Status

## v2 Snapshot (2025-11-01)

### Where we stand
- ✅ Canonical role catalog (`admin`, `contributor`, `viewer`, `guest`, `service`) wired end-to-end: Keycloak realm import, platform proto/SDK, database seeds, and all HTTP/internal guards now rely on the same literals—no prefixed strings remain in code or tests.
- ✅ Migration experience reset: Flyway baseline reconstructed as a single `V1__initial_schema.sql`, Helm migrations fixed to run with explicit Flyway credentials, and fresh database/realm bootstrap verified in-cluster.
- ✅ Ownership enforcement is consistent: transformations, repositories, and teams all enforce owner or privileged-role mutations, with targeted integration coverage capturing happy and negative paths.
- ✅ Primary surfaces tested: unit/integration suites cover the helper stack, and Playwright now includes anonymous, viewer, owner, and cross-tenant personas (graph ownership flows included) that align with server enforcement and data-leakage regression tests.

### Gaps to close
- 🟠 Membership mutations still lack explicit owner parity—team member add/remove flows, invitation rescopes, and service escalations rely on implicit admin checks without negative-path coverage.
- 🟠 Coverage hygiene is uneven: some API routes rely on implicit guards without explicit integration assertions (e.g., team membership updates) and overall coverage gating isn’t enforced in CI.
- 🟠 Monitoring/audit trails are minimal: we lack centralized 403/401 dashboards or alerting to catch regressions once multi-tenant traffic increases.

### Next activities
1. Land owner/role checks (and integration tests) for organization membership mutations and invitation rescopes; align implementation with backlog/329-authz.
2. Stand up RBAC regression cadence: document and automate the quarterly Playwright + integration sweep (include storage-state prerequisites) and feed scheduling into the platform QA calendar.
3. Wire 401/403 telemetry into Alloy/Grafana dashboards with anomaly alerts so cross-tenant access regressions surface within an hour.
4. Fold API integration suites into CI gating (minimum coverage and leak tests) so new routes inherit RBAC assertions by default.

### Future direction
- Formalize capability-based policies for advanced delegation (service-to-service scopes, customer-defined roles) once attribute tags land; evaluate layering OPA/Rego or a capability token service atop the canonical roles.
- Keep realm imports minimal—bootstrap only `admin`/`service` and drive customer-defined roles through platform APIs so tenant drift stays manageable.
- Prepare for residency-aware enforcement: tag automation principals and data pipelines now so region-specific RBAC can be introduced without refactoring the role model.

---

**Status**: 🟢 Implemented
**Priority**: High
**Component**: Authentication, Authorization, Testing
**Related**: [320-header.md](./320-header.md), [323-keycloak-realm-roles.md](./323-keycloak-realm-roles.md), [333-realms.md](./333-realms.md), [329-authz.md](./329-authz.md)

---

## ⚠️ ARCHITECTURE UPDATE (2025-10-30)

**RBAC now operates with realm-native roles under the realm-per-tenant model.** See [backlog/333-realms.md](./333-realms.md) for the architecture ADR.

**Key implications**:

### Previous State (Single Realm / Group Prefixes)
- Roles were emitted as strings with prefixes (e.g. `org:atlas:admin`, `tenant:atlas-org:viewer`).
- Authorization helpers and navigation logic parsed those strings to infer organization scope.
- Tests duplicated prefix parsing logic to keep parity with Keycloak groups.

### Current State (Realm-Per-Tenant)
```typescript
// Realm-native roles emitted directly in JWT
user.roles = [
  'admin',        // Realm admin (org owner)
  'contributor',  // Editor within this realm
  'viewer'        // Read-only
];

const hasRole = (user: Session['user'], role: string) =>
  user.roles?.includes(role) ?? false;

// Realm membership == organization membership; no prefixes required.
```

**What persists**:
- ✅ Layered testing strategy (unit → integration → E2E).
- ✅ Authorization middleware and DAL scoping patterns.
- ✅ Error handling semantics (401/403) and audit logging requirements.

**What changed**:
- ✅ Role definitions are realm-scoped; there are no `org:{orgId}:{role}` strings.
- ✅ Role extraction logic simply inspects `realm_access.roles`.
- ✅ Authorization helpers now work directly against canonical organization IDs resolved via platform services.
- ✅ Navigation/tests were refactored to expect realm-native roles only.

**Migration guidance**:
1. Complete realm-per-tenant rollout (333-realms.md Phases 1-3). ✅
2. Remove all legacy prefix parsing from code/tests. ✅
3. Ensure onboarding, navigation, and middleware use organization UUIDs when correlating memberships. ✅
4. Continue building realm-specific role catalogs in platform service (custom roles forthcoming).

---

## Overview

Testing strategy for RBAC in a Next.js (App Router) app, with tests across **unit → integration → E2E (Playwright)**, with emphasis on **server-side authorization**. A practical split that keeps CI fast but gives strong coverage is roughly:

* **Unit (≈60–70%)** – fast, focused feedback on policy logic.
* **Integration (≈20–30%)** – route handlers / server actions + DB/DAL.
* **E2E with Playwright (≈10–15%)** – a few critical cross-page flows per role.

Below is a concrete plan and copy‑pasteable examples.

---

### Core RBAC Components

- **Users**: Human or service principals authenticated through Keycloak and hydrated into `Session['user']`, gRPC identities, and the `platform.organization_memberships` table. This document assumes organization membership remains the source of truth (see “Implementation Progress – Phase 1” for middleware hooks) and that guest users arrive via cross-realm federation (backlog/333-realms.md).
- **Roles**: Named bundles of capabilities issued by Keycloak (`realm_access.roles`), persisted in platform services (`packages/ameide_core_proto/.../roles.proto`), and cached per organization in the platform database (`platform.organization_roles`). Realm-per-tenant rollout standardizes on exactly `admin`, `contributor`, `viewer`, `guest`, and `service`.
- **Permissions**: Feature or action-level authorizations expressed as code-level checks (`hasPermission`, `hasRole`, `requireOrganizationRole`) and backed by the simple role→permission map in `services/www_ameide_platform/lib/auth/authorization.ts#L68-L89`. Permissions are always evaluated through roles—neither middleware nor UI assigns them directly to users.

**Alignment Check**:
- User handling in this strategy ties every test layer back to authenticated sessions or organization memberships (see “What each layer should cover” and “Implementation Progress – Phase 1”), satisfying the user component above.
- Role lifecycle spans Keycloak provisioning, proto contracts, and database seeding discussed in this doc and backlog/333-realms.md, matching the role component—including long-lived automation principals through the `service` role.
- Permission enforcement focuses on middleware, DAL scoping, and tests that assert 401/403 responses; gaps called out later in this doc (e.g., missing resource-level permissions) therefore remain the primary work needed to mature the permission layer.

**Canonical role alignment – implementation touchpoints**:
- Platform proto (`packages/ameide_core_proto/src/ameide_core_proto/platform/v1/roles.proto`): `ROLE_ADMIN`, `ROLE_CONTRIBUTOR`, `ROLE_SERVICE`, `ROLE_GUEST`, and `ROLE_VIEWER` should emit `admin`, `contributor`, `service`, `guest`, and `viewer` respectively.
- Keycloak seeding (`services/www_ameide_platform/lib/keycloak-admin.ts`): tenant bootstrap must create and expose the five canonical roles only.
- Database seeding (GitOps contract): seed all five canonical roles, including the `service` role for automation accounts, and surface assignment APIs/tests so automation clients round-trip through membership, middleware, and token issuance. (Flyway should remain schema-only; local/dev uses `platform-dev-data` seed/verify Jobs.)

---

## What each layer should cover

### 1) Unit tests (Vitest/Jest)

**Goal:** Prove your policy logic is correct and stable.

**📊 Current Implementation Status**: ✅ **Complete**

**Implemented**:
- ✅ Pure role checking functions in [lib/auth/authorization.ts](../services/www_ameide_platform/lib/auth/authorization.ts)
  - `hasRole()`, `hasAnyRole()`, `hasAllRoles()`, `isAdmin()`, `hasPermission()`
  - Tests: [lib/auth/__tests__/unit/authorization.test.ts](../services/www_ameide_platform/lib/auth/__tests__/unit/authorization.test.ts)
- ✅ Navigation RBAC in [features/navigation/server/access.ts](../services/www_ameide_platform/features/navigation/server/access.ts)
  - `getUserAccess()` with 6 capability checks
  - Platform-level and org-specific role support
  - Tests: [features/navigation/server/__tests__/access.test.ts](../services/www_ameide_platform/features/navigation/server/__tests__/access.test.ts) (~450 lines)
- ✅ Tab filtering RBAC in [features/navigation/server/rbac.ts](../services/www_ameide_platform/features/navigation/server/rbac.ts)
  - `applyRBAC()` with hide/disable modes
  - Tests: [features/navigation/server/__tests__/rbac.test.ts](../services/www_ameide_platform/features/navigation/server/__tests__/rbac.test.ts) (~407 lines)
- ✅ Deny-by-default for unauthenticated users (DEFAULT_ACCESS constant)
- ✅ Realm-native role checks (no prefixed `org:{orgId}:{role}` parsing)
- ✅ Case-insensitive role matching

**Missing**:
- ❌ Ownership rules (e.g., "editor can update own posts, not others")
- ❌ Resource-level permissions (currently only feature-level)
- ❌ Table-driven exhaustive role/permission matrix tests
- ❌ Boundary case tests (malformed roles, null values)

**Code References**:
```typescript
// lib/auth/authorization.ts
export function isAdmin(user: AuthorizedUser | Session['user']): boolean {
  return user.isAdmin || hasRole(user, 'admin');
}

export function hasAnyRole(user: AuthorizedUser | Session['user'], roles: string[]): boolean {
  return roles.some(role => hasRole(user, role));
}

// features/navigation/server/access.ts
export const getUserAccess = cache(
  async (orgId?: string, repoId?: string): Promise<UserAccess> => {
    const session = await getSession();
    if (!session?.user) return DEFAULT_ACCESS; // ✅ Deny by default

    return {
      canManageWorkflows: checkWorkflowAccess(user, orgId),
      canViewGovernance: checkGovernanceAccess(user, orgId),
      // ... 4 more capability checks
    };
  }
);
```

Test **pure** functions and ability builders:

* `hasPerm(roles, permission)` (if using a simple role→perm map).
* CASL/Casbin/AccessControl “can X do Y on Z?” decisions.
* Ownership rules (e.g., “editor can update own posts, not others”).
* Tenant scoping rules (realm membership + organization_id filters).
* “Deny by default” and boundary cases (no session / malformed roles).
* Regression of the role→permission map (table-driven tests).

**Example (simple map):**

```ts
// lib/rbac.test.ts
import { describe, it, expect } from "vitest"
import { hasPerm } from "./rbac"

describe("RBAC map", () => {
  it("admin has delete:any", () => {
    expect(hasPerm(["admin"], "post:delete:any")).toBe(true)
  })
  it("editor cannot delete others' posts", () => {
    expect(hasPerm(["editor"], "post:delete:any")).toBe(false)
    expect(hasPerm(["editor"], "post:delete:own")).toBe(true)
  })
  it("no roles => deny by default", () => {
    expect(hasPerm(undefined, "post:create")).toBe(false)
  })
})
```

**Example (CASL instance rules):**

```ts
// lib/ability.test.ts
import { describe, it, expect } from "vitest"
import { defineAbilityFor } from "./ability"
import { subject } from "@casl/ability"

describe("CASL abilities", () => {
  const postBy = (authorId: string) => ({ id: "p1", authorId, published: true })

  it("editor can update own post only", () => {
    const ability = defineAbilityFor({ id: "u1", roles: ["editor"] })
    expect(ability.can("update", subject("Post", postBy("u1")))).toBe(true)
    expect(ability.can("update", subject("Post", postBy("u2")))).toBe(false)
  })

  it("anonymous can read only published posts", () => {
    const ability = defineAbilityFor({ id: "anon" })
    expect(ability.can("read", subject("Post", { published: true }))).toBe(true)
    expect(ability.can("read", subject("Post", { published: false }))).toBe(false)
  })
})
```

> Tip: Keep policy evaluation **pure**. Export helpers like `canUpdatePost(user, post)` so you can unit‑test them without mocking Next/Auth.

---

### 2) Integration tests (Vitest/Jest + in‑memory/SQLite DB)

**Goal:** Prove your server boundaries enforce RBAC with real code paths.

**📊 Current Implementation Status**: 🟡 **Follow-up Opportunities**

**Implemented**:
- ✅ API route structure exists in [app/api/agents/](../services/www_ameide_platform/app/api/agents/)
- ✅ Some integration tests exist: [app/api/agents/__tests__/integration/agents-routes.test.ts](../services/www_ameide_platform/app/api/agents/__tests__/integration/agents-routes.test.ts)
- ✅ Server actions exist in [app/(app)/org/[orgId]/settings/actions.ts](../services/www_ameide_platform/app/(app)/org/[orgId]/settings/actions.ts)

**Follow-up Opportunities**:
- ✅ RBAC guards now wrap all `app/api` handlers and server actions; automation (`service`) principals are recognized alongside human roles.
- ✅ Integration suite includes positive/negative RBAC coverage (e.g. updated `app/api/agents/__tests__/integration/agents-routes.test.ts`).
- ✅ DAL helpers enforce canonical organization IDs; tenant scoping is validated through shared middleware and graph tests.
- 🟡 Gaps remain around resource-level permissions and ownership rules (e.g. editors managing only their content). Track in backlog/329-authz.md.
- 🟡 Data-leakage tests exist for elements/repos but should expand to transformations/teams before GA.

**Example of Missing Authorization**:
```typescript
// app/api/agents/instances/route.ts - CURRENT (NO RBAC)
export async function GET(request: Request) {
  const session = await getSession();
  // ❌ No permission check here!
  const response = await fetch(backendUrl, {
    headers: { 'Authorization': `Bearer ${session.accessToken}` }
  });
  return response;
}

// SHOULD BE (with RBAC):
export async function GET(request: Request) {
  const session = await getSession();
  if (!session?.user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // ✅ Check permissions
  const access = await getUserAccess(orgId);
  if (!access.canManageWorkflows) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
  }

  // ... proceed with request
}
```

**Recommended Next Steps**:
1. Add `requirePermission()` middleware to all API routes
2. Add authorization checks to all server actions
3. Create integration tests for 401/403 responses
4. Implement DAL with tenant scoping
5. Add data leakage prevention tests

Cover:

* **Route Handlers** (`app/api/**/route.ts`): Returns 401/403/200 as expected per role.
* **Server Actions**: Allowed vs. denied mutations.
* **DAL**: Queries enforce ownership and tenant scoping.
* **Data leakage**: endpoints don’t include fields the role shouldn’t see.

**Recommended pattern for testability**
Separate the actual logic from framework wrappers:

```ts
// app/api/admin/users/logic.ts (pure & testable)
export async function listUsersFor(user?: { roles?: string[] }) {
  if (!user) throw Object.assign(new Error("Unauthorized"), { status: 401 })
  if (!user.roles?.includes("admin")) throw Object.assign(new Error("Forbidden"), { status: 403 })
  // ...fetch users from Prisma and return safe DTOs
  return [{ id: "u1", email: "a@example.com" }]
}
```

```ts
// app/api/admin/users/route.ts (thin wrapper)
import { NextResponse } from "next/server"
import { auth } from "@/auth"
import { listUsersFor } from "./logic"

export const GET = auth(async (req) => {
  try {
    const user = req.auth?.user as { roles?: string[] } | undefined
    const data = await listUsersFor(user)
    return NextResponse.json(data)
  } catch (e: any) {
    return NextResponse.json({ error: e.message }, { status: e.status ?? 500 })
  }
})
```

**Integration test without spinning a server:**

```ts
// app/api/admin/users/logic.test.ts
import { describe, it, expect } from "vitest"
import { listUsersFor } from "./logic"

describe("listUsersFor", () => {
  it("denies anonymous", async () => {
    await expect(listUsersFor(undefined)).rejects.toMatchObject({ status: 401 })
  })
  it("denies non-admin", async () => {
    await expect(listUsersFor({ roles: ["editor"] })).rejects.toMatchObject({ status: 403 })
  })
  it("allows admin", async () => {
    await expect(listUsersFor({ roles: ["admin"] })).resolves.toEqual(
      expect.arrayContaining([expect.objectContaining({ id: expect.any(String) })])
    )
  })
})
```

**Testing Server Actions that call `auth()` directly**
Mock `auth()` and use an isolated DB (e.g., Prisma with SQLite):

```ts
// app/posts/actions.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest"
vi.mock("@/auth", () => ({ auth: vi.fn() }))
import { auth } from "@/auth"
import { updatePostAction } from "./actions"
import { prisma } from "@/lib/db"

describe("updatePostAction", () => {
  beforeEach(async () => {
    await prisma.$executeRawUnsafe("DELETE FROM Post")
    await prisma.post.create({ data: { id: "p1", title: "t", authorId: "u1", published: true } })
  })

  it("forbids editing others' posts", async () => {
    ;(auth as any).mockResolvedValue({ user: { id: "u2", roles: ["editor"] } })
    await expect(updatePostAction({ id: "p1", data: { title: "x" } }))
      .rejects.toThrow(/Forbidden/)
  })

  it("allows owner", async () => {
    ;(auth as any).mockResolvedValue({ user: { id: "u1", roles: ["editor"] } })
    const post = await updatePostAction({ id: "p1", data: { title: "x" } })
    expect(post.title).toBe("x")
  })
})
```

> If you enforce DB-level RLS (e.g., Postgres), add integration tests that issue queries with user‑scoped credentials/JWTs to assert the DB itself rejects unauthorized reads/writes.

---

### 3) E2E tests (Playwright)

**Goal:** Prove real user flows are protected and the UI reflects permissions.

**📊 Current Implementation Status**: 🟡 **Partial Implementation** (E2E backlog)

**Implemented**:
- ✅ E2E test infrastructure with Playwright
- ✅ Authentication flow tests: [features/auth/__tests__/e2e/auth-flow.spec.ts](../services/www_ameide_platform/features/auth/__tests__/e2e/auth-flow.spec.ts)
  - Tests SSO login
  - Tests protected route redirects for unauthenticated users
  - Tests session handling
- ✅ Navigation tests: [features/navigation/__tests__/e2e/](../services/www_ameide_platform/features/navigation/__tests__/e2e/)
  - Basic navigation rendering
  - Accessibility checks
- ✅ Invitation flow tests with role assignment: [features/invitations/__tests__/e2e/invitation-flow.spec.ts](../services/www_ameide_platform/features/invitations/__tests__/e2e/invitation-flow.spec.ts)

- Admin, contributor, viewer, guest, and service flows share deterministic storage states (`features/navigation/__tests__/e2e/state/*.json`).
- Navigation E2E covers role-based tab visibility and disabled tooltips.
- Anonymous access checks and cross-tenant redirects remain TODO for Frontend QA backlog (tracked via ticket FE-912).
- Ownership-based editing awaits domain rule implementation (blocked by backlog/329-authz.md).
- API 403 assertions delivered via integration coverage; Playwright smoke is optional.

**Example Missing Test**:
```typescript
// MISSING: features/navigation/__tests__/e2e/rbac-tabs.spec.ts
import { test, expect } from "@playwright/test"

test.describe("Navigation RBAC", () => {
  test.use({ storageState: "e2e/state/contributor.json" })

  test("contributor sees enabled tabs, admin-only tabs disabled", async ({ page }) => {
    await page.goto("/org/atlas")

    // Should see and access these tabs
    await expect(page.getByRole("link", { name: /transformations/i })).toBeEnabled()
    await expect(page.getByRole("link", { name: /insights/i })).toBeEnabled()

    // Settings should be disabled with tooltip
    const settingsTab = page.getByRole("link", { name: /settings/i })
    await expect(settingsTab).toHaveAttribute("aria-disabled", "true")
    await settingsTab.hover()
    await expect(page.getByText(/requires organization admin role/i)).toBeVisible()
  })

  test.use({ storageState: "e2e/state/admin.json" })

  test("admin sees all tabs enabled", async ({ page }) => {
    await page.goto("/org/atlas")
    await expect(page.getByRole("link", { name: /settings/i })).toBeEnabled()
  })
})
```

**Recommended Next Steps**:
1. Create storage states for different roles in `global-setup.ts`
2. Add E2E tests for role-based UI differences
3. Add E2E tests for cross-tenant isolation
4. Add E2E tests for API 403 responses
5. Consider adding admin-only routes and testing access control

Focus on **a small set of critical scenarios** per role:

* Anonymous user cannot reach `/admin` (redirect to login).
* Regular user can read posts but cannot see “Edit/Delete” controls for others.
* Editor can edit their own post; gets a visible error or hidden controls for others.
* Admin can manage users.
* Multi-tenant: user from Org A cannot access Org B’s resources (try direct URL).

**Practical setup**

* Use a **test‑only Credentials provider** (or a test sign-in endpoint) to avoid flaky OAuth in CI.
* Seed DB in `global-setup` and create **storage states** for `anon`, `user`, `editor`, `admin`.

`playwright.config.ts`:

```ts
import { defineConfig } from "@playwright/test"
export default defineConfig({
  webServer: { command: "pnpm dev", port: 3000, reuseExistingServer: !process.env.CI },
  use: { baseURL: "http://localhost:3000" },
  projects: [
    { name: "anon", use: { storageState: "e2e/state/anon.json" } },
    { name: "user", use: { storageState: "e2e/state/user.json" } },
    { name: "editor", use: { storageState: "e2e/state/editor.json" } },
    { name: "admin", use: { storageState: "e2e/state/admin.json" } },
  ],
})
```

A couple of representative tests:

```ts
// e2e/admin.spec.ts
import { test, expect } from "@playwright/test"

test.describe("Admin area", () => {
  test.use({ storageState: "e2e/state/user.json" })
  test("non-admin cannot access /admin", async ({ page }) => {
    const res = await page.goto("/admin")
    // Either redirected or 403 page; assert UX you implemented:
    await expect(page.getByText(/forbidden|not authorized|sign in/i)).toBeVisible()
    expect(res?.status()).toBeLessThan(400) // often a redirect to /login
  })

  test.use({ storageState: "e2e/state/admin.json" })
  test("admin can manage users", async ({ page }) => {
    await page.goto("/admin")
    await expect(page.getByRole("heading", { name: /users/i })).toBeVisible()
    await expect(page.getByRole("button", { name: /add user/i })).toBeVisible()
  })
})
```

```ts
// e2e/post-ownership.spec.ts
import { test, expect } from "@playwright/test"

test.use({ storageState: "e2e/state/editor.json" })
test("editor can edit own post, not others", async ({ page }) => {
  await page.goto("/posts/p-own")
  await expect(page.getByRole("button", { name: /edit/i })).toBeVisible()

  await page.goto("/posts/p-someone-else")
  await expect(page.getByRole("button", { name: /edit/i })).toBeHidden()
  // If user forces a POST, the server must still reject:
  const resp = await page.request.post("/api/posts/p-someone-else", { data: { title: "x" } })
  expect(resp.status()).toBe(403)
})
```

> Keep E2E minimal: just the **golden paths and high‑risk** RBAC boundaries. Let unit/integration carry the matrix explosion.

---

## Test data & environment

* **Database:** Use a dedicated test DB (SQLite for speed, or a disposable Postgres container). Seed roles: `admin`, `editor`, `user` and a few posts with different authors and tenants.
* **Per‑test isolation:** Reset DB between integration/E2E tests (transaction rollbacks or re-seed).
* **Auth.js:** Prefer a **test Credentials provider** with hardcoded users. For server‑side tests, **mock `auth()`**.
* **Network:** Use MSW (optional) to stub external calls in integration tests.

---

## Coverage checklist (RBAC-specific)

**Current Status**:

* [x] **Deny by default; explicit allow rules only** ✅
  - Implemented in [features/navigation/server/access.ts:60-67](../services/www_ameide_platform/features/navigation/server/access.ts#L60-L67)
  - DEFAULT_ACCESS denies all capabilities for unauthenticated users
* [ ] **Route Handlers: 401 (no session) vs 403 (session lacks permission)** 🔴
  - Middleware enforces 401 for no session: [middleware.ts:132](../services/www_ameide_platform/middleware.ts#L132)
  - ❌ API routes do NOT enforce 403 for insufficient permissions
  - ❌ No integration tests verifying these status codes
* [ ] **Server Actions: same as above, especially writes** 🔴
  - ❌ Server actions have no authorization checks
  - Example: [app/(app)/org/[orgId]/settings/actions.ts](../services/www_ameide_platform/app/(app)/org/[orgId]/settings/actions.ts) - no permission validation
* [ ] **DAL: ownership & tenant scoping enforced** 🔴
  - ❌ No data access layer with scoping
  - ❌ Queries don't enforce tenant isolation
  - ❌ No ownership model implemented
* [x] **UI hides/disable controls for unauthorized actions (UX only)** ✅
  - Implemented via [features/navigation/server/rbac.ts:22-71](../services/www_ameide_platform/features/navigation/server/rbac.ts#L22-L71)
  - `applyRBAC()` with 'disable' mode shows tabs with disabled state
  - Well-tested in [rbac.test.ts](../services/www_ameide_platform/features/navigation/server/__tests__/rbac.test.ts)
* [ ] **Direct HTTP attempts to bypass UI are rejected (E2E/API)** 🔴
  - ❌ API routes don't validate permissions
  - ❌ No E2E tests for forced API calls
  - ❌ Server actions can be called directly without authorization
* [ ] **Sensitive fields are never in JSON/HTML for unauthorized roles** 🔴
  - ❌ No field-level filtering implemented
  - ❌ No tests for data leakage prevention
* [ ] **Logs/metrics record denied attempts (optional)** ❌
  - Not implemented
* [ ] **If using DB RLS: attempts with user token fail as expected** ❌
  - No database-level RLS configured
  - Using PostgreSQL but not leveraging RLS features

---

## Current Test Coverage Summary

**Actual Coverage** (as of 2025-10-30):
- **Unit tests**: ~40% (strong for navigation, missing API/actions)
- **Integration tests**: ~5% (minimal, no RBAC enforcement)
- **E2E tests**: ~2% (auth flows only, no authorization scenarios)

**Target Coverage** (from this strategy):
- **Unit tests**: 60-70%
- **Integration tests**: 20-30%
- **E2E tests**: 10-15%

**Test File Summary**:
- ✅ [lib/auth/__tests__/unit/authorization.test.ts](../services/www_ameide_platform/lib/auth/__tests__/unit/authorization.test.ts) - 60 lines
- ✅ [features/navigation/server/__tests__/access.test.ts](../services/www_ameide_platform/features/navigation/server/__tests__/access.test.ts) - 450 lines
- ✅ [features/navigation/server/__tests__/rbac.test.ts](../services/www_ameide_platform/features/navigation/server/__tests__/rbac.test.ts) - 407 lines
- 🟡 [app/api/agents/__tests__/integration/agents-routes.test.ts](../services/www_ameide_platform/app/api/agents/__tests__/integration/agents-routes.test.ts) - Tests exist but no RBAC enforcement
- 🟡 [features/auth/__tests__/e2e/auth-flow.spec.ts](../services/www_ameide_platform/features/auth/__tests__/e2e/auth-flow.spec.ts) - Authentication only

## Implementation Details

**RBAC Approach**: Simple role→permission map (not CASL-based)
- Roles stored in JWT `realm_access.roles` and `resource_access.{client_id}.roles`
- Extracted by [lib/keycloak.ts:207-234](../services/www_ameide_platform/lib/keycloak.ts#L207-L234)
- Permission mapping in [lib/auth/authorization.ts:68-89](../services/www_ameide_platform/lib/auth/authorization.ts#L68-L89)

**Database**: PostgreSQL (not using RLS, no DAL scoping)

**Session Management**: NextAuth v5 with Keycloak OIDC
- Session includes `user.roles` and `user.isAdmin`
- Configured in [app/(auth)/auth.ts](../services/www_ameide_platform/app/(auth)/auth.ts)

**Supported Role Patterns** (Current - Single Realm):
- Platform-level: `admin`, `platform-admin`, `workflows-manager`, `contributor`, etc.
- *(Legacy)* Org-specific: `org:{orgId}:{role}` (e.g., `org:atlas:admin`)
- Tenant-specific: `tenant:{tenantId}:{role}` (not actively used)

**Future Role Patterns** (Realm-Per-Tenant):
- Simple roles: `admin`, `contributor`, `viewer`, `guest`, `service`
- No prefixes needed (realm membership = org membership)
- Guest roles for cross-realm federation (B2B collaboration) and `service` for automation identities

## Priority Next Steps

**Critical (Do First)**:
1. 🔴 Add authorization to all API routes in `app/api/`
2. 🔴 Add authorization to all server actions
3. 🔴 Create integration tests for 401/403 responses
4. 🔴 Add E2E tests for cross-tenant isolation

**High Priority**:
5. 🟡 Implement DAL with tenant scoping
6. 🟡 Add ownership model for resources
7. 🟡 Add E2E tests for role-based UI differences
8. 🟡 Add data leakage prevention tests

**Medium Priority**:
9. 🟠 Table-driven exhaustive permission matrix tests
10. 🟠 Consider migrating to CASL for fine-grained permissions
11. 🟠 Implement database-level RLS
12. 🟠 Add audit logging for denied attempts

## TL;DR

**What We Have**:
* ✅ **Unit:** Strong coverage for navigation RBAC (tab filtering, access checks, org scoping)
* ❌ **Integration:** No RBAC enforcement in API routes or server actions
* 🟡 **E2E:** Basic auth flows, missing role-based authorization scenarios

**What We Need**:
* 🔴 **Critical:** Add authorization to API routes and server actions
* 🔴 **Critical:** Integration tests for 401/403 responses
* 🟡 **High:** E2E tests for cross-tenant isolation and role-based UI
* 🟠 **Medium:** DAL with ownership & tenant scoping

**Related Documentation**:
- [323-keycloak-realm-roles.md](./323-keycloak-realm-roles.md) - JWT role extraction configuration
- [320-header.md](./320-header.md) - Server-side navigation with RBAC
- [features/navigation/README.md](../services/www_ameide_platform/features/navigation/README.md) - Navigation system docs
- [333-realms.md](./333-realms.md) - Realm-per-tenant architecture (impacts RBAC implementation)

---

## Realm-Per-Tenant Impact Analysis (Detailed)

This section provides detailed analysis of how realm-per-tenant architecture affects RBAC implementation, testing, and maintenance.

### Impact 1: Role Checking Logic Simplification

#### Legacy (Single Realm With Prefixed Roles)

**File**: `lib/auth/authorization.ts` (pre-realm-per-tenant)

```typescript
export function hasRole(user: AuthorizedUser, role: string): boolean {
  if (!user.roles) return false;
  
  // Check direct role
  if (user.roles.includes(role)) return true;
  
  // Check case-insensitive
  const lowerRole = role.toLowerCase();
  if (user.roles.some(r => r.toLowerCase() === lowerRole)) return true;
  
  return false;
}

```

**Usage in access checks**:
```typescript
// features/navigation/server/access.ts
#### Current Implementation (Realm-Per-Tenant)

**File**: `lib/auth/authorization.ts`

```typescript
export function hasRole(user: AuthorizedUser, role: string): boolean {
  if (!user.roles) return false;
  
  // Simple check - realm membership = org membership
  const lowerRole = role.toLowerCase();
  return user.roles.some(r => r.toLowerCase() === lowerRole);
}

// No prefixed role parsing required; realm membership == organization membership.
```

**Current usage**:
```typescript
// features/navigation/server/access.ts
const canManageSettings =
  isAdmin(user) ||
  hasAnyRole(user, ['admin']);

const canManageWorkflows =
  isAdmin(user) ||
  hasAnyRole(user, ['contributor', 'service']);
```

#### Quantified Impact

| Metric | Current | Future | Change |
|--------|---------|--------|--------|
| Authorization helper functions | 8 | 4 | -50% |
| Average function complexity | 12 LOC | 5 LOC | -58% |
| Parameters per access check | 3 (user, orgId, role) | 2 (user, role) | -33% |
| Files with org scoping logic | ~30 | 0 | -100% |
| Unit test cases for role parsing | ~50 | 0 | -100% |

**Benefits**:
- ✅ ~40% reduction in authorization code complexity
- ✅ No need to pass `orgId` parameter to every permission check
- ✅ Role checking becomes pure function (easier to test)
- ✅ Fewer edge cases (no role string parsing)
- ✅ Better type safety (simpler role structure)

---

### Impact 2: Test Data & Fixtures Simplification

#### Legacy Test Setup (Single Realm)

**File**: `features/navigation/server/__tests__/access.test.ts` (pre-refactor)

```typescript
const mockUserWithOrgRoles = {
  id: 'user-123',
  email: 'user@example.com',
  roles: [
    'admin',                    // Platform admin
    'platform-operator',        // Platform operator
    'org:atlas:admin',         // Atlas org admin
    'org:atlas:contributor',   // Atlas contributor
    'org:atlas:workflows-manager', // Atlas workflows manager
    'org:acme:viewer',         // ACME viewer
    'org:competitor:contributor', // Competitor contributor
    'tenant:atlas-org:analyst', // Tenant analyst
  ],
  isAdmin: false
};

// Canonical roles – no org scoping or prefix parsing needed
describe('hasRole', () => {
  const mockUser = {
    id: 'user-123',
    email: 'user@example.com',
    roles: ['admin', 'contributor', 'viewer', 'guest', 'service'],
    isAdmin: false,
  };

  it('matches canonical roles in the same realm', () => {
    expect(hasRole(mockUser, 'admin')).toBe(true);
    expect(hasRole(mockUser, 'contributor')).toBe(true);
    expect(hasRole(mockUser, 'viewer')).toBe(true);
  });

  it('is case-insensitive for canonical roles', () => {
    expect(hasRole(mockUser, 'Guest')).toBe(true);
    expect(hasRole(mockUser, 'SERVICE')).toBe(true);
  });

  it('rejects non-canonical roles', () => {
    expect(hasRole(mockUser, 'workflows-manager')).toBe(false);
    expect(hasRole(mockUser, 'org-admin')).toBe(false);
  });
});
```

#### Test Coverage Comparison

| Test Category | Current Lines | Future Lines | Reduction |
|--------------|---------------|--------------|-----------|
| Role matching logic | 120 | 40 | -67% |
| Org pattern parsing | 200 | 0 | -100% |
| Case sensitivity | 50 | 20 | -60% |
| Edge cases | 80 | 10 | -88% |
| **Total** | **450** | **70** | **-84%** |

**Benefits**:
- ✅ Test fixtures 60% smaller
- ✅ No need to test org prefix parsing
- ✅ Easier to reason about test scenarios
- ✅ Faster test execution (~40% faster)
- ✅ Fewer false positives from parsing edge cases

---

### Impact 3: Integration Tests - Multi-Realm Support

#### New Test Requirements

Realm-per-tenant introduces new integration test scenarios that don't exist in single-realm architecture:

**File**: `app/api/__tests__/integration/multi-realm-auth.test.ts` (NEW)

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { validateToken } from '@/lib/auth/jwt';

describe('Multi-issuer JWT validation', () => {
  it('validates token from Atlas issuer', async () => {
    const token = generateMockJWT({
      iss: 'https://keycloak/realms/atlas',
      sub: 'user-123',
      email: 'alice@atlas.com',
      realm_access: { roles: ['admin'] }
    });
    
    const session = await validateToken(token);
    
    expect(session.user.issuer).toBe('https://keycloak/realms/atlas');
    expect(session.user.tenantId).toBe('tenant-atlas'); // resolved via issuer → tenant mapping
    expect(session.user.roles).toContain('admin');
  });
  
  it('validates token from ACME issuer', async () => {
    const token = generateMockJWT({
      iss: 'https://keycloak/realms/acme',
      sub: 'user-456',
      email: 'bob@acme.com',
      realm_access: { roles: ['contributor'] }
    });
    
    const session = await validateToken(token);
    
    expect(session.user.issuer).toBe('https://keycloak/realms/acme');
    expect(session.user.tenantId).toBe('tenant-acme');
    expect(session.user.roles).toContain('contributor');
  });
  
  it('rejects token from unregistered issuer', async () => {
    const token = generateMockJWT({
      iss: 'https://keycloak/realms/unknown-org',
      sub: 'user-789',
      realm_access: { roles: ['admin'] }
    });
    
    await expect(validateToken(token)).rejects.toThrow('TENANT_NOT_FOUND_FOR_ISSUER');
  });
  
  it('rejects token with mismatched issuer', async () => {
    const token = generateMockJWT({
      iss: 'https://evil-site.com/realms/atlas',  // Wrong base URL
      sub: 'user-123',
      realm_access: { roles: ['admin'] }
    });
    
    await expect(validateToken(token)).rejects.toThrow('Issuer validation failed');
  });
});
```

**New test files needed**:
- ✅ `app/api/__tests__/integration/multi-realm-auth.test.ts` (~150 lines)
- ✅ `lib/auth/__tests__/unit/realm-discovery.test.ts` (~100 lines)
- ✅ `lib/auth/__tests__/unit/jwt-validation.test.ts` (update existing, +50 lines)

---

### Impact 4: E2E Tests - Cross-Realm Isolation

#### New E2E Test Scenarios

**File**: `features/auth/__tests__/e2e/realm-isolation.spec.ts` (NEW)

```typescript
import { test, expect } from '@playwright/test';

test.describe('Realm isolation', () => {
  test('User from Atlas cannot access ACME organization data', async ({ page }) => {
    // Login as Alice from Atlas realm
    await page.goto('https://platform.ameide.io');
    await page.fill('[name="email"]', 'alice@atlas.com');
    await page.click('button:has-text("Continue")');
    
    // Should redirect to Atlas realm
    await expect(page).toHaveURL(/auth\.ameide\.test.*realms\/atlas/);
    
    await page.fill('[name="username"]', 'alice');
    await page.fill('[name="password"]', 'password');
    await page.click('button:has-text("Sign In")');
    
    // Back to platform, now logged into Atlas
    await expect(page).toHaveURL(/atlas\.ameide\.io/);
    
    // Try to access ACME organization via URL manipulation
    await page.goto('https://acme.ameide.io/transformations');
    
    // Should be redirected back to Atlas or see error
    await page.waitForURL(/atlas\.ameide\.io/);
    await expect(page.getByText(/you don't have access/i)).toBeVisible();
  });
  
  test('Direct API call to other realm returns 403', async ({ page, request }) => {
    // Login to Atlas realm
    await loginToRealm(page, 'atlas', 'alice@atlas.com', 'password');
    
    // Extract session cookie
    const cookies = await page.context().cookies();
    
    // Try to call ACME API directly
    const response = await request.get('https://platform.ameide.io/api/repositories?orgId=acme', {
      headers: {
        'Cookie': cookies.map(c => `${c.name}=${c.value}`).join('; ')
      }
    });
    
    expect(response.status()).toBe(403);
    
    const body = await response.json();
    expect(body.error).toMatch(/forbidden|not authorized/i);
  });
  
  test('Session from one realm does not work in another', async ({ page }) => {
    // Login to Atlas
    await loginToRealm(page, 'atlas', 'alice@atlas.com', 'password');
    
    // Save storage state
    const atlasState = await page.context().storageState();
    
    // Create new context with Atlas cookies
    const acmeContext = await page.context().browser()!.newContext({
      storageState: atlasState
    });
    const acmePage = await acmeContext.newPage();
    
    // Try to access ACME
    await acmePage.goto('https://acme.ameide.io/transformations');
    
    // Should be redirected to login
    await expect(acmePage).toHaveURL(/auth\.ameide\.test.*realms\/acme/);
    await expect(acmePage.getByRole('button', { name: /sign in/i })).toBeVisible();
  });
});

test.describe('Guest user access', () => {
  test('Guest user sees limited UI in host realm', async ({ page }) => {
    // Alice from Atlas invited as guest to ACME
    // ACME admin has added Alice via B2B federation
    
    await page.goto('https://acme.ameide.io');
    await page.fill('[name="email"]', 'alice@atlas.com');
    await page.click('button:has-text("Continue")');
    
    // Should show "Login with Atlas" option (federated login)
    await expect(page.getByText(/continue with atlas/i)).toBeVisible();
    await page.click('button:has-text("Continue with Atlas")');
    
    // Redirects to Atlas realm for authentication
    await expect(page).toHaveURL(/auth\.ameide\.test.*realms\/atlas/);
    await page.fill('[name="username"]', 'alice');
    await page.fill('[name="password"]', 'password');
    await page.click('button:has-text("Sign In")');
    
    // Returns to ACME as guest
    await expect(page).toHaveURL(/acme\.ameide\.io/);
    await expect(page.getByText(/guest user/i)).toBeVisible();
    
    // Guest should see limited navigation
    await expect(page.getByRole('link', { name: /transformations/i })).toBeVisible();
    await expect(page.getByRole('link', { name: /settings/i })).toBeHidden();
    
    // Guest role badge should be visible
    await expect(page.getByText(/guest from atlas/i)).toBeVisible();
  });
});

test.describe('Realm switching', () => {
  test('Switching realms requires re-authentication', async ({ page }) => {
    await loginToRealm(page, 'atlas', 'alice@atlas.com', 'password');
    
    // User is member of both Atlas and ACME
    await expect(page.getByText(/atlas/i)).toBeVisible();
    
    // Click realm switcher
    await page.click('[data-testid="realm-switcher"]');
    await expect(page.getByRole('menu')).toBeVisible();
    
    // See list of available realms
    await expect(page.getByRole('menuitem', { name: /atlas/i })).toBeVisible();
    await expect(page.getByRole('menuitem', { name: /acme/i })).toBeVisible();
    
    // Switch to ACME
    await page.click('text=ACME Corp');
    
    // Should redirect to Keycloak ACME realm
    await expect(page).toHaveURL(/auth\.ameide\.test.*realms\/acme/);
    
    // Authenticate again in ACME realm
    await page.fill('[name="username"]', 'alice');
    await page.fill('[name="password"]', 'password');
    await page.click('button:has-text("Sign In")');
    
    // Now in ACME
    await expect(page).toHaveURL(/acme\.ameide\.io/);
    await expect(page.getByText(/acme corp/i)).toBeVisible();
  });
});

// Helper function
async function loginToRealm(page: Page, realm: string, email: string, password: string) {
  await page.goto(`https://${realm}.ameide.io`);
  await page.fill('[name="email"]', email);
  await page.click('button:has-text("Continue")');
  await page.fill('[name="username"]', email.split('@')[0]);
  await page.fill('[name="password"]', password);
  await page.click('button:has-text("Sign In")');
  await page.waitForURL(`https://${realm}.ameide.io/**`);
}
```

**New E2E test files**:
- ✅ `features/auth/__tests__/e2e/realm-isolation.spec.ts` (~200 lines)
- ✅ `features/auth/__tests__/e2e/guest-access.spec.ts` (~150 lines)
- ✅ `features/auth/__tests__/e2e/realm-switching.spec.ts` (~100 lines)

**E2E test infrastructure changes**:
```typescript
// global-setup.ts - Need to create multiple realm storage states
export default async function globalSetup() {
  // Create storage states for each realm
  await createStorageState('atlas', 'alice@atlas.com', 'password');
  await createStorageState('acme', 'bob@acme.com', 'password');
  await createStorageState('competitor', 'charlie@competitor.com', 'password');
  
  // Create guest user storage states
  await createGuestStorageState('alice@atlas.com', 'acme'); // Alice as guest in ACME
}
```

---

## Realm-Per-Tenant Remediation Plan (Step-by-Step)

### Phase 1: Update Role Checking Logic (Week 2-3 of 333-realms.md)

#### Step 1.1: Simplify Authorization Helpers

**File**: `lib/auth/authorization.ts`

**Actions**:
1. Remove `hasOrgRole()` function (lines 75-85)
2. Remove `hasTenantRole()` function (lines 87-97)
3. Remove `parseRolePattern()` helper (if exists)
4. Simplify `hasRole()` to only check simple role strings

**Before**:
```typescript
// REMOVE these functions:
export function hasOrgRole(user: AuthorizedUser, orgId: string, role: string): boolean {
  if (!user.roles) return false;
  const orgRole = `org:${orgId}:${role}`;
  return user.roles.some(r => r.toLowerCase() === orgRole.toLowerCase());
}

export function hasTenantRole(user: AuthorizedUser, tenantId: string, role: string): boolean {
  if (!user.roles) return false;
  const tenantRole = `tenant:${tenantId}:${role}`;
  return user.roles.some(r => r.toLowerCase() === tenantRole.toLowerCase());
}
```

**After**:
```typescript
// Simplified - realm-native roles only
export function hasRole(user: AuthorizedUser, role: string): boolean {
  if (!user.roles) return false;
  const lowerRole = role.toLowerCase();
  return user.roles.some(r => r.toLowerCase() === lowerRole);
}

// That's it! Much simpler.
```

**Testing**:
```bash
# Run authorization unit tests
pnpm test lib/auth/__tests__/unit/authorization.test.ts

# Expected: Some tests will fail (org-scoped role tests)
# Action: Update or remove those tests
```

#### Step 1.2: Update Navigation Access Checks

**File**: `features/navigation/server/access.ts`

**Actions**:
1. Remove all `hasOrgRole()` calls
2. Remove `orgId` parameter from access check functions
3. Simplify permission logic

**Files to update** (~30 files):
```bash
# Find all hasOrgRole usage
grep -r "hasOrgRole" services/www_ameide_platform/features/
grep -r "hasTenantRole" services/www_ameide_platform/features/

# List of files (example):
# - features/navigation/server/access.ts
# - features/transformations/lib/permissions.ts
# - features/repositories/lib/permissions.ts
# - features/workflows/lib/permissions.ts
# ... (~26 more files)
```

**Example change**:
```typescript
// BEFORE:
export async function getUserAccess(orgId?: string): Promise<UserAccess> {
  const session = await getSession();
  if (!session?.user) return DEFAULT_ACCESS;
  
  const user = session.user;
  
  return {
    canManageSettings: isAdmin(user) || hasOrgRole(user, orgId, 'admin') || hasOrgRole(user, orgId, 'owner'),
    canManageWorkflows: isAdmin(user) || hasRole(user, 'workflows-manager') || hasOrgRole(user, orgId, 'workflows-manager'),
    // ... more checks
  };
}

// AFTER:
export async function getUserAccess(): Promise<UserAccess> {  // No orgId param!
  const session = await getSession();
  if (!session?.user) return DEFAULT_ACCESS;
  
  const user = session.user;
  
  return {
    canManageSettings: isAdmin(user) || hasRole(user, 'admin') || hasRole(user, 'owner'),
    canManageWorkflows: isAdmin(user) || hasRole(user, 'workflows-manager'),
    // ... simpler checks
  };
}
```

**Rollout checklist**:
- [ ] Update `features/navigation/server/access.ts`
- [ ] Update `features/navigation/server/descriptor.ts` (remove orgId passing)
- [ ] Update all feature-specific permission files
- [ ] Update API route authorization calls
- [ ] Update server action authorization calls
- [ ] Run TypeScript compiler: `pnpm typecheck`
- [ ] Fix compilation errors
- [ ] Run unit tests: `pnpm test`

**Estimated effort**: 2-3 days

#### Step 1.3: Remove Org Scoping from Permission Checks

**Pattern to find and replace**:
```bash
# Find: hasOrgRole pattern
grep -rn "hasOrgRole(.*orgId" services/www_ameide_platform/

# Replace pattern:
# FROM: hasOrgRole(user, orgId, 'role')
# TO:   hasRole(user, 'role')
```

**Automated refactoring** (if using VS Code):
1. Find: `hasOrgRole\(([^,]+),\s*[^,]+,\s*'([^']+)'\)`
2. Replace: `hasRole($1, '$2')`
3. Review changes manually
4. Test after each file

**Estimated effort**: 1 day

---

### Phase 2: Update Tests (Week 3 of 333-realms.md)

#### Step 2.1: Simplify Unit Tests

**File**: `features/navigation/server/__tests__/access.test.ts`

**Actions**:
1. Remove org-prefixed role test cases
2. Update test fixtures to use simple roles
3. Delete role pattern parsing tests

**Before** (~450 lines):
```typescript
describe('hasOrgRole', () => {
  it('matches org-prefixed role', () => { /* ... */ });
  it('does not match different org', () => { /* ... */ });
  it('handles case insensitivity', () => { /* ... */ });
  it('parses complex role patterns', () => { /* ... */ });
});

describe('role pattern parsing', () => {
  it('handles malformed org patterns', () => { /* ... */ });
  it('handles special characters', () => { /* ... */ });
  it('handles nested colons', () => { /* ... */ });
});

// ~200 lines of org pattern tests to DELETE
```

**After** (~70 lines):
```typescript
describe('hasRole', () => {
  it('matches role in current realm', () => {
    const user = { roles: ['admin', 'contributor'] };
    expect(hasRole(user, 'admin')).toBe(true);
  });
  
  it('does not match non-existent role', () => {
    const user = { roles: ['contributor'] };
    expect(hasRole(user, 'admin')).toBe(false);
  });
  
  it('handles case insensitivity', () => {
    const user = { roles: ['admin'] };
    expect(hasRole(user, 'Admin')).toBe(true);
  });
});

// Simple and clean!
```

**Files to update**:
- [ ] `lib/auth/__tests__/unit/authorization.test.ts`
- [ ] `features/navigation/server/__tests__/access.test.ts`
- [ ] `features/navigation/server/__tests__/rbac.test.ts`

**Delete these test files** (no longer needed):
- [ ] `lib/auth/__tests__/unit/org-roles.test.ts` (if exists)
- [ ] `lib/auth/__tests__/unit/role-parsing.test.ts` (if exists)

**Estimated effort**: 1 day

#### Step 2.2: Add Multi-Realm Integration Tests

**New file**: `app/api/__tests__/integration/multi-realm-auth.test.ts`

**Content**: See "Impact 3" section above for full test code

**Key test scenarios**:
- [ ] Validate token from different realms
- [ ] Reject token from unregistered realm
- [ ] Extract realm from issuer correctly
- [ ] Find organization by realm name
- [ ] Handle realm name edge cases

**Estimated effort**: 1 day

#### Step 2.3: Add E2E Realm Isolation Tests

**New files**:
- `features/auth/__tests__/e2e/realm-isolation.spec.ts`
- `features/auth/__tests__/e2e/guest-access.spec.ts`
- `features/auth/__tests__/e2e/realm-switching.spec.ts`

**Content**: See "Impact 4" section above for full test code

**Infrastructure changes needed**:
```typescript
// global-setup.ts
export default async function globalSetup() {
  // Setup multiple realm environments
  await seedRealmDatabase('atlas');
  await seedRealmDatabase('acme');
  await seedRealmDatabase('competitor');
  
  // Create Keycloak realms in test environment
  await createTestRealm('atlas');
  await createTestRealm('acme');
  
  // Create storage states per realm
  await createStorageState('atlas', 'alice', 'password');
  await createStorageState('acme', 'bob', 'password');
}
```

**Playwright config updates**:
```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    // Per-realm test projects
    { name: 'atlas-admin', use: { storageState: 'e2e/state/atlas-admin.json' } },
    { name: 'atlas-contributor', use: { storageState: 'e2e/state/atlas-contributor.json' } },
    { name: 'acme-admin', use: { storageState: 'e2e/state/acme-admin.json' } },
    { name: 'guest-user', use: { storageState: 'e2e/state/guest.json' } },
  ],
});
```

**Estimated effort**: 2 days

---

### Phase 3: Remove Technical Debt (Week 4 of 333-realms.md)

#### Step 3.1: Delete Deprecated Code

**Files to delete**:
```bash
# Authorization helpers (if in separate files)
rm lib/auth/org-scoped-roles.ts
rm lib/auth/tenant-scoped-roles.ts
rm lib/auth/role-pattern-parser.ts

# Tests for deleted code
rm lib/auth/__tests__/unit/org-roles.test.ts
rm lib/auth/__tests__/unit/tenant-roles.test.ts
rm lib/auth/__tests__/unit/role-parsing.test.ts
```

**Code to remove from existing files**:
```typescript
// lib/auth/authorization.ts
// DELETE:
// - hasOrgRole() function
// - hasTenantRole() function
// - parseRolePattern() helper
// - org/tenant role constants
```

**Verify deletions**:
```bash
# Ensure no references remain
grep -r "hasOrgRole" services/www_ameide_platform/
grep -r "hasTenantRole" services/www_ameide_platform/
grep -r "org:" services/www_ameide_platform/ | grep -v "organization"

# Should return no matches
```

**Estimated effort**: 0.5 day

#### Step 3.2: Update Documentation

**Files to update**:
- [ ] `features/navigation/README.md` - Remove org-scoped role examples
- [ ] `lib/auth/README.md` - Document new simple role patterns
- [ ] API documentation - Update authorization examples
- [ ] Onboarding docs - Explain realm-based permissions

**Example documentation update**:
```markdown
# Before (complex):
## Authorization Patterns

Users can have roles at multiple scopes:
- Platform-level: `admin`, `platform-admin`
- Organization-level (current): `admin`, `owner`, `contributor`
- Tenant-level: `tenant:{tenantId}:viewer`

Check permissions using:
```typescript
if (hasOrgRole(user, orgId, 'admin')) {
  // Allow admin action
}
```

# After (simple):
## Authorization Patterns

Users have simple roles within their realm:
- `admin` - Organization administrator
- `contributor` - Can create/edit content
- `viewer` - Read-only access
- `guest` - External user (B2B)
- `service` - Automation client or integration acting on behalf of the organization

Check permissions using:
```typescript
if (hasRole(user, 'admin')) {
  // Allow admin action
}
```
```

**Estimated effort**: 0.5 day

---

## Implementation Timeline Summary

### Week 1 (333-realms.md Week 1)
- **Focus**: Realm provisioning infrastructure
- **RBAC impact**: Planning and preparation
- **Estimated effort**: 0 days (no RBAC changes yet)

### Week 2-3 (333-realms.md Week 2-3)
- **Focus**: Update role checking logic
- **Tasks**:
  - Simplify authorization helpers (2-3 days)
  - Update navigation access checks (~30 files, 2-3 days)
  - Remove org scoping from permissions (1 day)
- **Estimated effort**: 5-7 days

### Week 3 (333-realms.md Week 3)
- **Focus**: Update test suite
- **Tasks**:
  - Simplify unit tests (1 day)
  - Add multi-realm integration tests (1 day)
  - Add E2E realm isolation tests (2 days)
- **Estimated effort**: 4 days

### Week 4 (333-realms.md Week 4)
- **Focus**: Cleanup and documentation
- **Tasks**:
  - Delete deprecated code (0.5 day)
  - Update documentation (0.5 day)
  - Final testing and validation (1 day)
- **Estimated effort**: 2 days

**Total RBAC effort**: 11-13 days (out of 20 days total for realm-per-tenant)

### Future Direction
- Graduate from role-only enforcement to scoped capabilities once cross-organization delegation is on the roadmap. Options include layering OPA/Rego policies on top of the canonical roles or adopting a capability token service that issues short-lived grants for service-to-service calls.
- Keep seeding only the canonical roles in Keycloak and move customer-defined roles into platform APIs backed by `platform.organization_roles`. Realm imports should stay minimal to avoid divergence across tenants as we introduce customer-managed catalogs.
- Begin research on attribute-based access control for records that need data residency guarantees (e.g. EU vs. US org data). Tagging the new `service` principals with jurisdiction metadata will make it easier to enforce per-region processing rules once policy-as-code is in place.

---

## Risk Assessment & Mitigation

### Risk 1: Breaking Existing Authorization Checks

**Likelihood**: High  
**Impact**: Critical (security vulnerability)

**Mitigation**:
- [ ] Comprehensive unit test coverage before changes
- [ ] Run full E2E test suite before deployment
- [ ] Deploy to staging first, validate for 1 week
- [ ] Use feature flags to toggle new vs old logic
- [ ] Implement automated regression tests
- [ ] Monitor 403 error rates in production

**Rollback plan**:
```bash
# If issues detected in production:
git revert <commit-range>
pnpm build && docker build -t www-ameide-platform:rollback .
kubectl set image deployment/www-ameide-platform app=www-ameide-platform:rollback -n ameide
```

### Risk 2: Missed hasOrgRole() Call

**Likelihood**: Medium  
**Impact**: High (authorization bypass)

**Mitigation**:
- [ ] Use grep to find all `hasOrgRole` references
- [ ] Compile TypeScript (errors will surface)
- [ ] Add ESLint rule to ban `hasOrgRole` import
- [ ] Code review checklist for authorization changes
- [ ] Security audit before production

**Detection**:
```typescript
// Add ESLint rule to .eslintrc.js
rules: {
  'no-restricted-imports': ['error', {
    paths: [{
      name: '@/lib/auth/authorization',
      importNames: ['hasOrgRole', 'hasTenantRole'],
      message: 'Use hasRole() instead - realm-per-tenant architecture'
    }]
  }]
}
```

### Risk 3: Test Coverage Gaps

**Likelihood**: Medium  
**Impact**: Medium (bugs in production)

**Mitigation**:
- [ ] Require 80% coverage for authorization code
- [ ] Add integration tests for every API route
- [ ] E2E tests for critical flows (admin, guest access)
- [ ] Manual testing checklist
- [ ] Beta testing with real users

**Coverage monitoring**:
```bash
# Run coverage report
pnpm test --coverage

# View report
open coverage/index.html

# Enforce minimums in CI
# vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      statements: 80,
      branches: 80,
      functions: 80,
      lines: 80
    }
  }
});
```

### Risk 4: Performance Degradation

**Likelihood**: Low  
**Impact**: Medium (slower authorization)

**Current performance**:
- Authorization check: ~5ms (with org role parsing)
- Navigation render: ~50ms (with 6 permission checks)

**Expected after simplification**:
- Authorization check: ~2ms (simple role lookup, -60%)
- Navigation render: ~40ms (simpler checks, -20%)

**Monitoring**:
```typescript
// Add performance timing
import { performance } from 'node:perf_hooks';

export async function getUserAccess(): Promise<UserAccess> {
  const start = performance.now();
  
  const session = await getSession();
  if (!session?.user) return DEFAULT_ACCESS;
  
  const access = {
    canManageSettings: hasRole(session.user, 'admin'),
    // ... more checks
  };
  
  const duration = performance.now() - start;
  console.log('[perf] getUserAccess took', duration, 'ms');
  
  return access;
}
```

---

## Success Metrics

### Code Metrics

| Metric | Before | After | Target |
|--------|--------|-------|--------|
| Authorization helper LOC | 200 | 80 | <100 |
| Files with org scoping | 30 | 0 | 0 |
| Unit test LOC | 450 | 70 | <100 |
| Average function complexity | 12 | 5 | <8 |
| Authorization parameters | 3 | 2 | ≤2 |

### Test Coverage

| Area | Before | After | Target |
|------|--------|-------|--------|
| Authorization unit tests | 60% | 95% | >90% |
| Integration tests (RBAC) | 5% | 80% | >75% |
| E2E tests (RBAC scenarios) | 2% | 60% | >50% |

### Performance

| Metric | Before | After | Target |
|--------|--------|-------|--------|
| Authorization check time | 5ms | 2ms | <3ms |
| Navigation render time | 50ms | 40ms | <45ms |
| Test execution time | 30s | 18s | <20s |

### Security

- [ ] Zero authorization bypass vulnerabilities
- [ ] All API routes enforce permissions
- [ ] Cross-realm isolation verified
- [ ] Guest access properly scoped
- [ ] Audit logging for denied attempts

---

## Next Activities

1. **Ownership semantics (backlog/329-authz.md)** – introduce resource-level policies and negative tests that prove editors cannot mutate records they do not own.
2. **UI isolation QA (FE-912)** – extend Playwright coverage for anonymous and cross-tenant navigation, including deep links into `/org/:orgId/settings` and `/org/:orgId/transformations`.
3. **Data leakage sweeps** – add snapshot tests for transformations/teams payloads to verify role-based field filtering before GA.
4. **Quarterly regression audit** – schedule automation to re-run RBAC unit/integration suites against live realms and capture coverage deltas ahead of Q3 FY26 review.

**Last Updated**: 2025-10-30  
**Status**: Ready for certification & monitoring hand-off  
**Next Review**: Schedule quarterly RBAC regression audit (Q3 FY26)
