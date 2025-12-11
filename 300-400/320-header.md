# Header Refactor - Implementation Status

**Last Updated:** 2025-10-30
**Overall Status:** 🟡 **85% Complete - Critical Integration Gap**

## Executive Summary

All 7 PRs have been **implemented** with excellent code quality, but there is a **critical integration gap**: the server-side navigation infrastructure (PR 2) has been built but is **not being used** by `Header.tsx`, which still relies entirely on client-side resolution.

### Quick Status

| PR | Status | Files | Integration | Critical Issue |
|----|--------|-------|-------------|----------------|
| PR 1 | ✅ Complete | server/access.ts, server/rbac.ts | ⚠️ Duplicate client logic | Client-side RBAC duplicates server |
| PR 2 | 🟡 Built, Not Used | server/descriptor.ts, server/patterns.ts | ❌ Not integrated | **Header.tsx ignores server descriptor** |
| PR 3 | ✅ Complete | features/header/components/* | ✅ Integrated | None |
| PR 4 | ✅ Complete | server/features.ts | ⚠️ Client duplication | Client still uses Zustand store |
| PR 5 | ✅ Complete | server/errors.ts | ✅ Integrated | None |
| PR 6 | ✅ Complete | hooks/useIsSegmentActive.ts | ✅ Integrated | None |
| PR 7 | ✅ Complete | components/SkipLink.tsx, nav-tabs.tsx | ✅ Integrated | None |

### Critical Finding

**Header.tsx (297 lines) still uses client-side resolution:**

```typescript
// ❌ Current: features/header/components/Header.tsx
'use client';

export function Header() {
  const pathname = usePathname();

  // Client-side resolution (OLD SYSTEM)
  const contextDescriptor = useMemo(() => {
    return resolveHeaderDescriptor(pathname);  // contextual-nav/config
  }, [pathname]);

  // Client-side features (DUPLICATE)
  const features = useOrganizationFeaturesStore(...);

  // Client-side RBAC (DUPLICATE)
  const canManageWorkflows = useMemo(() => { /* ... */ }, []);

  // Client-side filtering (SHOULD BE SERVER)
  tabs = filterTabsByFeatures(tabs, features);
  if (!canManageWorkflows) {
    tabs = tabs.filter(tab => tab.id !== 'workflows');
  }
}
```

**Meanwhile, the complete server infrastructure sits unused:**
- ✅ [server/descriptor.ts](../services/www_ameide_platform/features/navigation/server/descriptor.ts) - 290 lines of unused code
- ✅ [server/patterns.ts](../services/www_ameide_platform/features/navigation/server/patterns.ts) - URLPattern matching (unused)
- ✅ [server/builders.ts](../services/www_ameide_platform/features/navigation/server/builders.ts) - Tab builders (unused)
- ✅ [server/rbac.ts](../services/www_ameide_platform/features/navigation/server/rbac.ts) - RBAC filtering (duplicated client-side)

### What Needs to Happen

**Single Task: Integrate server descriptor into Header.tsx**

```typescript
// ✅ Target: app/(app)/layout.tsx (Server Component)
import { getNavigationDescriptor, getFallbackDescriptor } from '@/features/navigation/server';
import { HeaderClient } from '@/features/header/components/HeaderClient';

export default async function AppLayout({ children }) {
  const pathname = headers().get('x-pathname') ?? '/';
  const result = await getNavigationDescriptor(pathname);

  const descriptor = result.ok ? result.descriptor : getFallbackDescriptor(pathname);

  return (
    <>
      <HeaderClient descriptor={descriptor} />
      {children}
    </>
  );
}

// ✅ New: features/header/components/HeaderClient.tsx (Client Component)
'use client';

export function HeaderClient({ descriptor }: { descriptor: NavigationDescriptor }) {
  // Purely presentational - no resolution, no RBAC, no filtering
  useHeaderHeightSync();
  useAnnounceNavChange(descriptor.activeTab);

  return (
    <header>
      <ScopeTrail breadcrumbs={descriptor.breadcrumbs} />
      <NavTabs tabs={descriptor.tabs} activeTab={descriptor.activeTab} />
    </header>
  );
}
```

**Estimated Time:** 5 hours
**Impact:**
- ✅ Eliminate 15KB client bundle bloat
- ✅ Close security gap (client-side RBAC bypass)
- ✅ Remove code duplication
- ✅ Complete the refactor

---

Below is a pragmatic refactor plan that keeps what's great about your navigation feature while tightening security, shrinking the client bundle, and improving UX and maintainability. I've organized it as a sequence of focused PRs you can ship independently. Each PR includes the goal, key changes, and concrete code examples.

---

## North‑star architecture (what we’re aiming for)

* **Server‑resolved navigation**: Resolve context + feature flags + RBAC on the **server** and pass a typed `NavigationDescriptor` to the client. Client components become purely presentational.
* **Modular header**: Split `Header.tsx` into small pieces + extract behavior into hooks; code‑split heavy/optional bits (search, notifications).
* **Explicit resolver semantics**: Named patterns, explicit priorities, and observable failures (no silent nulls).
* **Single source of truth for features**: Eliminate duplicate defaults; cache feature flags on the server; show explicit loading states on the client.
* **Accessible by default**: Skip links, focus management on nav changes, and non‑color affordances for the active tab.

---

## PR 1 — Re‑enable RBAC and gate on the server (security) ✅ COMPLETE

**Goal:** Close the security gap and make "who can see what" unambiguous.

**Status:** ✅ **Implemented**
- Created `features/navigation/server/access.ts` with `getUserAccess()` function
- Created `features/navigation/server/rbac.ts` with `applyRBAC()` tab filtering
- Updated `Header.tsx` to remove TODO and implement proper RBAC checks
- Added comprehensive unit tests (100+ test cases)
- Supports global roles (`admin`, `platform-admin`, `workflows-manager`, etc.)
- Supports org-specific roles (`org:{orgId}:{role}`)
- Client-side implementation (server-side enforcement in PR 2)

**Key changes**

1. Introduce a server function that returns `UserAccess` for the current session:

   * Reads session via NextAuth on the server.
   * Checks org/repo membership and roles.
   * Returns typed capabilities (e.g., `{ canManageWorkflows: boolean }`).

2. Enforce capability checks when building navigation **on the server**; tabs that require privileges are either hidden or rendered as disabled with a reason.

3. Add audit hooks for denied access attempts.

**Example (server): `features/navigation/server/access.ts`**

```ts
// server-only file
import { getServerSession } from "next-auth";
import { cache } from "react";

export type UserAccess = {
  canManageWorkflows: boolean;
  canViewGovernance: boolean;
  // ...extend as needed
};

export const getUserAccess = cache(async (orgId?: string, repoId?: string): Promise<UserAccess> => {
  const session = await getServerSession();
  // fetch roles/memberships from your API/DB
  const roles = await fetchRolesFor(session?.user?.id, orgId, repoId);
  return {
    canManageWorkflows: roles.includes("admin") || roles.includes("workflows_manager"),
    canViewGovernance: roles.includes("governance_viewer") || roles.includes("admin"),
  };
});
```

**RBAC in nav building (server):**

```ts
function applyRBAC(tabs: NavTab[], access: UserAccess): NavTab[] {
  return tabs
    .map(t => {
      if (t.id === "workflows" && !access.canManageWorkflows) return { ...t, disabled: true, disabledReason: "Requires Workflow Manager role" };
      if (t.id === "governance" && !access.canViewGovernance) return null;
      return t;
    })
    .filter(Boolean) as NavTab[];
}
```

**Tests to add**

* Unit: workflows tab hidden/disabled based on roles.
* E2E: authenticated user without role cannot access workflows URL (gets 403 or redirected).

---

## PR 2 — Server‑side context resolution & typed descriptor ✅ COMPLETE

**Goal:** Move resolver logic off the client, make behavior explicit and testable, and shrink client work.

**Status:** ✅ **Implemented**
- Created `server/types.ts` with typed NavigationContext and NavigationDescriptor
- Created `server/patterns.ts` using URLPattern for clean route matching
- Created `server/builders.ts` with pure tab/breadcrumb/action builders
- Created `server/descriptor.ts` with main `getNavigationDescriptor()` function
- Created `client/useNavigationDescriptor.ts` hook for client consumption
- Integrated RBAC from PR 1 with feature filtering
- Added 150+ comprehensive unit tests
- Pattern matching with explicit priorities
- Discriminated union for NavigationContext (user | org | repo | transformation | workflows)
- Result type that surfaces resolution errors

**Key changes**

1. Create `navigation-core` (server‑first) with:

   * Explicit `ResolverPriority` and named patterns.
   * Result type that surfaces failure reasons.
   * Server `getNavigationDescriptor(pathname, searchParams)` that returns `{ context, tabs, breadcrumbs, activeTab }`.

2. Use native `URLPattern` or `path-to-regexp` with **named** groups for readability.

3. Expose a thin client hook that consumes the server-provided descriptor.

**Types**

```ts
export type NavigationContext =
  | { kind: "user"; userId: string }
  | { kind: "organization"; orgSlug: string }
  | { kind: "repo"; orgSlug: string; repoSlug: string }
  | { kind: "transformation"; orgSlug: string; transformationId: string }
  | { kind: "workflows"; orgSlug: string; repoSlug?: string };

export type NavigationDescriptor = {
  context: NavigationContext;
  breadcrumbs: { label: string; href: string }[];
  tabs: NavTab[];            // after feature gating + RBAC
  activeTab?: string;
  warnings?: string[];       // e.g., “feature flags loading”, “fallback context”
};
```

**Resolvers with priorities**

```ts
type MatchFn = (url: URL) => null | Record<string,string>;

type ContextResolver = {
  id: string;
  priority: number; // higher wins
  match: MatchFn;
  resolve: (params: Record<string,string>) => Promise<NavigationContext>;
};

const R_REPO = new URLPattern({ pathname: "/org/:orgSlug/repo/:repoSlug/:rest*" });
const R_ORG  = new URLPattern({ pathname: "/org/:orgSlug/:rest*" });
const R_USER = new URLPattern({ pathname: "/u/:userId/:rest*" });

export const CONTEXT_RESOLVERS: ContextResolver[] = [
  {
    id: "repo",
    priority: 100,
    match: (url) => R_REPO.exec(url)?.pathname.groups ?? null,
    resolve: async ({ orgSlug, repoSlug }) => ({ kind: "repo", orgSlug, repoSlug }),
  },
  {
    id: "organization",
    priority: 50,
    match: (url) => R_ORG.exec(url)?.pathname.groups ?? null,
    resolve: async ({ orgSlug }) => ({ kind: "organization", orgSlug }),
  },
  {
    id: "user",
    priority: 10,
    match: (url) => R_USER.exec(url)?.pathname.groups ?? null,
    resolve: async ({ userId }) => ({ kind: "user", userId }),
  },
];

export async function getNavigationDescriptor(urlStr: string): Promise<NavigationDescriptor | ResolveError> {
  const url = new URL(urlStr, "https://example.local"); // base ignored
  const matches = CONTEXT_RESOLVERS
    .map(r => ({ r, params: r.match(url) }))
    .filter(x => x.params);
  if (matches.length === 0) return { error: "NoContextMatched", detail: { url: url.pathname } };

  const { r, params } = matches.sort((a,b) => b.r.priority - a.r.priority)[0];
  const context = await r.r.resolve(params!);

  // feature flags + RBAC (server)
  const features = await getOrgFeatures(context);
  const access   = await getUserAccess(/* ids from context */);
  const tabs     = applyRBAC(filterTabsByFeatures(buildTabs(context), features), access);

  return {
    context,
    breadcrumbs: buildBreadcrumbs(context),
    tabs,
    activeTab: computeActiveTab(context, url),
  };
}
```

**Usage in a layout (RSC):**

```tsx
// app/(app)/layout.tsx
import { getNavigationDescriptor } from "@/features/navigation/server";
export default async function AppLayout({ children }: { children: React.ReactNode }) {
  // derive path from headers or pass pathname down via props in your routing setup
  const descriptor = await getNavigationDescriptor(/* absolute URL string */);
  return (
    <>
      <Header descriptor={descriptor} />
      {children}
    </>
  );
}
```

---

## PR 3 — Split and code‑split `Header.tsx` ✅ COMPLETE

**Goal:** Reduce size/complexity and avoid loading everything on first paint.

**Status:** ✅ **Implemented**
- Created `Header/HeaderLogo.tsx` - Simple logo link component
- Created `Header/HeaderSearch.tsx` - Combined desktop/mobile search button with skeleton
- Created `Header/HeaderActions.tsx` - Notifications, request feature, theme toggle with blink animation
- Created `Header/HeaderUserMenu.tsx` - User dropdown menu with profile/settings/signout
- Created `Header/ScopeTrail.tsx` - Breadcrumb trail for org → transformation → element hierarchy
- Extracted `Header/useHeaderHeightSync.ts` - Hook for syncing header height to CSS variable
- Extracted `Header/useAnnounceNavChange.ts` - Accessibility hook for screen reader announcements
- Refactored main `Header.tsx` to use extracted components with dynamic imports
- Implemented code-splitting with `next/dynamic` for HeaderSearch, HeaderActions, HeaderUserMenu
- Reduced main Header.tsx from ~600 lines to ~320 lines
- All components properly typed and tested

**Key changes**

* Split into:

  * `HeaderLogo.tsx` (pure) ✅
  * `HeaderSearch.tsx` (client, **dynamically imported**) ✅
  * `HeaderActions.tsx` (notifications, create buttons; dynamic import) ✅
  * `HeaderUserMenu.tsx` (client, dynamic) ✅
  * `ScopeTrail.tsx` (receives server breadcrumbs) ✅
* Extract behavior to hooks:

  * `useHeaderHeightSync()` → encapsulates your CSS var sizing + debounced resize. ✅
  * `useAnnounceNavChange()` → focus/ARIA live announcements. ✅

**Example entry**

```tsx
// features/navigation/components/Header/index.tsx
"use client";
import dynamic from "next/dynamic";
import { ScopeTrail } from "./ScopeTrail";
import { NavTabs } from "./NavTabs";
import { useHeaderHeightSync } from "./useHeaderHeightSync";
import { useAnnounceNavChange } from "./useAnnounceNavChange";

const HeaderSearch    = dynamic(() => import("./HeaderSearch"), { ssr: false, loading: () => <SearchSkeleton/> });
const HeaderActions   = dynamic(() => import("./HeaderActions"), { ssr: false });
const HeaderUserMenu  = dynamic(() => import("./HeaderUserMenu"), { ssr: false });

export function Header({ descriptor }: { descriptor: NavigationDescriptor }) {
  useHeaderHeightSync();
  useAnnounceNavChange(descriptor.activeTab);

  return (
    <header role="banner" className="sticky top-0 z-50 border-b bg-background">
      <a href="#main-content" className="sr-only focus:not-sr-only">Skip to content</a>
      <div className="flex items-center justify-between gap-3 p-3">
        <ScopeTrail breadcrumbs={descriptor.breadcrumbs} />
        <HeaderSearch />
        <HeaderActions />
        <HeaderUserMenu />
      </div>
      <nav aria-label="Primary">
        <NavTabs tabs={descriptor.tabs} activeTab={descriptor.activeTab} />
      </nav>
      <div aria-live="polite" aria-atomic="true" className="sr-only" id="nav-announcer" />
    </header>
  );
}
```

---

## PR 4 — Feature flags: unify source, add loading UI, remove duplication ✅ COMPLETE

**Goal:** Single canonical data source; no duplicate `DEFAULT_FEATURES`; visible loading state; optional discovery UX.

**Status:** ✅ **Implemented**
- Created `features/navigation/server/features.ts` with cached `getOrgFeatures()` function
- Integrated `getOrgFeatures()` into `getNavigationDescriptor()` - automatically fetches if not provided
- Removed duplicate `DEFAULT_FEATURES` from Header.tsx - now uses `DEFAULT_ORGANIZATION_FEATURES` from SDK
- Created `TabSkeleton` component with loading states for navigation tabs
- Enhanced `NavTab` type with `disabled` and `disabledReason` properties
- Updated `NavTabs` component to render disabled tabs with tooltips
- Added `disabledMode` prop to `OrgNavigation` ('hide' or 'disable')
- Server-side features are fetched once per request and cached with React `cache()`
- Features fall back to defaults gracefully when org fetch fails

**Key changes**

* `getOrgFeatures` lives on the server; cached via `cache()`; returns typed flags. ✅
* Client shows `<TabSkeleton count={N} />` while hydrating (for client-only cases). ✅
* Optional: surface disabled tabs with a tooltip "Unavailable — ask an admin to enable *Workflows*". ✅

**Filtering stays simple and pure:**

```ts
export function filterTabsByFeatures(tabs: NavTab[], features: OrgFeatures): NavTab[] {
  return tabs.filter(t => {
    if (t.id === "workflows") return !!features.workflows;
    if (t.id === "transformations") return !!features.transformations;
    if (t.id === "governance") return !!features.governance;
    return true; // overview/settings always visible
  });
}
```

---

## PR 5 — Resolver readability & error‑handling upgrades ✅ COMPLETE

**Goal:** Make the patterns and failure modes obvious.

**Status:** ✅ **Implemented**
- Enhanced NavigationResolutionError with 5 specific error types (NO_CONTEXT_MATCHED, RESOLVER_FAILED, PRIORITY_COLLISION, UNAUTHORIZED, INTERNAL_ERROR)
- Each error includes detailed context: pathname, cause, params, stack traces, suggestions
- Created `server/errors.ts` with comprehensive error utilities:
  - Error creation helpers (createNoContextMatchedError, createResolverFailedError, etc.)
  - Structured logging with severity levels (error, warn, info)
  - Human-readable error messages
  - Retryability checks
- Updated descriptor.ts to use error helpers and log with full context
- Made filterTabsByFeatures and applyRBAC generic to work with any tab type
- Created comprehensive ERROR_HANDLING.md documentation with examples and best practices
- All errors provide actionable suggestions for fixing issues
- Automatic severity classification for proper logging

**Key changes**

* Extract regex/URLPattern constants into `PATTERNS`. ✅
* Use `Result` type for resolution: `{ ok: true; value } | { ok: false; error }`. ✅
* Add a `ResolutionDiagnostics` object you can log in development/tests. ✅
* Add telemetry on "no context matched" or "priority collision". ✅

**Example**

```ts
export type ResolveError =
  | { error: "NoContextMatched"; detail: { url: string } }
  | { error: "PriorityCollision"; detail: { candidates: string[] } }
  | { error: "ResolverFailed"; detail: { id: string; cause?: string } };
```

---

## PR 6 — Active state: nested segment detection & URL builder ✅ COMPLETE

**Goal:** Fix the TODO so the Workflows tab activates reliably and centralize URL creation.

**Status:** ✅ **Implemented**
- Created `hooks/useIsSegmentActive.ts` with comprehensive active segment detection:
  - `useIsSegmentActive()` - Check if segment is active at any depth
  - `useIsSegmentActiveAtDepth()` - Check segment at specific depth
  - `useActiveSegments()` - Get all active segments
  - `isRouteWithinBase()` - Check if pathname is within a base path
- Created `utils/buildNavHref.ts` - Centralized URL builder:
  - `buildNavHref()` - Build URLs for all navigation contexts (user, org, repo, transformation, workflows)
  - `parsePathname()` - Parse pathname into segments
  - `matchesPattern()` - Pattern matching with wildcards
- All URL building logic centralized in one place
- Consistent URL generation across server and client
- Existing `computeActiveTab()` in builders.ts handles server-side active detection

**Key changes**

* Implement a small `useIsSegmentActive(target: string)` using `useSelectedLayoutSegments()` and a depth check. ✅
* Centralize URLs in `buildNavHref(context, tabId)`. ✅

**Example**

```ts
// client util
import { useSelectedLayoutSegments } from "next/navigation";
export function useIsSegmentActive(target: string): boolean {
  const segments = useSelectedLayoutSegments(); // ['repo','workflows','runs']
  return segments.includes(target);
}

// server + client shared
export function buildNavHref(ctx: NavigationContext, tabId: string): string {
  switch (ctx.kind) {
    case "repo":
      return `/org/${ctx.orgSlug}/repo/${ctx.repoSlug}/${tabId === "overview" ? "" : tabId}`;
    case "organization":
      return `/org/${ctx.orgSlug}/${tabId === "overview" ? "" : tabId}`;
    // ...
  }
}
```

---

## PR 7 — Accessibility & UX polish ✅ COMPLETE

**Goal:** Improve keyboard and screen reader experience.

**Status:** ✅ **Implemented**
- Created `components/SkipLink.tsx` - Keyboard navigation skip link:
  - Appears on focus to allow skipping navigation
  - WCAG 2.1 Level A compliant
  - Includes `SkipLinkTarget` wrapper for main content
- Updated `nav-tabs.tsx` with comprehensive accessibility:
  - Added `aria-current="page"` to active tabs
  - Added visible focus rings with `focus-visible:ring-2`
  - Added underline indicator (non-color) to active tabs using CSS `after` pseudo-element
  - Changed wrapper to semantic `<nav>` with `aria-label="Primary navigation"`
  - Added `role="list"` for tab container
  - Improved contrast and hover states
- Integrated SkipLink into Header.tsx
- Enhanced disabled tab accessibility with `aria-disabled`
- Screen reader announcements already implemented in PR 3 (useAnnounceNavChange)

**Key changes**

* **Skip link** (added in PR 3). ✅
* **Non‑color indicators**: underline or marker for active tab; `aria-current="page"` on the active item. ✅
* **Focus management**: On tab change, programmatically announce via the `aria-live` region and optionally move focus to the page heading. ✅
* **Contrast & states**: Ensure tab hover/active/focus visible with outline. ✅

**NavTabs snippet**

```tsx
<li key={t.id}>
  <Link
    href={t.href}
    aria-current={active ? "page" : undefined}
    aria-disabled={t.disabled || undefined}
    onClick={t.disabled ? (e) => e.preventDefault() : undefined}
    className={cn(
      "rounded-md px-3 py-2 focus:outline focus:outline-2",
      active && "underline underline-offset-4",
      t.disabled && "opacity-50 cursor-not-allowed"
    )}
    title={t.disabledReason}
  >
    {t.label}
  </Link>
</li>
```

---

## Cross‑cutting improvements (apply opportunistically)

### Performance

* **Server-first**: With descriptor built on the server, client work is small.
* **Code splitting**: `dynamic()` for search, notifications, user menu; keep base nav lean.
* **Memoization**: Remove client recomputation of `fallbackNavigation`; compute once on the server.
* **Caching**: `cache()` for org, repo, features; add sane revalidation intervals.

### Error boundaries

* Wrap navigation section with an RSC boundary (Next `error.js`) and client fallback:

  ```tsx
  <NavigationErrorBoundary fallback={<MinimalNav />}>
    <ConditionalOrgNavigation />
  </NavigationErrorBoundary>
  ```

  On the RSC side, prefer `app/(app)/navigation/error.tsx` to catch server failures.

### Documentation

* Add an ADR: “Why a server‑resolved navigation descriptor?”
* Add a README example: “How to add a new context” and “How to add a new tab gated by a feature.”

### Analytics

* `useNavAnalytics()` hook that fires events on tab click + context change:

  * `{ event: "nav_click", tabId, contextKind, orgSlug, repoSlug }`
  * Useful to validate UX and detect dead tabs.

---

## Tests you should add (or expand)

* **Resolvers**

  * Priority dominance when multiple match; collision detection.
  * URLPattern route coverage for edge paths (`/settings`, nested paths).
* **RBAC**

  * Hidden/disabled tabs by role; direct URL access returns 403 or redirect.
* **Features**

  * Flags off → tabs absent/disabled with `title` reason; flags on → visible.
* **Accessibility**

  * Playwright + axe: no violations on header/nav.
  * Keyboard E2E: Tab order, Enter/Space activation, skip link works.
* **Performance**

  * Header height sync: throttle behavior verified (unit or small integration test).
* **Resilience**

  * Network failure fetching features/roles → fallback MinimalNav with warning banner.

---

## File structure (after refactor)

```
features/navigation/
  server/
    access.ts
    descriptor.ts
    patterns.ts
    features.ts      // getOrgFeatures()
  components/Header/
    index.tsx
    HeaderLogo.tsx
    HeaderSearch.tsx
    HeaderActions.tsx
    HeaderUserMenu.tsx
    ScopeTrail.tsx
    NavTabs.tsx
    TabSkeleton.tsx
    useHeaderHeightSync.ts
    useAnnounceNavChange.ts
  contextual-nav/
    buildTabs.ts
    filterTabs.ts
    breadcrumbs.ts
    linkBuilder.ts
  errors/
    NavigationErrorBoundary.tsx
  tests/
    descriptor.test.ts
    rbac.test.ts
    features.test.ts
    a11y-header.spec.ts
    navigation-flows.spec.ts     // keep + extend
```

---

## Addressing each weakness from your review (quick map)

* **Regex maintenance / no priority / silent failures** → PR 2 & 5 (URLPattern + priority + `ResolveError`).
* **Header too large / coupling** → PR 3 (split + dynamic imports + hooks).
* **Feature flag defaults / flicker** → PR 4 (server source + skeleton).
* **Race conditions (org context after render)** → PR 2 (resolve context server‑side).
* **Client bundle size / unnecessary re-renders** → PR 2 & 3 (server descriptor + code splitting).
* **Nested segment detection TODO** → PR 6 (`useSelectedLayoutSegments()` helper).
* **Accessibility gaps** → PR 7 (skip link, focus mgmt, non-color cues).
* **Security: disabled RBAC / feature bypass** → PR 1 (server gating + audit).
* **Docs & ADRs** → cross‑cutting docs improvements.

---

## A few targeted code extras

**Announce nav changes (focus management)**

```ts
// useAnnounceNavChange.ts
"use client";
import { useEffect } from "react";

export function useAnnounceNavChange(activeTab?: string) {
  useEffect(() => {
    const el = document.getElementById("nav-announcer");
    if (!el || !activeTab) return;
    el.textContent = `Navigation changed. Active tab: ${activeTab}.`;
  }, [activeTab]);
}
```

**Header height sync (extracted hook)**

```ts
// useHeaderHeightSync.ts
"use client";
import { useEffect } from "react";

export function useHeaderHeightSync() {
  useEffect(() => {
    const root = document.documentElement;
    let raf = 0;
    const update = () => {
      const h = document.querySelector("header")?.getBoundingClientRect().height ?? 0;
      root.style.setProperty("--header-h", `${Math.round(h)}px`);
    };
    const onResize = () => {
      cancelAnimationFrame(raf);
      raf = requestAnimationFrame(update);
    };
    update();
    window.addEventListener("resize", onResize);
    return () => window.removeEventListener("resize", onResize);
  }, []);
}
```

**Active tab detection**

```ts
export function computeActiveTab(ctx: NavigationContext, url: URL): string | undefined {
  const segs = url.pathname.split("/").filter(Boolean);
  // Example heuristic: last meaningful segment if it matches a tab id
  const candidates = new Set(["overview","graph","workflows","transformations","governance","settings","insights"]);
  const hit = segs.findLast(s => candidates.has(s));
  return hit ?? "overview";
}
```

---

## Rollout & risk management

* Ship PR 1 (RBAC) first to close the security gap quickly.
* Land PR 2 behind a short‑lived feature flag `serverResolvedNav`. Keep the current client resolver as a fallback for one release.
* Monitor: client bundle size, TTFB, hydration warnings, and a11y checks.
* If any regression, switch the flag off (fallback remains functionally equivalent).

---

### TL;DR

* **Move resolution + gating to the server**, return a typed descriptor.
* **Split/code‑split the header** and extract shared behavior to hooks.
* **Make resolvers explicit** (priorities, URLPattern, no silent nulls).
* **Unify feature flags** with server caching and show loading skeletons.
* **Fix nested active state**, add **skip link** and **focus announcements**.

This refactor keeps your A‑grade design, but closes the RBAC hole, trims the client bundle, and makes the system easier to read, test, and extend. If you want, I can turn any section above into ready‑to‑paste files tailored to your exact folder paths and types.

---

## 🚨 CURRENT IMPLEMENTATION GAPS (2025-10-30 Review)

### Gap 1: Server Descriptor Not Integrated ❌ CRITICAL

**Location:** [features/header/components/Header.tsx:74-81](../services/www_ameide_platform/features/header/components/Header.tsx#L74-L81)

**Problem:** Header.tsx uses client-side `resolveHeaderDescriptor()` instead of server `getNavigationDescriptor()`

**Current Code:**
```typescript
// features/header/components/Header.tsx
import { resolveHeaderDescriptor } from '@/features/navigation/contextual-nav';  // ❌ Client-side

const contextDescriptor = useMemo(() => {
  return resolveHeaderDescriptor(pathname);  // ❌ Client resolution
}, [pathname]);
```

**Should Be:**
```typescript
// app/(app)/layout.tsx (Server Component)
import { getNavigationDescriptor } from '@/features/navigation/server';  // ✅ Server-side

export default async function AppLayout({ children }) {
  const pathname = headers().get('x-pathname') ?? '/';
  const result = await getNavigationDescriptor(pathname);
  const descriptor = result.ok ? result.descriptor : getFallbackDescriptor(pathname);

  return <HeaderClient descriptor={descriptor} />;
}
```

**Impact:**
- 🔴 Security: Client-side resolution can be manipulated
- 🔴 Performance: ~15KB unnecessary client bundle
- 🔴 Maintenance: Two parallel systems (server/descriptor.ts + contextual-nav/config.ts)

**Estimated Fix:** 5 hours

---

### Gap 2: Client-Side RBAC Duplication ⚠️ HIGH

**Location:** [features/header/components/Header.tsx:108-135](../services/www_ameide_platform/features/header/components/Header.tsx#L108-L135)

**Problem:** RBAC logic duplicated in Header.tsx, duplicating server/access.ts and server/rbac.ts

**Current Code:**
```typescript
// ❌ Client-side RBAC (duplicates server logic)
const canManageWorkflows = useMemo(() => {
  const user = session?.user;
  if (!user) return false;
  if (isAdmin(user)) return true;
  if (hasAnyRole(user, ['platform-admin', 'platform-operator', 'workflows-manager'])) {
    return true;
  }
  if (orgId && user.roles) {
    const hasOrgRole = user.roles.some((role) => {
      return role.toLowerCase() === `org:${orgId}:workflows-manager`;
    });
    if (hasOrgRole) return true;
  }
  return false;
}, [session?.user, orgId]);

// Later: manual filtering
if (!canManageWorkflows) {
  tabs = tabs.filter((tab) => tab.id !== 'workflows');
}
```

**Should Be:**
```typescript
// ✅ Server handles all RBAC
// Header receives pre-filtered tabs from descriptor
export function HeaderClient({ descriptor }: { descriptor: NavigationDescriptor }) {
  // descriptor.tabs already filtered by server/rbac.ts
  return <NavTabs tabs={descriptor.tabs} />;
}
```

**Impact:**
- 🔴 Security: Client bypass possible
- 🟡 Maintenance: Must update RBAC logic in two places
- 🟡 Consistency: Server and client can drift

**Estimated Fix:** 30 minutes (part of Gap 1 fix)

---

### Gap 3: Client-Side Feature Flags ⚠️ MEDIUM

**Location:** [features/header/components/Header.tsx:94-105](../services/www_ameide_platform/features/header/components/Header.tsx#L94-L105)

**Problem:** Features read from Zustand store client-side instead of server descriptor

**Current Code:**
```typescript
// ❌ Client-side features
const features = useOrganizationFeaturesStore(
  useCallback((state) => {
    if (!orgId) return DEFAULT_ORGANIZATION_FEATURES;
    return state.features[orgId] ?? DEFAULT_ORGANIZATION_FEATURES;
  }, [orgId])
);

// ❌ Client-side filtering
tabs = filterTabsByFeatures(tabs, features);
```

**Should Be:**
```typescript
// ✅ Server handles feature filtering
// server/descriptor.ts already applies filterTabsByFeatures()
const resolvedFeatures = features ?? (orgId ? await getOrgFeatures(orgId) : undefined);
if (resolvedFeatures && orgId) {
  tabs = filterTabsByFeatures(tabs, resolvedFeatures);
}
```

**Impact:**
- 🟡 Security: Feature flags can be manipulated client-side
- 🟡 Consistency: Race condition between server and Zustand store
- 🟡 Performance: Unnecessary Zustand dependency

**Estimated Fix:** 30 minutes (part of Gap 1 fix)

---

### Gap 4: Dual Type Systems 🟡 MEDIUM

**Problem:** Incompatible types between server and client navigation

**Current State:**
```typescript
// server/types.ts
export type NavigationTab = {
  id: string;
  label: string;
  href: string;
  icon?: NavigationIcon;  // ✅ Uses NavigationIcon
  disabled?: boolean;
  disabledReason?: string;
  badge?: ReactNode;
  activeMatch?: RegExp;
};

// contextual-nav/config.ts
export type HeaderTab = {
  id: string;
  label: string;
  href: string;
  icon?: HeaderIcon;  // ❌ Different: HeaderIcon
  badge?: ReactNode;
  activeMatch?: RegExp;
  // ❌ Missing: disabled, disabledReason
};
```

**Impact:**
- 🟡 Type confusion: Developers unsure which type to use
- 🟡 Code sharing impossible between systems
- 🟡 Type casting required at boundaries

**Recommendation:**
1. Unify on `NavigationTab` as the canonical type
2. Mark `HeaderTab` as deprecated
3. Add type migration guide

**Estimated Fix:** 3 hours

---

### Gap 5: Missing Integration Tests 🟡 MEDIUM

**Problem:** No tests verify server descriptor → Header integration

**Current Test Coverage:**
- ✅ Server infrastructure (patterns, RBAC, errors): 250+ tests
- ✅ Accessibility: E2E tests exist
- ✅ Active state: Integration tests exist
- ❌ **Header.tsx integration: 0 tests**
- ❌ **Server→Client flow: 0 tests**
- ❌ **Security (RBAC bypass): 0 tests**

**Missing Tests:**
1. Header receives and renders server descriptor correctly
2. RBAC: Server-filtered tabs cannot be bypassed client-side
3. Features: Server-filtered tabs match feature flags
4. Direct URL access: Restricted routes return 403
5. Bundle size: Header stays under 30KB budget

**Estimated Fix:** 4 hours

---

## 📋 Recommended Action Plan

### Phase 1: Critical Integration (Sprint 1, Week 1)

**Total Time: 7 hours**

1. **Add pathname propagation middleware** (1 hour)
   ```typescript
   // middleware.ts
   export function middleware(request: NextRequest) {
     const response = NextResponse.next();
     response.headers.set('x-pathname', request.nextUrl.pathname);
     return response;
   }
   ```

2. **Create HeaderClient component** (2 hours)
   - Copy Header.tsx → HeaderClient.tsx
   - Add `descriptor: NavigationDescriptor` prop
   - Remove all resolution logic (lines 74-250)
   - Update type imports to use NavigationTab

3. **Update app layout** (1 hour)
   - Make layout async RSC
   - Call `getNavigationDescriptor(pathname)`
   - Pass descriptor to HeaderClient

4. **Test integration** (3 hours)
   - Manual testing: all routes render correct tabs
   - RBAC: workflows tab only shows for authorized users
   - Features: disabled features hide tabs
   - Performance: verify bundle size reduction

### Phase 2: Cleanup & Hardening (Sprint 1, Week 2)

**Total Time: 6 hours**

5. **Remove client-side duplication** (2 hours)
   - Delete RBAC logic from HeaderClient
   - Remove Zustand feature store dependency
   - Mark `resolveHeaderDescriptor()` deprecated

6. **Add integration tests** (4 hours)
   - Test server descriptor → HeaderClient flow
   - Test RBAC enforcement
   - Add bundle size tracking

### Phase 3: Type Unification (Sprint 2)

**Total Time: 3 hours**

7. **Unify type system** (3 hours)
   - Migrate all code to use `NavigationTab`
   - Deprecate `HeaderTab`
   - Update documentation

### Phase 4: Documentation (Sprint 2)

**Total Time: 3 hours**

8. **Write ADR and guides** (3 hours)
   - ADR: "Why Server-Resolved Navigation?"
   - Guide: "How to Add a New Tab"
   - Update navigation/README.md

---

## 🎯 Success Criteria

### Must Have (Phase 1)

- [ ] Header.tsx uses server descriptor (not client resolution)
- [ ] No client-side RBAC logic in Header
- [ ] No client-side feature filtering in Header
- [ ] Client bundle reduced by 10-15KB
- [ ] All existing E2E tests pass

### Should Have (Phase 2)

- [ ] Integration tests covering server → client flow
- [ ] Security tests preventing RBAC bypass
- [ ] Bundle size budget enforced in CI
- [ ] contextual-nav/config.ts marked deprecated

### Nice to Have (Phase 3-4)

- [ ] Unified type system (NavigationTab everywhere)
- [ ] ADR and documentation complete
- [ ] Analytics integration (useNavAnalytics hook)
- [ ] Error boundary with graceful degradation

---

## 📊 Final Grade

**Implementation Quality:** A+ (excellent code, tests, types)
**Architecture Design:** A (clean separation, well-designed)
**Integration:** D (critical path not completed)
**Documentation:** B (good but needs ADR)

**Overall:** B+ (great work that needs final push to finish line)

### Bottom Line

The foundation is solid. The house is built. The plumbing works. The electricity works. Everything is ready. Just need to **flip the switch** (connect Header.tsx to server descriptor).

**Estimated time to "done-done": 13 hours over 2 sprints**
