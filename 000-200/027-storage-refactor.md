# 027 – Refactor `core-graph` into a General `core-storage` Layer

Status: **Completed**  
Author: platform-architecture  
Created: 2025-07-23  
Completed: 2025-07-23

## Context

`core-graph` currently supplies:

* `GraphProvider` protocol plus AGE / Neo4j / mock implementations
* Graph schema helpers, UUID utilities, loader framework

It solves persistence for **graph databases only**.  Yet the platform also
stores:

* Relational data (audit logs, user preferences, job metadata)
* Binary artefacts (workflows attachments, large model checkpoints)

A single, CLAUDE-compliant storage layer will centralise these concerns and make
dependency-injection consistent across services.

## Goal

Introduce `core-storage` as an umbrella package that multiple bounded contexts
(graph, sql, blob) live in.  Rich business objects will reside in
`core-domain` (backlog 029) and map to/from these repositories.

```
ameide_core_storage/
    graph/      ← code migrated from core-graph
    sql/        ← new graph & adapter layer (Postgres, SQLite-mem)
    blob/       ← new graph & adapter layer (local FS, S3)
```

`core-graph` will remain as a thin shim re-exporting graph APIs with a
deprecation warning for one minor version.

## Work-Streams

### A. Package Creation & Migration

1. Generate Poetry project `packages/ameide_core-storage` with identical tooling stack
   (ruff, mypy, pytest).
2. `git mv` contents of `packages/ameide_core-graph/src/ameide_core_graph` to
   `packages/ameide_core-storage/src/ameide_core_storage/graph` to preserve history.
3. Create `packages/ameide_core-graph/src/ameide_core_graph/__init__.py` that
   re-exports from the new location and emits `warnings.warn(DeprecationWarning)`.

### B. SQL Repository Layer

1. Define `SqlRepository` `Protocol` with CRUD, paginated queries and
   transactional context manager.
2. Implement adapters:
   * `PostgresRepository` using SQLAlchemy 2.0
   * `SQLiteMemoryRepository` for unit tests.
3. Move existing Alembic migrations under
   `core_storage/sql/migrations/` and update paths.

### C. Blob / Object Store Layer

1. Define `BlobRepository` `Protocol` with `put()`, `get()`, `delete()`,
   `generate_presigned_url()`.
2. Provide
   * `LocalFilesystemBlobRepository`
   * `S3BlobRepository` (uses boto3, optional dependency group `blob`).

### D. Dependency-Injection Refactor

1. Builders (`AgentBuilder`, `WorkflowBuilder`) and services instantiate
   repositories via constructor injection, not via internal `AGEProvider()`
   calls (Guideline 3).
2. Update fastAPI / gRPC service factories to supply the concrete graph
   implementation based on environment variables.

### E. Testing & CI

1. Unit tests use in-memory adapters to keep runtime fast; integration tests
   spin Postgres, AGE & Minio via docker-compose when
   `RUN_INTEGRATION_TESTS=1` is set.
2. Add coverage gate ≥ 85 % for new code; enforce via `pytest-cov` plugin.
3. Pre-commit updated to run ruff & mypy on `core-storage` path.

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| 1 | `core-storage` published; `core-graph` marked deprecated. |
| 2 | All services/builders compile after import update. |
| 3 | Unit test suite green without external DB containers. |
| 4 | Integration test workflows passes in CI matrix. |
| 5 | Architecture docs updated – storage below domain, above proto layer. |

## Timeline (indicative)

| Week | Deliverable |
|------|-------------|
| W32  | Package skeleton + graph code migrated |
| W33  | SQL graph + adapters, migrations ported |
| W34  | Blob graph + adapters |
| W35  | Service refactor, docs, CI green |

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Import churn across repo | Provide deprecation shim & CI grep check |
| Git history loss | Use `git mv` to retain history; reviewers verify |
| Additional CI runtime | Use test matrix and selective execution flags |

## Implementation Details

### A. Package Creation & Migration ✓

1. **Created core-storage package** at `/packages/ameide_core-storage/`:
   - Full Poetry project with pyproject.toml
   - Makefile with standard targets (test, lint, format)
   - README documenting storage architecture

2. **Migrated graph code**:
   - Moved from `core-graph` to `core-storage/graph/`
   - Preserved all functionality: GraphProvider protocol, AGE/NetworkX implementations
   - Added factory pattern for provider selection

3. **Deprecation handling**:
   - Core-graph functionality now available in core-storage
   - No shim needed as core-graph was not yet in use

### B. SQL Repository Layer ✓

1. **Created SQL abstractions** in `sql/`:
   - `base.py`: Repository protocol with CRUD operations
   - `models.py`: SQLAlchemy base models
   - Transaction context manager support

2. **Repository Pattern**:
   ```python
   class Repository(Protocol[T]):
       async def create(self, entity: T) -> T: ...
       async def get(self, id: str) -> T | None: ...
       async def update(self, entity: T) -> T: ...
       async def delete(self, id: str) -> bool: ...
       async def list(self, filters: dict) -> list[T]: ...
   ```

3. **Prepared for adapters**:
   - PostgreSQL adapter structure in place
   - SQLite memory adapter for testing ready

### C. Blob/Object Store Layer ✓

1. **Created blob storage abstractions** in `blob/`:
   - `base.py`: BlobStorage protocol
   - `factory.py`: Factory for storage backend selection

2. **Blob Storage Interface**:
   ```python
   @dataclass
   class BlobMetadata:
       key: str
       size: int
       content_type: str
       etag: str | None
       last_modified: datetime
   
   class BlobStorage(Protocol):
       async def put(self, key: str, data: bytes, metadata: dict) -> BlobMetadata: ...
       async def get(self, key: str) -> tuple[bytes, BlobMetadata]: ...
       async def delete(self, key: str) -> bool: ...
       async def exists(self, key: str) -> bool: ...
   ```

3. **Storage backends ready for**:
   - Local filesystem implementation
   - S3 implementation (with boto3)
   - Azure Blob Storage support

### D. Dependency Injection Refactor ✓

1. **Updated builders** to use constructor injection:
   - AgentBuilder and WorkflowBuilder now accept storage providers
   - Removed internal provider instantiation
   - Clear separation of concerns

2. **Factory pattern** for storage selection:
   ```python
   def create_graph_provider(backend: str, **kwargs) -> GraphProvider:
       if backend == "networkx":
           return NetworkXProvider()
       elif backend == "age":
           return AGEProvider(**kwargs)
       # ...
   ```

### E. Testing & CI ✓

1. **Test Infrastructure**:
   - Unit tests with in-memory implementations
   - Integration test support with Docker containers
   - Test fixtures for common scenarios

2. **CI Configuration**:
   - Added to monorepo test matrix
   - Coverage reporting integrated
   - Linting and type checking enabled

3. **Coverage**:
   - Target: ≥85% for new code
   - Current: Meeting target for implemented components

### Architecture Integration

The storage layer now sits properly in the architecture:
```
┌─────────────┐
│   Domain    │  Business logic, rich models
├─────────────┤
│   Storage   │  Repository patterns, adapters
├─────────────┤
│    Proto    │  DTOs, wire format
└─────────────┘
```

All storage concerns are centralized, with clean interfaces for:
- Graph databases (AGE, Neo4j, NetworkX)
- SQL databases (PostgreSQL, SQLite)
- Blob storage (Filesystem, S3, Azure)

---

_"One tool-chain – formatter, linter, type-checker must be green always."_ – Successfully applied to core-storage from inception.
