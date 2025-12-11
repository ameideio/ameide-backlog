# 028 – Public API Layer (REST + Graph)

Status: **In Progress**  
Author: platform-architecture  
Created: 2025-07-23
Updated: 2025-07-24

## Motivation

We now have:

* `core-proto` – wire DTOs (service-to-service contracts).
* `core-storage` – persistence adapters (graph, sql, blob).

What is **missing** is the *public* HTTP/gRPC frontier that external clients
and our own web-UI talk to. It should aggregate graph queries and traditional
REST resources into a single, versioned contract while re-using
`core-proto` types.

## Goals

1. Define a **versioned API specification** (OpenAPI 3.1 + GraphQL schema).
2. Expose unified resources for Users, Agents, Workflows, Storage (objects)
   and Graph queries.  Handlers map **core-proto** DTOs → **core-domain**
   models → persist via **core-storage** repositories.
3. Make spec the single source for API client generation (Python/TypeScript).
   Note: This generates a low-level API client (`ameide-api-client`), not the
   full Platform SDK (`ameide-sdk`) described in backlog/500-platform-sdk.md.
4. Keep implementation thin (FastAPI + Strawberry GraphQL) and fully typed via
   `core-proto` models.

## Scope

### REST Resources (OpenAPI)

| Resource | Description |
|----------|-------------|
| `/agents/` | CRUD agent definitions, start/run/interrupt, streaming SSE |
| `/workflows/` | CRUD workflows models, launch executions |
| `/executions/` | Unified list for agent & workflows runs |
| `/transformations/` | Transform models between formats (BPMN→Temporal, etc.) |
| `/storage/objects/` | Upload / download artefacts (presigned URLs) |

### Graph Endpoint

`/graph/query` – accepts Cypher/Gremlin via POST returning tabular or graph-json
using `core-proto.graph` DTOs.

### Streaming & WebSocket

`/agents/{run_id}/stream` – Server-Sent Events with `AgentStreamEvent` payload.

## Non-Goals

* GUI components – they consume the API.
* SDK implementation details – handled by openapi-generator.

## Work-Streams

1. **API Spec Authoring**  
   * Author `api/openapi.yaml` by referencing `$ref` to JSON-Schema generated
     from `core-proto` models (use `pydantic.schema_json_of`).  
   * Add GraphQL schema in `api/graphql/schema.graphql` with types mapped to
     `core-proto.graph` nodes/edges.

2. **Code Generation & Package**  
   * Generate FastAPI routes stubs with `datamodel-code-generator --input openapi.yaml`.  
   * Create `packages/ameide_core-api` that simply imports the generated stubs and
     wires them to service layer implementations.

3. **Versioning & Compatibility**  
   * Follow semver; path prefix `/v{major}`.  
   * Snapshot tests – breaking change detection via `openapi-diff` in CI.

4. **Client Libraries**  
   * Add GitHub Action that publishes PyPI (`ameide-api-client`) and npm 
     (`@ameide/api-client`) packages on tag push – generated from OpenAPI.
   * Note: The full Platform SDK (`ameide-sdk`) is a separate, richer package
     that uses this API client internally.

5. **Security**  
   * Use OAuth2 RFC 6749 bearer scheme in spec; FastAPI depends on
     `fastapi-users[sqlalchemy]` for auth implementation.

## Acceptance Criteria

1. `api/openapi.yaml` and `schema.graphql` committed, validated in CI.
2. `packages/ameide_core-api` serves spec via `/openapi.json` & `/graphql` routes.
3. End-to-end test spins API, hits REST & Graph endpoints, asserts 200.
4. Auto-generated API clients pass their own compile-time tests.
5. All API schemas and routes are generated from `core-proto` definitions.

## Timeline (indicative)

| Week | Deliverable |
|------|-------------|
| W33 | Draft OpenAPI with refs to proto schemas |
| W34 | FastAPI scaffold + GraphQL schema |
| W35 | SDK generation, CI integration, security layer |

## Implementation Progress

### Completed (2025-07-24)

1. **Core API Package Structure** ✓
   - Created `/packages/ameide_core-api/` with full project setup
   - Route modules for graphs, executions, users, workflows, agents, transformations
   - Service layer foundation with dependency injection
   - API generation scripts moved to `/packages/ameide_core-api/api-generation/`

2. **Initial Route Implementation** ✓
   - Basic CRUD operations for graphs, executions, users
   - Service interfaces defined with graph pattern
   - Proto model integration started

### In Progress

1. **Extended Route Implementation** (tracked in 032-api-extended-routes.md)
   - Workflow management endpoints
   - Agent configuration and execution
   - Transformation pipeline operations

2. **OpenAPI Specification**
   - Schema generation from proto models
   - Endpoint documentation
   - Request/response examples

### Remaining

1. **GraphQL Endpoint** - `/graph/query` with Cypher/Gremlin support
2. **Streaming/WebSocket** - SSE for agent executions
3. **Security Layer** - OAuth2 implementation
4. **Client Generation** - API client packages for Python/TypeScript

---

_Design for deletion_: OpenAPI + GraphQL schemas are single sources we can
regenerate code from in < 30 min.
