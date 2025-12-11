# 325 - AMEIDE Layout & Routing (v2)

**Purpose**: Provide a concise, decision-ready view of the current layout system, routing model, and open work for the AMEIDE web app.  
**Audience**: Product, UX, and engineering leads working on navigation, layout patterns, and related roadmap.  
**Last audited**: 2025-10-30 (matches source documentation)

---

## 1. Scope & Objectives
- Unify how authenticated and unauthenticated shells, shared providers, and navigation components fit together.
- Track the five standardized page patterns and their coverage across the page inventory.
- Capture routing conventions (route groups, parallel routes, modal intercepts) in one place.
- Highlight outstanding migrations, UX parity gaps, and success metrics that define “done”.
- Out of scope: visual design specs, detailed wireframes for each feature flow, and non-web clients.

---

## 2. Platform Architecture Snapshot (Current)

### 2.1 Design System Foundations
- **Libraries**: shadcn/ui primitives on top of Radix UI, styled with Tailwind CSS tokens (`--background`, `--primary`, etc.).
- **Component sources**: `components/ui/*` for primitives; domain compositions live under `features/*/components`.
- **Principles**: server-first data access, composition over configuration, accessibility-first interaction, and progressive enhancement (core content works without JS; heavy features load dynamically).

### 2.2 Root Shell
- **File**: `services/www_ameide_platform/app/layout.tsx`
- **Providers**: `ThemeProvider` → `ToastProvider` → `Providers` (session + config). Global CSS lives in `app/globals.css`.
- **Behaviors**: Syncs theme colors in `<head>`, boots Plausible analytics, ensures focus management via skip links.

### 2.3 Route Partitions
- **Next.js route groups**: `(auth)` for unauthenticated flows, `(app)` for the full application shell.
- `(auth)` layout → centered card for `/login` and `/register` (Keycloak hand-off).
- `(app)` layout → wraps every authenticated page with the header chrome, threads container, and shared data streams.

### 2.4 Authenticated Frame
- **Provider stack**: `DebugProvider` → `TooltipProvider` → `SearchProvider` → `DataStreamProvider` → `ChatProvider`.
- **Structure**: fixed header (`HeaderClient`), spacer div for layout offset, main content in `#main-content`, threads footer pinned to the bottom.
- **Accessibility**: skip link to main content, ARIA live region for header updates, keyboard-ready command palette (⌘/Ctrl+K).

---

## 3. Navigation Model
- **Descriptor resolution**: performed server-side in `app/(app)/layout.tsx`, applying RBAC and feature flags before rendering `HeaderClient`.
- **Header composition**:
  - Top row: logo, scope trail, centered search, actions (notifications dropdown, user menu).
  - Bottom row: tabbed navigation (`NavTabs`) derived from the descriptor; tab state follows the active route.
- **Notifications**: `NotificationsDropdown` provides unread badge, per-org filters, and link into `/inbox`. Real-time updates announce through ARIA live regions.
- **Command palette**: MVP state supports navigation and search categories; roadmap includes action execution (P1).
- **Chat layout**: `ChatLayoutWrapper` manages threads docking; when active, secondary panels (e.g., list activity panes) respond to available width.

---

## 4. Layout Pattern System

| Pattern | Primary Routes | Key Elements | Notes |
|---------|----------------|--------------|-------|
| Dashboard | `/org/[orgId]`, transformation overview | `DashboardLayout`, widget grid (`react-grid-layout`), configurable cards | Supports responsive reflow; badges exposed via `PageHeader`. |
| List | `/org/[orgId]/transformations`, `/repo/*` lists | `ListPageLayout`, optional activity panel tied to threads width | Standard filters today; typed qualifiers planned (P2). |
| Settings | `/org/[orgId]/settings`, `/repo/[graphId]/settings` | `SettingsLayout` with section nav and sticky sidebar | Hides sidebar when threads panel is detached. |
| Editor | (reserved for full-screen modeling tools) | `EditorLayout` scaffold, tool palette, canvas, properties panel | Element editors currently stay modal; full-screen editor is a Phase 6 option. |
| Data Shell | Initiative sub-pages | `InitiativeSectionShell` preloads transformation + workspace context | Used across 15+ transformation sub-sections. |
| Placeholder | `/org/[orgId]/reports`, governance | `PlaceholderLayout` with metadata badges and action slots | Acts as a holding pattern for future feature areas. |

**Coverage (2025-10-30)**:
- 50 tracked pages. 29 pages (≈58%) consume standardized layouts per adoption audit; older checklist still states 25/50. Reconcile counts during the next audit.
- High-priority custom hold-out: `user/profile` (445-line custom two-column layout).
- Low-priority custom areas: element detail modals, workflows admin flows (1–2 day migrations each).

---

## 5. Context Layouts & Data Flow

| Context | File | Responsibilities | Error Modes |
|---------|------|------------------|-------------|
| Organization | `app/(app)/org/[orgId]/layout.tsx` | Server-fetch organization (tenant-aware), wrap children with `OrgContextProvider`. | 404 fallback for missing org; generic fallback for other failures. |
| Initiative | `.../org/[orgId]/transformations/[transformationId]/layout.tsx` | Resolves transformation, workspace, and navigation tabs; feeds `InitiativeSectionShell`. | Missing transformation → not found state; soft-fails workspace data. |
| Repository | `.../org/[orgId]/repo/[graphId]/layout.tsx` | Loads graph metadata, element tree, permission scopes. | Surfaces guard rails before rendering editors or lists. |
| User | `app/(app)/user/layout.tsx` | Provides user session context, handles profile/settings routes. | Auth-required; falls back to session refresh. |

Shared hooks (`useActiveOrganization`, `useActiveInitiative`, etc.) expose the loaded context to leaf pages without repeating fetches.

---

## 6. Routing Overview

### 6.1 Parameter Hierarchy
```
/inbox
/org/[orgId]
  /inbox
  /transformations/[transformationId]
    /inbox
    /architect/capabilities
  /repo/[graphId]
    /inbox
    /element/[elementId]
    /@modal/(.)element/[elementId]
```

### 6.2 Patterns to Observe
- **Route groups**: `(auth)` and `(app)` define shell boundaries.
- **Parallel routes**: modal intercepts live under `@modal` to let element editors appear as overlays without leaving the list page.
- **Dynamic segments**: `[orgId]`, `[transformationId]`, `[graphId]`, `[elementId]`, `[workflowsId]`, `[executionId]`.
- **Layout mapping**: global routes → Root → App; org routes → Root → App → Org; transformation routes add the transformation shell; repositories mirror that pattern.

---

## 7. Page Inventory & Status
- **Standardized layouts (≈58%)**: Dashboard (Org + Initiative overview), five list pages (graph, transformations, graph detail, user management, teams), four core settings pages, placeholder areas, 15+ transformation data pages.
- **Custom/high-priority**: `user/profile` remains bespoke; decide whether to preserve or migrate to DashboardLayout. Impact low but simplifies maintenance.
- **Custom/low-priority**: workflows pages and element modals perform well as-is; evaluate when the editor roadmap activates.
- **Special/auth pages**: root redirect, onboarding, invitation accept flows sit outside the layout migration scope.

---

## 8. Near-Term Backlog

**P0**
1. Reconcile page coverage counts (25 vs 29 of 50) and complete `user/profile` layout decision.
2. Finish documentation hand-off (`docs/PAGE_PATTERNS.md`) and ensure Storybook stories exist for all five patterns.

**P1**
1. Extend command palette with action execution (beyond navigation/search).
2. Ship discoverable keyboard shortcut overlay (`?`) with global + page-specific shortcuts.
3. Consolidate empty states into a reusable component suite aligned with Primer blankslate patterns.
4. Complete accessibility + performance audit pass (WCAG AA checks, keyboard regression suite, dashboard/list/editor perf benchmarks).

**P2**
1. Upgrade list filtering with typed qualifiers and autosuggest, ensuring URLs remain shareable.
2. Promote notifications into a dual-inbox experience (triage + saved) with reason codes.
3. Add collaboration affordances (mentions, autolinks, reactions, saved replies) across editors and discussions.

**P3**
1. Explore full-screen editor adoption once element modals prove limiting.
2. Evaluate threads detach improvements for narrow viewports (mobile overlay or split screen).

---

## 9. Metrics & Definition of Done

| Metric | Current | Target | Notes |
|--------|---------|--------|-------|
| Org settings LOC | 2,368 → 665 | ≈150 | Already cut by ~72%; final refactor pending. |
| Dashboard configurability | Static | User-customizable | Requires widget edit/persist work. |
| Repository browser layout | 3-column | 2-column | Awaiting UX validation; part of list pattern evolution. |
| Pattern coverage | Ad-hoc | 5 standard patterns | Track via adoption audit; close remaining pages. |
| Component reuse | ~30% → ~60% | ≈80% | Settings module shows improvement; expand to other surfaces. |
| Page build time | ≈4 h | ≈1 h | Measure once migration checklist is complete. |

Successful delivery means the coverage metric is met, keyboard and command features reach parity with GitHub benchmarks, and audits confirm accessibility + performance baselines.

---

## 10. References
- Root layout: `services/www_ameide_platform/app/layout.tsx`
- Auth layout: `app/(auth)/layout.tsx`
- App chrome: `app/(app)/layout.tsx`, `features/header/components/HeaderClient.tsx`
- Organization shell: `app/(app)/org/[orgId]/layout.tsx`
- Layout components: `features/common/components/layouts/*`
- Navigation utilities: `features/navigation/*`, `components/ui/*`
- Context hooks: `features/common/hooks/useActiveOrganization.ts`, etc.

_Discrepancies to verify_: the pattern adoption count (25 vs 29 pages) and whether `EditorLayout` remains part of the active component library despite modal-first editors. Flag during the next audit.

