# Backlog Item 329: Authorization & Organization Isolation Architecture

> **Status (2026-01): ACTIVE, but contains legacy schema examples**
>
> Some examples in this document reference legacy nouns/tables (e.g., `graph_id`, `archimate_elements`) from earlier iterations.
>
> **Interpretation rule (authoritative):**
> - Repository scope is `{tenant_id, organization_id, repository_id}` (not `graph_id`).
> - Canonical storage is Elements + Versions + Relationships; typed tables are historical. See `backlog/300-400/303-elements.md`.
> - For agentic memory, “permission-trimmed retrieval” is a hard requirement: `backlog/656-agentic-memory-v6.md`.

**Status**: 🟡 Phase 2 In Progress
**Priority**: P0 - Critical
**Component**: Authorization, Security, Architecture
**Estimated Effort**: 8-10 weeks (phased implementation)
**Related**: [322-rbac.md](./322-rbac.md), [323-keycloak-realm-roles.md](./323-keycloak-realm-roles.md), [319-onboarding.md](./319-onboarding.md), [333-realms.md](./333-realms.md)

---

## ⚙️ PLATFORM UPDATE (2025-12-22)

Transport/routing determinism is now tracked in `backlog/589-rpc-transport-determinism.md`. This does not change authz policy goals, but it is a prerequisite for validating authz and tenant isolation with confidence (failures should come from authorization decisions and data scoping, not protocol misrouting).

---

## ⚠️ ARCHITECTURE UPDATE (2025-10-30)

**Long-term architecture is changing to realm-per-tenant.** See [backlog/333-realms.md](./333-realms.md) for full details.

**Impact on this document**:
- This document describes authorization within a **single Keycloak realm** where organizations are groups
- The new architecture will use **realm-per-tenant** where each organization has its own Keycloak realm
- The API authorization patterns (middleware, RLS) described here **remain valid** but will be applied per-realm
- Organization membership will be determined by realm membership, not database groups
- Phase 1 & 2 work here is still valuable and can be adapted to realm-per-tenant architecture

**Migration path**:
1. Complete Phase 1 & 2 (API authorization + database scoping) for current single-realm architecture
2. Transition to realm-per-tenant architecture (Phase 3 of 333-realms.md)
3. Adapt authorization middleware to work with multi-realm tokens

---

## Implementation Progress (2025-11-02)

### Phase 1: API Authorization Layer – PARTIAL 🟡

**Delivered**
- Central `withAuth` / `requireOrganizationMember` / `requireOrganizationRole` helpers now wrap repositories, transformations, agents, threads (history + stream), and invitation APIs. Coverage extends to org-scoped server actions under `app/(app)/org/[orgId]/**`.
- Session enrichment continues to supply canonical membership roles (platform id first, `kcSub` fallback) so middleware decisions align with Keycloak + platform ids.

**Gaps**
- Some legacy `/api/v1/**` and onboarding utility routes still need an explicit audit to confirm they invoke `withAuth` or equivalent checks.
- No regression coverage yet that exercises the middleware paths (401/403/200) end-to-end; failures would surface only in production.
- Authorization denials are not consistently logged/observed; telemetry for rejected requests is still pending.

**Next Steps**
- Complete the audit of remaining API routes/server actions (including onboarding and any `/api/v1` stragglers) and apply `withAuth` where missing.
- Backfill unit/integration tests for the middleware (happy path, 401, 403) and add smoke coverage in Playwright.
- Add structured logging / metrics for authorization failures to support Phase 3 auditing.

### Phase 2: Database Organization Scoping – NOT STARTED 🔴

- Repository currently contains only `db/flyway/sql/platform/V1__initial_schema.sql`; the referenced `V2__add_organization_scoping.sql` migration has not been authored or committed.
- Proto definitions and SDKs still lack `organization_id` fields on repositories/initiatives/etc., so isolation is still tenant-only.

**Actions**
- Draft and commit the Flyway migration(s) that introduce `organization_id` plus supporting indexes, along with a deterministic backfill plan.
- Regenerate protobufs/SDKs once the schema delta exists, then plumb the new fields through services.

---

## Executive Summary

**Critical Finding**: Most public APIs now enforce organization membership/roles, but authorization remains incomplete because data is not isolated at the database layer and a few legacy endpoints still need verification.

**Risk Level**: 🟡 **High** (reduced from Critical)
- ✅ Majority of API routes protected with membership checks via shared middleware
- ⚠️ Role-based permissions applied selectively; some handlers still allow any member
- ⚠️ Legacy/utility routes require audit to confirm coverage
- ❌ Organization isolation: Data not scoped to organizations in database

**Current State**:
- ✅ Authentication: Keycloak SSO, session management
- ✅ UI-level RBAC: Tab filtering, visual permissions
- 🟡 API authorization: Core routes protected, role enforcement TODO
- ❌ Organization isolation: Data not scoped to organizations in database
- ⚠️ Role enforcement: Middleware checks membership, full role validation pending

**Discovered During**: Investigation of onboarding session refresh bug (item 319)
**Documented**: 2025-10-30
**Implementation Started**: 2025-10-30

---

## Problem Statement

### Architecture Gap: Organizations Don't Isolate Data

**Intended Architecture**:
```
Tenant (atlas-org)
├── Organization: ACME Corp
│   ├── Users: [Alice, Bob]
│   ├── Repositories: [ACME-Repo-1, ACME-Repo-2]
│   └── Data: Scoped to ACME
│
└── Organization: Competitor Inc
    ├── Users: [Charlie]
    ├── Repositories: [Competitor-Repo-1]
    └── Data: Scoped to Competitor
```

**Actual Architecture**:
```
Tenant (atlas-org)
├── Organizations: [ACME, Competitor] (metadata only)
├── Memberships: [Alice→ACME, Bob→ACME, Charlie→Competitor]
├── Repositories: [ALL] (tenant_id only, no organization_id)
├── Initiatives: [ALL] (tenant_id only, no organization_id)
└── Elements: [ALL] (tenant_id only, no organization_id)

❌ No organization_id in business data tables
❌ All queries filter by tenant only
❌ No isolation between organizations
```

### Security Implications

**Attack Vector 1: Cross-Organization Data Access (service-to-service)**
```javascript
// Alice belongs to ACME.
// A compromised service calls the graph API directly (bypassing Next API middleware)
// and supplies only tenantId. Because repositories lack organization_id, all tenant data returns.

await client.graphService.listRepositories({
  context: { tenantId: 'atlas-org', userId: 'alice' }
});

// Result today: Returns repositories across every organization within atlas-org ❌
// Desired: Backend requires organizationId and enforces membership/role.
```

**Attack Vector 2: Lax Role Enforcement**
```typescript
// Bob is a basic member in ACME.
// A handler that only checks membership (not role) allows sensitive mutations.

export const POST = withAuth(async (_req, auth) => {
  // Missing: requireOrganizationRole(['admin', 'owner'])
  return enablePremiumFeature(auth.membership!.organizationId);
}, { requireOrgMembership: true });

// Result today: Any member can flip premium features unless handler specifies requiredRoles ❌
// Desired: Middleware enforces role gates consistently.
```

**Attack Vector 3: Organization Enumeration (non-API consumers)**
```sql
-- Direct SQL or ad-hoc scripts that filter only by tenant_id
SELECT * FROM platform.organizations WHERE tenant_id = 'atlas-org';
-- Still reveals every org slug/key unless queries add user/org predicates ❌
```

---

## Current State Analysis

### Database Layer

#### ✅ What Exists:
```sql
-- Platform schema with organizations
CREATE TABLE platform.organizations (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  key VARCHAR(128) NOT NULL,
  name VARCHAR(255) NOT NULL
);

-- Membership tracking
CREATE TABLE platform.organization_memberships (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  organization_id VARCHAR(255) REFERENCES platform.organizations(id),
  user_id VARCHAR(255) NOT NULL,
  role_ids TEXT[],
  state VARCHAR(64) -- ACTIVE, SUSPENDED, etc.
);

-- Role definitions
CREATE TABLE platform.organization_roles (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  organization_id VARCHAR(255) NOT NULL,
  code VARCHAR(64) NOT NULL, -- 'owner', 'admin', 'member', 'viewer'
  permissions JSONB
);
```

#### ❌ What's Missing:
```sql
-- Business data tables lack organization_id
CREATE TABLE repositories (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  key VARCHAR(128) NOT NULL,
  name VARCHAR(255) NOT NULL
  -- ❌ NO organization_id column!
);

CREATE TABLE transformations (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  graph_id VARCHAR(255),
  key VARCHAR(128) NOT NULL
  -- ❌ NO organization_id column!
);

CREATE TABLE archimate_elements (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  graph_id VARCHAR(255)
  -- ❌ NO organization_id column!
);

-- No Row-Level Security policies
-- All tables have: ENABLE ROW LEVEL SECURITY = false
-- Any query can access any data (app-level filtering only)
```

### Proto/API Layer

#### ✅ Recent Fix (2025-10-30):
```protobuf
message ListOrganizationsRequest {
  RequestContext context = 1;
  PaginationRequest pagination = 2;
  string tenant_id = 3;
  string user_id = 4;  // ✅ Added, implemented with membership filtering
}
```

**Implementation**:
```typescript
// services/platform/src/organizations/service.ts
async listOrganizations(request) {
  // ✅ Now filters by user membership
  if (request.userId) {
    query += `
      AND EXISTS (
        SELECT 1 FROM platform.organization_memberships m
        WHERE m.organization_id = o.id
          AND m.user_id = $2
          AND m.state = 'MEMBERSHIP_STATE_ACTIVE'
      )
    `;
  }
}
```

> ℹ️ Membership state filters must use the enum-formatted values persisted in Postgres (`MEMBERSHIP_STATE_ACTIVE`, `MEMBERSHIP_STATE_INVITED`, etc.), not the shortened enum keys.

#### ❌ Gaps in Other Protos:
```protobuf
// All other list operations lack user/org filtering
message ListRepositoriesRequest {
  RequestContext context = 1;
  PaginationRequest pagination = 2;
  string tenant_id = 3;
  // ❌ NO user_id filter
  // ❌ NO organization_id filter
}

message ListInitiativesRequest {
  RequestContext context = 1;
  PaginationRequest pagination = 2;
  string tenant_id = 3;
  // ❌ NO organization_id filter
}
```

### Application Layer

#### ✅ Current Coverage
- Routes under `app/api/repositories/**`, `app/api/transformations/**`, `app/api/agents/**`, `app/api/threads/**`, and invitation flows wrap handlers with `withAuth` to require membership (and, where specified, roles).
- Org-scoped server actions in `app/(app)/org/[orgId]/**` now require membership/roles before mutating state.
- Example:
  ```typescript
  export const GET = withAuth(handleGet, {
    requireOrgMembership: true,
    requiredRoles: ['admin', 'owner'],
    orgIdSource: 'query',
  });
  ```

#### ⚠️ Remaining Gaps
- Legacy `/api/v1/**` utilities and onboarding status endpoints still need verification; not all use `withAuth`.
- Role requirements are opt-in per handler. Several routes ensure membership only, so sensitive actions may still be accessible to any member.
- No automated regression tests exist for the authorization wrappers; behaviour is validated manually.

---

## Remediation Plan

### Phase 1: API Authorization Layer (P0 - 2 weeks) - IN PROGRESS

**Goal**: Add permission checks to all API routes and server actions

**Status**: 🟡 Core implementation complete, remaining work needed

**1.1 Create Authorization Middleware** ✅ COMPLETE

Key building blocks now live in `services/www_ameide_platform/lib/auth/middleware/authorize.ts`:

- `requireAuthentication`, `requireOrganizationMember`, `requireOrganizationRole` – resolve platform vs Keycloak ids, translate slugs → ids, and enforce membership/role checks.
- `withAuth` – wraps route handlers, automatically extracts/derives the org id (query/body/custom resolver), invokes membership/role guards, and forwards `AuthorizationResult` to the handler.
- Helpers such as `extractOrgId`, `handleAuthError`, and `withSessionAuth` simplify legacy callers.

Example (excerpt):

```typescript
export function withAuth<T extends unknown[]>(
  handler: (request: Request, auth: AuthorizationResult, ...args: T) => Promise<Response>,
  options: WithAuthOptions<T> = {}
) {
  return async (request: Request, ...args: T): Promise<Response> => {
    const session = await requireAuthentication();
    // Resolve orgId, run membership/role checks, then call handler
    const authResult = await runAuthorizationChecks(request, session, options, ...args);
    return handler(request, authResult, ...args);
  };
}
```

**1.2 Apply to All API Routes**

```typescript
// app/api/repositories/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { withAuth, type AuthorizationResult } from '@/lib/auth/middleware';
import { getServerClient } from '@/lib/sdk/server-client';

async function handleGet(request: NextRequest, auth: AuthorizationResult) {
  const { searchParams } = new URL(request.url);
  const orgId = searchParams.get('orgId');
  if (!orgId) {
    return NextResponse.json({ error: 'orgId parameter required' }, { status: 400 });
  }

  const client = getServerClient();
  const repos = await client.graphService.listRepositories({
    context: {
      tenantId: auth.session.user.tenantId!,
      userId: auth.session.user.id,
      requestId: crypto.randomUUID(),
    },
  });

  return NextResponse.json({ repositories: repos.repositories });
}

async function handlePost(request: NextRequest, auth: AuthorizationResult) {
  const body = await request.json();
  const { orgId, ...data } = body;
  if (!orgId) {
    return NextResponse.json({ error: 'orgId required' }, { status: 400 });
  }

  const client = getServerClient();
  const repo = await client.graphService.createRepository({
    context: {
      tenantId: auth.session.user.tenantId!,
      userId: auth.session.user.id,
      requestId: crypto.randomUUID(),
    },
    graph: data,
  });

  return NextResponse.json({ graph: repo.graph });
}

export const GET = withAuth(handleGet, {
  requireOrgMembership: true,
  orgIdSource: 'query',
});

export const POST = withAuth(handlePost, {
  requiredRoles: ['owner', 'admin', 'contributor'],
  resolveOrgId: async (_req, _session, auth) => auth.membership?.organizationId ?? null,
});
```

**1.3 Apply to All Server Actions**

```typescript
// app/(app)/org/[orgId]/settings/actions.ts
'use server';

import { requireOrganizationRole } from '@/lib/auth/middleware';
import { getServerClient } from '@/lib/sdk/server-client';

export async function updateOrganizationFeatures(
  orgId: string,
  features: Record<string, boolean>
) {
  // ✅ Add authorization check
  const { session } = await requireOrganizationRole(orgId, ['owner', 'admin']);

  const client = getServerClient();
  await client.organizationService.updateOrganization({
    context: {
      tenantId: session.user.organization?.tenantId || 'atlas-org',
      userId: session.user.id,
      requestId: crypto.randomUUID(),
    },
    organization: {
      metadata: { id: orgId },
      settings: {
        featureToggles: features,
      },
    },
  });
}

export async function deleteRepository(orgId: string, repoId: string) {
  // ✅ Add authorization check
  const { session } = await requireOrganizationRole(orgId, ['owner', 'admin']);

  const client = getServerClient();
  await client.graphService.deleteRepository({
    context: {
      tenantId: session.user.organization?.tenantId || 'atlas-org',
      userId: session.user.id,
      requestId: crypto.randomUUID(),
    },
    graphId: repoId,
  });
}
```

**1.4 Add Integration Tests**

```typescript
// lib/auth/middleware/__tests__/authorize.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { requireOrganizationMember, requireOrganizationRole } from '../authorize';

vi.mock('@/app/(auth)/auth');
vi.mock('@/lib/sdk/server-client');

describe('Authorization Middleware', () => {
  describe('requireOrganizationMember', () => {
    it('throws UnauthorizedError when no session', async () => {
      vi.mocked(getSession).mockResolvedValue(null);

      await expect(requireOrganizationMember('acme')).rejects.toThrow('Authentication required');
    });

    it('throws ForbiddenError when user not member', async () => {
      vi.mocked(getSession).mockResolvedValue({
        user: { id: 'user-123', email: 'test@example.com' }
      });

      vi.mocked(getServerClient).mockReturnValue({
        organizationService: {
          listOrganizationMemberships: vi.fn().mockResolvedValue({
            memberships: []  // No membership
          })
        }
      });

      await expect(requireOrganizationMember('acme')).rejects.toThrow(
        'Not a member of organization: acme'
      );
    });

    it('returns session and membership when authorized', async () => {
      vi.mocked(getSession).mockResolvedValue({
        user: { id: 'user-123', email: 'test@example.com' }
      });

      vi.mocked(getServerClient).mockReturnValue({
        organizationService: {
          listOrganizationMemberships: vi.fn().mockResolvedValue({
            memberships: [{
              userId: 'user-123',
              organizationId: 'acme',
              state: 2,  // ACTIVE
              roleIds: ['role-1']
            }]
          })
        }
      });

      const result = await requireOrganizationMember('acme');
      expect(result.session.user.id).toBe('user-123');
      expect(result.membership.organizationId).toBe('acme');
    });
  });

  describe('requireOrganizationRole', () => {
    it('throws ForbiddenError when role insufficient', async () => {
      // Mock user with 'viewer' role trying to access admin endpoint
      // ... test implementation
    });

    it('allows access when role matches', async () => {
      // Mock user with 'admin' role accessing admin endpoint
      // ... test implementation
    });
  });
});
```

```typescript
// app/api/repositories/__tests__/authorization.integration.test.ts
import { describe, it, expect } from 'vitest';
import { NextRequest } from 'next/server';
import { GET, POST } from '../route';

describe('GET /api/repositories authorization', () => {
  it('returns 401 for unauthenticated users', async () => {
    const request = new NextRequest('http://localhost/api/repositories?orgId=acme');
    const response = await GET(request);

    expect(response.status).toBe(401);
    const body = await response.json();
    expect(body.error).toContain('Authentication required');
  });

  it('returns 403 for non-members', async () => {
    // Mock session for user NOT in 'acme' org
    // ... test implementation
  });

  it('returns 200 for members', async () => {
    // Mock session for user IN 'acme' org
    // ... test implementation
  });
});

describe('POST /api/repositories authorization', () => {
  it('returns 403 for viewers', async () => {
    // Mock user with 'viewer' role trying to create
    // ... test implementation
  });

  it('returns 201 for contributors', async () => {
    // Mock user with 'contributor' role creating
    // ... test implementation
  });
});
```

**1.5 Rollout Checklist**

- [x] Create authorization middleware module
- [x] Update all routes in `app/api/agents/**`
- [x] Update all routes in `app/api/transformations/**`
- [x] Update all routes in `app/api/repositories/**`
- [x] Update all routes in `app/api/elements/**`
- [x] Update all routes in `app/api/threads/**`
- [ ] Update all routes in `app/api/workflows/**` (confirm coverage or note N/A)
- [x] Update all server actions in `app/(app)/org/[orgId]/**`
- [ ] Add integration tests for each route (401/403/200)
- [ ] Add E2E tests for critical flows
- [ ] Document authorization patterns in README
- [x] TypeScript compilation passing

**Success Criteria**:
- All API routes check organization membership
- All write operations check role permissions
- Integration tests verify 401/403 responses
- Zero regressions in E2E tests

---

### Phase 2: Organization Data Scoping (P0 - 4 weeks)

**Goal**: Add organization_id to all business data and enforce isolation

**2.1 Database Migrations**

- TODO: Author `db/flyway/sql/platform/V2__add_organization_scoping.sql` (or equivalent) to introduce `organization_id` columns, foreign keys, and supporting indexes across repositories, transformations, initiatives, graph tables, etc.
- Backfill strategy must be defined: how to map existing tenant-scoped data to organizations (e.g., derive from repository metadata, default to primary org, or block upgrade until org mapped).
- Migration should finish by enforcing `NOT NULL` + FK constraints once backfill succeeds.
- Deployment checklist (to be built once scripts exist):
  1. Generate and publish updated protobufs/SDKs.
  2. Apply Flyway migration in staging → prod with verification queries for new indexes/constraints.
  3. Validate data integrity and roll back plan.

**2.2 Update Proto Definitions**

*Planned once database schema supports organization scoping.*

```protobuf
// packages/ameide_core_proto/src/ameide_core_proto/graph/v1/graph.proto

message Repository {
  ameide_core_proto.common.v1.ResourceMetadata metadata = 1;
  string organization_id = 2;  // ✅ NEW: Organization ownership
  string key = 3;
  string name = 4;
  string description = 5;
  // ... rest of fields
}

message ListRepositoriesRequest {
  ameide_core_proto.common.v1.RequestContext context = 1;
  ameide_core_proto.common.v1.PaginationRequest pagination = 2;
  string tenant_id = 3;
  string organization_id = 4;  // ✅ NEW: Filter by org
  string user_id = 5;           // ✅ NEW: Filter by user membership
}

message CreateRepositoryRequest {
  ameide_core_proto.common.v1.RequestContext context = 1;
  Repository graph = 2;  // Must include organization_id
}
```

**2.3 Update Backend Services**

*Target implementation after proto/schema updates land.*

```typescript
// services/platform/src/repositories/service.ts

async listRepositories(request: graphV1.ListRepositoriesRequest) {
  const tenantId = requireTenantId(request.context);
  const organizationId = request.organizationId;
  const userId = request.userId;

  let query = `
    SELECT r.*
    FROM repositories r
    WHERE r.tenant_id = $1 AND r.deleted_at IS NULL
  `;

  const params: (string | number)[] = [tenantId];

  // Filter by organization if provided
  if (organizationId && organizationId.trim().length > 0) {
    query += ` AND r.organization_id = $${params.length + 1}`;
    params.push(organizationId.trim());
  }

  // Filter by user's organizations if userId provided
  if (userId && userId.trim().length > 0) {
    query += `
      AND r.organization_id IN (
        SELECT m.organization_id
        FROM platform.organization_memberships m
        WHERE m.tenant_id = r.tenant_id
          AND m.user_id = $${params.length + 1}
          AND m.state = 'MEMBERSHIP_STATE_ACTIVE'
      )
    `;
    params.push(userId.trim());
  }

  query += ` ORDER BY r.created_at DESC OFFSET $${params.length + 1} LIMIT $${params.length + 2}`;
  params.push(offset, pageSize);

  const { rows } = await pool.query<RepositoryRow>(query, params);

  return create(graphV1.ListRepositoriesResponseSchema, {
    repositories: rows.map(toRepository),
    pagination: buildPaginationResponse(rows.length, pageSize, offset),
  });
}

async createRepository(request: graphV1.CreateRepositoryRequest) {
  const tenantId = requireTenantId(request.context);
  const userId = request.context?.userId;
  const graph = request.graph;

  if (!graph?.organizationId) {
    throw new RpcError('organization_id is required', Code.InvalidArgument);
  }

  // ✅ Verify user is member of the organization
  if (userId) {
    const membership = await this.verifyUserMembership(
      tenantId,
      userId,
      graph.organizationId
    );

    if (!membership) {
      throw new RpcError(
        'User is not a member of this organization',
        Code.PermissionDenied
      );
    }
  }

  // Create graph with organization_id
  const { rows } = await pool.query<RepositoryRow>(
    `
      INSERT INTO repositories (
        id, tenant_id, organization_id, key, name, description, created_at, updated_at
      )
      VALUES ($1, $2, $3, $4, $5, $6, NOW(), NOW())
      RETURNING *
    `,
    [
      crypto.randomUUID(),
      tenantId,
      graph.organizationId,
      graph.key,
      graph.name,
      graph.description,
    ]
  );

  return create(graphV1.CreateRepositoryResponseSchema, {
    graph: toRepository(rows[0]),
  });
}
```

**2.4 Create Data Access Layer**

*Proposed abstraction to keep controllers thin once organization_id exists.*

```typescript
// lib/dal/graph-dal.ts
import { getServerClient } from '@/lib/sdk/server-client';
import type { Session } from 'next-auth';

export class RepositoryDAL {
  constructor(
    private session: Session,
    private tenantId: string = 'atlas-org'
  ) {}

  /**
   * List repositories scoped to user's organizations
   */
  async list(organizationId?: string) {
    const client = getServerClient();

    // If specific org requested, verify membership
    if (organizationId) {
      await this.verifyMembership(organizationId);
    }

    const response = await client.graphService.listRepositories({
      context: {
        tenantId: this.tenantId,
        userId: this.session.user.id,
        requestId: crypto.randomUUID(),
      },
      tenantId: this.tenantId,
      organizationId,
      userId: this.session.user.id,  // Backend filters by membership
    });

    return response.repositories || [];
  }

  /**
   * Get single graph (with ownership check)
   */
  async get(graphId: string) {
    const repo = await client.graphService.getRepository({
      context: {
        tenantId: this.tenantId,
        userId: this.session.user.id,
        requestId: crypto.randomUUID(),
      },
      graphId,
    });

    // Verify user is member of repo's organization
    await this.verifyMembership(repo.graph.organizationId);

    return repo.graph;
  }

  private async verifyMembership(organizationId: string) {
    const client = getServerClient();
    const response = await client.organizationService.listOrganizationMemberships({
      context: {
        tenantId: this.tenantId,
        userId: this.session.user.id,
        requestId: crypto.randomUUID(),
      },
      organizationId,
      pagination: { pageSize: 1 },
    });

    const isMember = response.memberships?.some(
      m => m.userId === this.session.user.id && m.state === 2
    );

    if (!isMember) {
      throw new Error(`User is not a member of organization: ${organizationId}`);
    }
  }
}
```

**2.5 Update Frontend API Routes**

```typescript
// app/api/repositories/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getSession } from '@/app/(auth)/auth';
import { RepositoryDAL } from '@/lib/dal/graph-dal';

export async function GET(request: NextRequest) {
  const session = await getSession();
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { searchParams } = new URL(request.url);
  const orgId = searchParams.get('orgId');

  // ✅ Use DAL with automatic scoping
  const dal = new RepositoryDAL(session);
  const repositories = await dal.list(orgId);

  return NextResponse.json({ repositories });
}
```

**2.6 Rollout Checklist**

- [ ] Create database migrations for all tables
- [ ] Test migrations on staging database
- [ ] Update all proto definitions with organization_id
- [ ] Regenerate proto code: `buf generate`
- [ ] Update all backend services to filter by organization
- [ ] Create DAL modules for each resource type
- [ ] Update frontend to use DAL instead of direct client calls
- [ ] Add integration tests for org scoping
- [ ] Add E2E tests for cross-org isolation
- [ ] Deploy migrations (with rollback plan)
- [ ] Deploy backend services
- [ ] Deploy frontend

**Success Criteria**:
- All queries filter by organization_id
- Cross-organization data access impossible
- Integration tests verify isolation
- Zero data leakage in E2E tests

---

### Phase 3: Defense in Depth (P1 - 4 weeks)

**Goal**: Add database-level security and audit logging

**3.1 Enable Row-Level Security**

```sql
-- Migration: V51__enable_rls.sql

-- Enable RLS on all tables
ALTER TABLE repositories ENABLE ROW LEVEL SECURITY;
ALTER TABLE transformations ENABLE ROW LEVEL SECURITY;
ALTER TABLE archimate_elements ENABLE ROW LEVEL SECURITY;
-- ... repeat for all business tables

-- Policy: Users can only SELECT resources in their organizations
CREATE POLICY repositories_select_policy ON repositories
  FOR SELECT
  USING (
    tenant_id = current_setting('app.tenant_id', true)::VARCHAR
    AND organization_id IN (
      SELECT m.organization_id
      FROM platform.organization_memberships m
      WHERE m.tenant_id = repositories.tenant_id
        AND m.user_id = current_setting('app.user_id', true)::VARCHAR
        AND m.state = 'MEMBERSHIP_STATE_ACTIVE'
    )
  );

-- Policy: Only org admins/owners can UPDATE
CREATE POLICY repositories_update_policy ON repositories
  FOR UPDATE
  USING (
    tenant_id = current_setting('app.tenant_id', true)::VARCHAR
    AND EXISTS (
      SELECT 1
      FROM platform.organization_memberships m
      JOIN platform.organization_roles r ON r.id = ANY(m.role_ids)
      WHERE m.organization_id = repositories.organization_id
        AND m.tenant_id = repositories.tenant_id
        AND m.user_id = current_setting('app.user_id', true)::VARCHAR
        AND m.state = 'MEMBERSHIP_STATE_ACTIVE'
        AND r.code IN ('owner', 'admin')
    )
  );

-- Policy: Only org admins/owners can DELETE
CREATE POLICY repositories_delete_policy ON repositories
  FOR DELETE
  USING (
    tenant_id = current_setting('app.tenant_id', true)::VARCHAR
    AND EXISTS (
      SELECT 1
      FROM platform.organization_memberships m
      JOIN platform.organization_roles r ON r.id = ANY(m.role_ids)
      WHERE m.organization_id = repositories.organization_id
        AND m.tenant_id = repositories.tenant_id
        AND m.user_id = current_setting('app.user_id', true)::VARCHAR
        AND m.state = 'MEMBERSHIP_STATE_ACTIVE'
        AND r.code IN ('owner', 'admin')
    )
  );

-- Policy: Members with contributor role can INSERT
CREATE POLICY repositories_insert_policy ON repositories
  FOR INSERT
  WITH CHECK (
    tenant_id = current_setting('app.tenant_id', true)::VARCHAR
    AND EXISTS (
      SELECT 1
      FROM platform.organization_memberships m
      JOIN platform.organization_roles r ON r.id = ANY(m.role_ids)
      WHERE m.organization_id = repositories.organization_id
        AND m.tenant_id = repositories.tenant_id
        AND m.user_id = current_setting('app.user_id', true)::VARCHAR
        AND m.state = 'MEMBERSHIP_STATE_ACTIVE'
        AND r.code IN ('owner', 'admin', 'contributor')
    )
  );
```

**3.2 Set Session Variables on Each Request**

```typescript
// services/platform/src/common/db.ts
import { Pool } from 'pg';
import type { RequestContext } from '@ameideio/ameide-sdk-ts/common';

export async function setSessionContext(
  pool: Pool,
  context: RequestContext
): Promise<void> {
  if (!context.tenantId || !context.userId) {
    throw new Error('tenantId and userId required for RLS');
  }

  await pool.query(
    `SELECT
      set_config('app.tenant_id', $1, true),
      set_config('app.user_id', $2, true)`,
    [context.tenantId, context.userId]
  );
}

// Use in service handlers
async listRepositories(request: ListRepositoriesRequest) {
  const context = requireContext(request.context);

  // ✅ Set RLS session variables
  await setSessionContext(pool, context);

  // Now queries automatically respect RLS policies
  const { rows } = await pool.query(`SELECT * FROM repositories`);
  // RLS filters results automatically

  return { repositories: rows.map(toRepository) };
}
```

**3.3 Add Audit Logging**

```sql
-- Migration: V52__audit_logging.sql

CREATE TABLE platform.audit_log (
  id VARCHAR(255) PRIMARY KEY DEFAULT gen_random_uuid()::text,
  tenant_id VARCHAR(255) NOT NULL,
  organization_id VARCHAR(255),
  user_id VARCHAR(255) NOT NULL,
  action VARCHAR(64) NOT NULL,
  resource_type VARCHAR(64) NOT NULL,
  resource_id VARCHAR(255),
  allowed BOOLEAN NOT NULL,
  denied_reason TEXT,
  request_id VARCHAR(255),
  timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  metadata JSONB,
  ip_address INET,
  user_agent TEXT
);

CREATE INDEX idx_audit_log_tenant_user ON platform.audit_log(tenant_id, user_id);
CREATE INDEX idx_audit_log_resource ON platform.audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_log_denied ON platform.audit_log(tenant_id, allowed) WHERE allowed = false;
CREATE INDEX idx_audit_log_timestamp ON platform.audit_log(timestamp DESC);
```

```typescript
// lib/audit/logger.ts
import { getServerClient } from '@/lib/sdk/server-client';

export interface AuditEntry {
  tenantId: string;
  organizationId?: string;
  userId: string;
  action: string;
  resourceType: string;
  resourceId?: string;
  allowed: boolean;
  deniedReason?: string;
  requestId?: string;
  metadata?: Record<string, any>;
  ipAddress?: string;
  userAgent?: string;
}

export async function logAudit(entry: AuditEntry): Promise<void> {
  const client = getServerClient();

  await client.auditService.createAuditEntry({
    context: {
      tenantId: entry.tenantId,
      userId: entry.userId,
      requestId: entry.requestId || crypto.randomUUID(),
    },
    entry: {
      tenantId: entry.tenantId,
      organizationId: entry.organizationId,
      userId: entry.userId,
      action: entry.action,
      resourceType: entry.resourceType,
      resourceId: entry.resourceId,
      allowed: entry.allowed,
      deniedReason: entry.deniedReason,
      metadata: entry.metadata,
      ipAddress: entry.ipAddress,
      userAgent: entry.userAgent,
    },
  });
}

// Use in authorization middleware
export async function requireOrganizationMember(orgId: string) {
  const session = await getSession();

  try {
    // ... membership check logic

    // ✅ Log successful authorization
    await logAudit({
      tenantId: 'atlas-org',
      organizationId: orgId,
      userId: session.user.id,
      action: 'authz:check_membership',
      resourceType: 'organization',
      resourceId: orgId,
      allowed: true,
    });

    return { session, membership };
  } catch (error) {
    // ✅ Log denied attempt
    await logAudit({
      tenantId: 'atlas-org',
      organizationId: orgId,
      userId: session?.user?.id || 'anonymous',
      action: 'authz:check_membership',
      resourceType: 'organization',
      resourceId: orgId,
      allowed: false,
      deniedReason: error.message,
    });

    throw error;
  }
}
```

**3.4 Rollout Checklist**

- [ ] Create RLS policies for all tables
- [ ] Test RLS policies on staging
- [ ] Update services to set session context
- [ ] Add audit logging table and service
- [ ] Integrate audit logging in middleware
- [ ] Add monitoring/alerting for denied attempts
- [ ] Deploy RLS policies (with bypass for migrations)
- [ ] Deploy audit logging
- [ ] Create dashboard for security monitoring

**Success Criteria**:
- Database enforces permissions (defense in depth)
- All authorization attempts logged
- Security team can monitor denied access attempts
- Compliance audit trail available

---

## Testing Strategy

### Unit Tests
```typescript
// lib/auth/middleware/__tests__/authorize.test.ts
- requireOrganizationMember throws UnauthorizedError when no session
- requireOrganizationMember throws ForbiddenError when not member
- requireOrganizationRole throws ForbiddenError when insufficient role
- requireOrganizationRole allows access when role matches
```

### Integration Tests
```typescript
// app/api/repositories/__tests__/authorization.integration.test.ts
- GET /api/repositories returns 401 for unauthenticated
- GET /api/repositories returns 403 for non-members
- GET /api/repositories returns 200 for members
- POST /api/repositories returns 403 for viewers
- POST /api/repositories returns 201 for contributors
- DELETE /api/repositories returns 403 for contributors
- DELETE /api/repositories returns 200 for admins
```

### E2E Tests (Playwright)
```typescript
// features/authorization/__tests__/e2e/cross-org-isolation.spec.ts
test('user cannot access other organization data', async ({ page }) => {
  // Login as Alice (member of ACME)
  await loginAs(page, 'alice@acme.com');

  // Navigate to ACME's repos
  await page.goto('/org/acme/repositories');
  await expect(page.getByText('ACME-Repo-1')).toBeVisible();

  // Try to access Competitor's repos via URL manipulation
  await page.goto('/org/competitor/repositories');

  // Should see 403 error page
  await expect(page.getByText(/forbidden|access denied/i)).toBeVisible();

  // Try direct API call
  const response = await page.request.get('/api/repositories?orgId=competitor');
  expect(response.status()).toBe(403);
});

test('viewer cannot perform admin actions', async ({ page }) => {
  // Login as Bob (viewer in ACME)
  await loginAs(page, 'bob@acme.com');

  await page.goto('/org/acme/settings');

  // Settings page should show but not allow edits
  await expect(page.getByText(/you don't have permission/i)).toBeVisible();

  // Try direct API call to update settings
  const response = await page.request.patch('/api/v1/organizations/acme/settings', {
    data: { features: { premium: true } }
  });
  expect(response.status()).toBe(403);
});
```

---

## Rollout Plan

### Week 1-2: Phase 1 Implementation
- Day 1-2: Create authorization middleware
- Day 3-5: Apply to all API routes
- Day 6-7: Apply to all server actions
- Day 8-10: Write integration tests

### Week 3-4: Phase 1 Testing & Deployment
- Day 11-12: Run integration tests, fix issues
- Day 13-14: E2E testing
- Day 15: Deploy to staging
- Day 16-17: Staging validation
- Day 18: Deploy to production (with rollback plan)

### Week 5-8: Phase 2 Implementation
- Week 5: Database migrations + proto updates
- Week 6: Update backend services
- Week 7: Create DAL layer
- Week 8: Testing & deployment

### Week 9-12: Phase 3 Implementation
- Week 9: RLS policies
- Week 10: Audit logging
- Week 11: Testing
- Week 12: Deployment & monitoring setup

---

## Success Metrics

### Security Metrics
- ✅ Zero cross-organization data access incidents
- ✅ All API endpoints return 403 for unauthorized access
- ✅ 100% of write operations check permissions
- ✅ Audit log captures all denied attempts

### Code Coverage
- ✅ 80%+ unit test coverage for authorization logic
- ✅ 100% of API routes have integration tests
- ✅ Critical flows have E2E tests

### Performance
- ✅ Authorization checks add < 50ms latency
- ✅ RLS policies don't degrade query performance > 20%
- ✅ Audit logging doesn't block request path

---

## Risks & Mitigation

### Risk 1: Breaking Existing Functionality
**Mitigation**:
- Phased rollout with feature flags
- Comprehensive test suite before deployment
- Rollback plan for each phase
- Staging environment validation

### Risk 2: Performance Degradation
**Mitigation**:
- Add database indexes for membership queries
- Cache membership lookups (with TTL)
- Load test before production
- Monitor p99 latency

### Risk 3: Complex Migration of Existing Data
**Mitigation**:
- Test migrations on copy of production data
- Provide clear backfill logic for organization_id
- Handle edge cases (orphaned data)
- Manual review of ambiguous cases

### Risk 4: Incomplete Coverage
**Mitigation**:
- Audit all API routes before starting
- Use linting rules to enforce authorization
- Code review checklist for new endpoints
- Periodic security audits

---

## Related Documentation

- [322-rbac.md](./322-rbac.md) - RBAC testing strategy
- [323-keycloak-realm-roles.md](./323-keycloak-realm-roles.md) - JWT role extraction
- [319-onboarding.md](./319-onboarding.md) - Onboarding flow (where this was discovered)
- [services/www_ameide_platform/features/navigation/README.md](../services/www_ameide_platform/features/navigation/README.md) - Navigation RBAC

---

## Appendix: Code Examples

### Example: Protected API Route
```typescript
// app/api/protected-resource/route.ts
import { NextResponse } from 'next/server';
import { withAuth, type AuthorizationResult } from '@/lib/auth/middleware';

async function handleGet(_request: Request, auth: AuthorizationResult) {
  // Business logic runs only after auth passes
  return NextResponse.json({ user: auth.session.user.id });
}

export const GET = withAuth(handleGet, {
  requiredRoles: ['admin', 'contributor'],
  orgIdSource: 'query',
});
```

### Example: Using DAL
```typescript
// app/api/repositories/route.ts
import { RepositoryDAL } from '@/lib/dal/graph-dal';
import { getSession } from '@/app/(auth)/auth';

export async function GET(request: NextRequest) {
  const session = await getSession();
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { searchParams } = new URL(request.url);
  const orgId = searchParams.get('orgId');

  // ✅ DAL automatically enforces membership
  const dal = new RepositoryDAL(session);
  const repositories = await dal.list(orgId);  // Throws if not member

  return NextResponse.json({ repositories });
}
```

---

**Last Updated**: 2025-10-30
**Status**: 🔴 Critical - Requires immediate action
**Next Review**: After Phase 1 completion
