# 317: Route-Based Navigation Implementation

**Status**: ✅ Complete
**Priority**: High
**Type**: Technical Debt / Refactoring
**Date**: 2025-01-29

## Overview

Migrated navigation system from regex-based pathname matching to Next.js layout-based route navigation, following Next.js 14+ best practices. Navigation tabs now use `useSelectedLayoutSegment()` instead of manual pathname parsing.

## Problem Statement

### Original Implementation Issues

1. **Regex-based route matching** in `features/navigation/contextual-nav/config.ts`
   - Complex regex patterns prone to errors
   - Easy to forget updating when adding routes
   - Route logic duplicated (file system + regex)
   - Not maintainable or scalable

2. **Centralized navigation rendering** in Header component
   - All navigation logic in one place
   - Hard to co-locate with relevant routes
   - Client-side pathname parsing

3. **Missing route contexts**
   - Repository settings routes didn't match properly
   - Required constant regex pattern updates

### Example of Old Approach
```typescript
// features/navigation/contextual-nav/config.ts
{
  id: 'repo',
  matchers: [
    /^\/org\/[^/]+\/repo\/[^/]+\/?$/,
    /^\/org\/[^/]+\/repo\/[^/]+\/settings(?:\/.*)?$/,  // Constantly adding more!
  ],
  resolve: (ctx) => {
    // Manual pathname parsing
    const tabs = [...];
    return { tabs, breadcrumbs, scope };
  }
}
```

## Solution: Layout-Based Navigation

### Architecture

```
Route Structure                           Navigation Component
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
/org/[orgId]/layout.tsx                → OrgNavigation
/org/[orgId]/repo/[graphId]/      → RepoNavigation
/org/[orgId]/transformations/[id]/         → InitiativeNavigation
```

### Implementation Details

#### 1. Created Reusable Navigation Components

**File**: `components/ui/nav-tabs.tsx`
```typescript
export function NavTabs({ tabs }: NavTabsProps) {
  return (
    <div className="w-full bg-background border-b border-border">
      <div className="flex w-full items-center gap-1.5 px-4 pt-1.5 pb-2 lg:px-6">
        {tabs.map(tab => (
          <Link href={tab.href} className={tab.active ? 'bg-primary/10' : ''}>
            {tab.label}
          </Link>
        ))}
      </div>
    </div>
  );
}
```

**Files**: Context-specific navigation components
- `features/navigation/org-navigation.tsx`
- `features/navigation/repo-navigation.tsx`
- `features/navigation/transformation-navigation.tsx`

Each uses `useSelectedLayoutSegment()` to determine active tab:
```typescript
export function RepoNavigation({ orgId, graphId }: Props) {
  const segment = useSelectedLayoutSegment();
  // segment = null | 'settings' | 'element' | ...

  const tabs = [
    { href: base, label: 'Elements', active: segment === null },
    { href: `${base}/settings`, label: 'Settings', active: segment === 'settings' },
  ];

  return <NavTabs tabs={tabs} />;
}
```

#### 2. Updated Layouts to Render Navigation

**Organization Layout** (`app/(app)/org/[orgId]/layout.tsx`)
```typescript
export default async function OrganizationLayout({ children, params }) {
  const { orgId } = await params;
  const organization = await getOrganization(orgId);

  return (
    <OrgContextProvider organization={organization}>
      <ConditionalOrgNavigation orgId={orgId} features={organization.features} />
      {children}
    </OrgContextProvider>
  );
}
```

**Repository Layout** (`app/(app)/org/[orgId]/repo/[graphId]/layout.tsx`)
```typescript
export default async function RepositoryLayout({ children, params }) {
  const { orgId, graphId } = await params;

  return (
    <>
      <RepoNavigation orgId={orgId} graphId={graphId} />
      <div className="mx-auto w-full max-w-7xl px-4 py-6 sm:px-6 lg:px-8">
        {children}
      </div>
    </>
  );
}
```

**Initiative Layout** (`app/(app)/org/[orgId]/transformations/[transformationId]/layout.tsx`)
```typescript
export default async function InitiativeLayout({ children, params }) {
  const { orgId, transformationId } = await params;

  return (
    <>
      <InitiativeNavigation orgId={orgId} transformationId={transformationId} />
      <div className="mx-auto w-full max-w-7xl px-4 py-6 sm:px-6 lg:px-8">
        {children}
      </div>
    </>
  );
}
```

#### 3. Conditional Navigation Rendering

Created `ConditionalOrgNavigation` to prevent duplicate navigation bars when on nested routes:

```typescript
export function ConditionalOrgNavigation(props: OrgNavigationProps) {
  const segment = useSelectedLayoutSegment();

  // Hide org navigation when on repo or transformations routes
  const isNestedRoute = segment === 'repo' || segment === 'transformations';

  if (isNestedRoute) return null;

  return <OrgNavigation {...props} />;
}
```

#### 4. Visual Layout Structure

Ensured navigation tabs are full-width (like header):

```
┌──────────────────────────────────────────────┐
│  HEADER (full width)                         │
│  [Logo] [Search] [User]                      │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│  NAVIGATION TABS (full width) ✅              │
│  [Elements] [Workflows] [Settings]           │
└──────────────────────────────────────────────┘
  ┌────────────────────────────────┐
  │  PAGE CONTENT (constrained)    │
  │  max-w-7xl                     │
  └────────────────────────────────┘
```

Removed `max-w-7xl` constraint from org layout, added it to:
- Individual org pages
- Nested layout children wrappers

#### 5. Removed Header Tab Rendering

**File**: `features/navigation/components/Header.tsx`

Removed ~70 lines of tab rendering code (lines 451-517). Header now only renders:
- Logo
- User menu
- Notifications
- Search

## Benefits

### ✅ Co-located Navigation
Navigation logic lives next to the routes it serves, not in a central config file.

### ✅ No Regex Maintenance
Adding a new route = adding a new tab. No regex patterns to update.

### ✅ Type-safe Params
TypeScript knows route params from the file structure automatically.

### ✅ Server Components
Layouts can fetch data server-side before rendering (e.g., graph name, organization features).

### ✅ Built-in Hooks
`useSelectedLayoutSegment()` tells us which route is active—no pathname parsing needed.

### ✅ Single Source of Truth
The file system IS the routing logic. No duplication between routes and navigation config.

### ✅ Better Performance
Less client-side JavaScript, navigation is part of initial server-rendered HTML.

## Testing

Verified navigation works correctly on:

| Route | Navigation Context | Active Tab |
|-------|-------------------|------------|
| `/org/atlas` | Organization | Overview |
| `/org/atlas/settings` | Organization | Settings |
| `/org/atlas/repo/primary-architecture` | Repository | Elements |
| `/org/atlas/repo/primary-architecture/settings` | Repository | Settings |
| `/org/atlas/transformations/digital-transformation` | Initiative | Overview |
| `/org/atlas/transformations/digital-transformation/settings` | Initiative | Settings |

All navigation appears full-width below the header as expected.

## Files Changed

### Created
- `components/ui/nav-tabs.tsx` - Reusable tab navigation component
- `features/navigation/org-navigation.tsx` - Organization navigation with `useSelectedLayoutSegment()`
- `features/navigation/repo-navigation.tsx` - Repository navigation
- `features/navigation/transformation-navigation.tsx` - Initiative navigation
- `features/navigation/conditional-org-navigation.tsx` - Prevents duplicate navigation

### Modified
- `app/(app)/org/[orgId]/layout.tsx` - Renders conditional org navigation, removed max-w constraint
- `app/(app)/org/[orgId]/page.tsx` - Added content wrapper
- `app/(app)/org/[orgId]/settings/page.tsx` - Added content wrapper, changed default section to 'features'
- `app/(app)/org/[orgId]/transformations/page.tsx` - Added content wrapper (list page)
- `app/(app)/org/[orgId]/repo/page.tsx` - Added content wrapper (list page)
- `app/(app)/org/[orgId]/repo/[graphId]/layout.tsx` - Renders repo navigation + content wrapper
- `app/(app)/org/[orgId]/transformations/[transformationId]/layout.tsx` - Renders transformation navigation + content wrapper
- `features/navigation/conditional-org-navigation.tsx` - Updated with segment-aware logic using `useSelectedLayoutSegments()`
- `features/navigation/components/Header.tsx` - Removed tab rendering code (~70 lines)
- `components/ui/nav-tabs.tsx` - Updated styling to match Header tabs exactly
- `features/navigation/contextual-nav/config.ts` - Added repo settings matcher (temporary, can be removed)

## Technical Debt Cleanup (Optional)

The following files can now be deleted since they're no longer used:

- ❌ `features/navigation/contextual-nav/config.ts` - Regex-based route matchers
- ❌ `features/navigation/contextual-nav/filterTabs.ts` - Tab filtering logic moved to components
- ❌ `features/navigation/contextual-nav/index.ts` - Exports for removed resolver

These are kept temporarily to avoid breaking anything, but are not actively used.

## Migration Notes

### Before: Regex Approach
```typescript
// Add new route → Update regex pattern
matchers: [
  /^\/org\/[^/]+\/repo\/[^/]+\/?$/,
  /^\/org\/[^/]+\/repo\/[^/]+\/settings(?:\/.*)?$/,
  /^\/org\/[^/]+\/repo\/[^/]+\/workflows(?:\/.*)?$/,  // New!
]
```

### After: Layout Approach
```typescript
// Add new route → Add new tab
const tabs = [
  { href: base, label: 'Elements', active: segment === null },
  { href: `${base}/settings`, label: 'Settings', active: segment === 'settings' },
  { href: `${base}/workflows`, label: 'Workflows', active: segment === 'workflows' },  // New!
];
```

## Post-Implementation Fixes

### Issue: List Pages Missing Navigation and Layout

**Problem**: When navigating to `/org/atlas/transformations` or `/org/atlas/repo`, users lost both contextual navigation and constrained layout.

**Root Cause**:
1. `ConditionalOrgNavigation` was too aggressive - hiding navigation for ANY `transformations` segment, including list pages
2. List pages lacked content wrapper constraints

**Solution** (2025-01-29):

#### 1. Smart Segment Detection
Updated `ConditionalOrgNavigation` to use `useSelectedLayoutSegments()` (plural) to distinguish list from detail pages:

```typescript
// features/navigation/conditional-org-navigation.tsx
const segments = useSelectedLayoutSegments();

// /org/atlas/transformations → segments = ['transformations'] → Show OrgNav ✓
// /org/atlas/transformations/[id] → segments = ['transformations', '[transformationId]'] → Hide
if (segment === 'transformations' && segments.length > 1) {
  return null;
}
```

#### 2. Added Content Wrappers
Applied `mx-auto w-full max-w-7xl px-4 py-6 sm:px-6 lg:px-8` to all return statements in:
- `app/(app)/org/[orgId]/transformations/page.tsx`
- `app/(app)/org/[orgId]/repo/page.tsx`

**Verification**:
| Route | Navigation | Layout |
|-------|-----------|--------|
| `/org/atlas/transformations` | ✅ OrgNav (Initiatives active) | ✅ Constrained |
| `/org/atlas/transformations/[id]` | ✅ InitiativeNav | ✅ Constrained |
| `/org/atlas/repo` | ✅ OrgNav (Repositories active) | ✅ Constrained |
| `/org/atlas/repo/[id]` | ✅ RepoNav | ✅ Constrained |

## Conclusion

This refactoring aligns the codebase with Next.js 14+ best practices for navigation, making the system more maintainable, performant, and easier to extend. The route structure is now the single source of truth for navigation, eliminating the need for complex regex patterns and manual pathname parsing.

The implementation now properly distinguishes between list and detail pages, ensuring navigation and layout constraints apply correctly across all routes.

## Related Issues

- Fixed: Organization settings page defaulting to "Repositories" section (now defaults to "Features")
- Fixed: Repository settings routes not showing proper navigation context
- Fixed: Navigation tabs constrained to page width (now full-width like header)
- Fixed: List pages (`/transformations`, `/repo`) missing navigation and layout constraints

## References

- Next.js Documentation: [Layouts and Templates](https://nextjs.org/docs/app/building-your-application/routing/pages-and-layouts)
- Next.js API: [`useSelectedLayoutSegment`](https://nextjs.org/docs/app/api-reference/functions/use-selected-layout-segment)
- Related Backlog: `backlog/315-ui-quality-foundations.md` (navigation consistency)
