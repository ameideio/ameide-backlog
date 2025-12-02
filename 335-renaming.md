# Domain Renaming and Proto Restructuring

**Status**: Completed
**Date**: 2025-11-01
**Impact**: Breaking changes to proto APIs, SDK, services
**Migration Type**: Domain consolidation and renaming

---

## Executive Summary

Major restructuring of the proto architecture to consolidate domains and improve naming clarity. This migration renames key domains, consolidates overlapping services, and establishes a cleaner architectural boundary between platform and business layers.

## Key Changes

### 1. Domain Renames

| Old Domain | New Domain | Scope |
|------------|------------|-------|
| `initiative` | `transformation` | Business transformation tracking |
| `chat` | `threads` | Messaging/conversation management |
| `metamodel` + `repository` | `graph` | Unified element and repository management |

### 2. Service Consolidation

| Old Services | New Service | Description |
|--------------|-------------|-------------|
| `ElementService` (metamodel) + `RepositoryService` | `GraphService` | Repository-scoped element operations |
| `InitiativeService` | `TransformationService` | Business transformation management |
| `ChatService` | `ThreadsService` | Thread-based conversations |

### 3. Architectural Layers

```
Platform Layer:
├── platform/         (users, organizations, teams, roles, invitations, tenants)
└── threads/         (messaging infrastructure)

Business Layer:
├── graph/           (repositories, elements, relationships, nodes)
├── transformation/  (business transformations, ex-initiatives)
├── governance/      (governance processes)
├── agents/          (AI agents + runtime)
└── workflows/       (workflows engine + runtime)
```

---

## Detailed Changes

### A. Initiative → Transformation

**Proto Changes:**
```protobuf
// OLD: ameide_core_proto/initiative/v1/initiative.proto
enum InitiativeStage {
  INITIATIVE_STAGE_UNSPECIFIED = 0;
  INITIATIVE_STAGE_DISCOVERY = 1;
  // ...
}

service InitiativeService {
  rpc CreateInitiative(...) returns (...);
}

// NEW: ameide_core_proto/transformation/v1/transformation.proto
enum TransformationStage {
  TRANSFORMATION_STAGE_UNSPECIFIED = 0;
  TRANSFORMATION_STAGE_DISCOVERY = 1;
  // ...
}

service TransformationService {
  rpc CreateTransformation(...) returns (...);
}
```

**Impact:**
- All enum values prefixed with `INITIATIVE_` changed to `TRANSFORMATION_`
- Service renamed from `InitiativeService` to `TransformationService`
- All RPC methods renamed (CreateInitiative → CreateTransformation, etc.)
- Field names in messages changed from `initiative_id` to `transformation_id`

### B. Metamodel + Repository → Graph

**Consolidation Strategy:**
```
OLD Architecture:
- metamodel.v1.ElementService → Element CRUD operations
- repository.v1.RepositoryService → Repository management
- Elements as first-class entities with cross-repository queries

NEW Architecture:
- graph.v1.GraphService → Repository + repository-scoped element operations
- Elements are repository-scoped (no cross-repository queries)
- No direct element CRUD - elements accessed through repositories
```

**Proto Structure:**
```protobuf
// NEW: ameide_core_proto/graph/v1/graph.proto
message Element {
  string id = 1;
  string tenant_id = 2;
  string organization_id = 3;
  string repository_id = 4;  // Elements are repository-scoped
  ElementKind kind = 5;
  string type_key = 7;
  // ... (consolidated from metamodel)
}

message Repository {
  ResourceMetadata metadata = 1;
  string organization_id = 2;
  string key = 3;
  string name = 4;
  // ... (from old repository.proto)
}

// NEW: ameide_core_proto/graph/v1/graph_service.proto
service GraphService {
  // Repository operations
  rpc CreateRepository(...) returns (...);
  rpc ListRepositories(...) returns (...);

  // Repository-scoped element queries (NO element CRUD)
  rpc ListRepositoryElements(...) returns (...);

  // Node/tree operations
  rpc CreateNode(...) returns (...);
  rpc GetTree(...) returns (...);

  // Assignment operations
  rpc AssignElement(...) returns (...);
}
```

**Key Architectural Decision:**
- **Elements are NOT first-class entities** - they exist within repository context
- **No ElementService** - deliberate choice to avoid global element operations
- **Repository-centric model** - repositories are the primary containers

### C. Chat → Threads

**Simple Rename:**
```protobuf
// OLD: ameide_core_proto/chat/v1/
service ChatService { }

// NEW: ameide_core_proto/threads/v1/
service ThreadsService { }
```

### D. ElementKind Evolution

**From Generic to Specific:**
```protobuf
enum ElementKind {
  // Legacy generic kinds (being phased out)
  ELEMENT_KIND_NODE = 1;
  ELEMENT_KIND_VIEW = 3;
  ELEMENT_KIND_DOCUMENT = 4;

  // New notation-specific kinds
  ELEMENT_KIND_ARCHIMATE_ELEMENT = 20;
  ELEMENT_KIND_ARCHIMATE_VIEW = 21;
  ELEMENT_KIND_BPMN_ELEMENT = 30;
  ELEMENT_KIND_BPMN_VIEW = 31;
  ELEMENT_KIND_DOCUMENT_MD = 10;
  ELEMENT_KIND_DOCUMENT_PDF = 11;
  // ...
}
```

**Design Decision:**
- Removed `ontology` field entirely
- Element kind now encodes both notation and type (e.g., `ARCHIMATE_VIEW`)
- `type_key` provides subtype within kind

---

## TypeScript SDK Updates

### Client Changes:
```typescript
// OLD
class AmeideClient {
  readonly elementService: Client<typeof metamodelService.ElementService>;
  readonly repositoryService: Client<typeof repositoryService.RepositoryService>;
  readonly initiativeService: Client<typeof initiativeService.InitiativeService>;
  readonly chatService: Client<typeof chatService.ChatService>;
}

// NEW
class AmeideClient {
  readonly graphService: Client<typeof graphService.GraphService>;
  readonly transformationService: Client<typeof transformationService.TransformationService>;
  readonly threads: Client<typeof threadsService.ThreadsService>;
}
```

### Query Pattern Changes:
```typescript
// OLD - Cross-repository element queries
export async function listAllElements(
  client: AmeideClient,
  query: { tenantId: string; typeKeys?: string[] }
): Promise<Element[]> {
  return client.elementService.listElements(query);
}

// NEW - Repository-scoped queries only
export async function listAllElements(
  client: AmeideClient,
  query: {
    tenantId: string;
    repositoryId: string;  // REQUIRED
    nodeIds?: string[];
  }
): Promise<Element[]> {
  return client.graphService.listRepositoryElements(query);
}
```

---

## Migration Impact

### Breaking Changes:
1. **All proto imports must be updated**
   - `@ameide/core-proto/initiative` → `@ameide/core-proto/transformation`
   - `@ameide/core-proto/metamodel` → removed
   - `@ameide/core-proto/repository` → `@ameide/core-proto/graph`

2. **SDK client initialization**
   - Must update to use new service names
   - Element queries now require `repositoryId`

3. **Frontend code**
   - All references to "initiative" must change to "transformation"
   - Element queries must provide repository context
   - No more cross-repository element searches

4. **Backend services**
   - Initiative service implementations → Transformation service
   - Element service handlers → Must use graph service patterns
   - Repository service → Graph service

### Files Affected:
- **Proto definitions**: ~15 files deleted, ~10 files created/modified
- **TypeScript SDK**: ~20 files modified
- **Frontend**: ~100+ files need updating (based on 303-elements.md estimate)
- **Backend services**: All service implementations need updating

---

## Rationale for Changes

### Why "Transformation" instead of "Initiative"?
- Better reflects the business purpose (transformation tracking)
- Aligns with enterprise architecture terminology
- Clearer intent than generic "initiative"

### Why consolidate to "Graph"?
- Eliminates confusion between metamodel and repository
- Reflects the actual data structure (graph of elements and relationships)
- Simplifies the mental model

### Why repository-scoped elements?
- Provides clear ownership boundaries
- Simplifies access control
- Prevents unbounded cross-repository queries
- Aligns with actual usage patterns

### Why no ElementService?
- Elements are not autonomous entities
- They exist within repository context
- Direct CRUD would bypass repository governance
- Queries are always repository-scoped in practice

---

## Status

✅ **Completed Tasks:**
1. Renamed initiative → transformation in all protos
2. Deleted metamodel proto directory
3. Created unified graph domain with consolidated types
4. Created GraphService with repository-scoped operations
5. Removed chat proto (conflicts with threads)
6. Updated TypeScript SDK exports
7. Fixed SDK client to use new services
8. Updated element query functions to require repositoryId
9. Successfully compiled TypeScript SDK

⚠️ **Not Implemented:**
- Element CRUD operations (deliberate choice)
- Cross-repository element queries (deliberate choice)
- Relationship query operations (not needed yet)

---

## Comparison with 303-elements.md Plan

The 303-elements.md document proposed a more radical "everything is an element" approach. Our implementation is more conservative:

| Aspect | 303-elements.md Plan | Our Implementation |
|--------|----------------------|-------------------|
| Model | Element-centric (flat) | Repository-centric (hierarchical) |
| Element CRUD | Full CRUD service | No direct CRUD |
| Cross-repo queries | Supported | Not supported |
| Containment | Via relationships | Via node assignments |
| Repositories | Could be elements | First-class containers |

This represents a pragmatic middle ground that achieves consolidation without the full flattening proposed in the original plan.

---

## Implementation Status (2025-11-01)

### ✅ Completed
- **Frontend:** ~200 files updated (features, routes, components, stores)
- **Helm Charts:** All charts/values/environments renamed
- **TypeScript:** 133 of 226 errors fixed (59% reduction)

### 📁 Key Renames
```
/features/chat/              → /features/threads/
/features/repository/        → /features/graph/
/features/initiatives/       → /features/transformations/
[initiativeId]              → [transformationId]
InitiativeSectionShell      → TransformationSectionShell
```

### ⚠️ Remaining Work
- 93 TypeScript errors (mostly type mismatches)
- Database schema updates needed
- CI/CD pipeline updates
- Full test suite run required

## Next Steps

1. ~~**Frontend Migration**: Update all frontend code to use new domain names~~ ✅ DONE
2. **Backend Services**: Implement remaining GraphService handlers
3. **Database Migration**: Update tables to match new proto structure
4. **Documentation**: Update all API docs and examples
5. **Testing**: Comprehensive testing of repository-scoped operations
6. **Clean up**: Remove deprecated proto files and old generated code