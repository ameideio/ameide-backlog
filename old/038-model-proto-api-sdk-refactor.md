# 038 – Unified Model/Proto/API/SDK Refactor

> “One contract to rule them all, one pipeline to generate them, **zero** hand-written drift.”

This backlog item captures the **long-horizon** refactors required to completely align the core-domain models, protocol layer, HTTP/gRPC API surface and developer SDK around a *single, version-controlled contract*.

## Motivation

The current code base evolved organically and now contains:

* **Duplicate mappers** (proto ↔ domain ↔ storage) maintained by hand.
* **Hand-written clients** (Python SDK) that occasionally lag behind the API.
* **Multiple transport contracts** (REST, SSE, soon gRPC) that are not generated from the same schema.

This refactor eliminates drift, reduces boiler-plate and unlocks faster product evolution.

## High-level Goals

1. **Single Source of Truth** – `core-proto` definitions are published as a versioned package and drive code-generation for every other layer.
2. **Contract-First Transport** – Generate both **gRPC** stubs and **REST/OpenAPI** specs from the same proto schemas.
3. **Generated Clients** – SDK(s) (Python, TS) are fully generated from the OpenAPI / gRPC definitions. No more handwritten request code.
4. **Immutable Domain** – Domain models stay framework-free; mappers are generated, not written.
5. **Streaming & Artifacts** – WebSocket / gRPC streaming replaces ad-hoc SSE; `TranspiledPackage.publish()` pushes to OCI registry.

## Work-packages & Tasks

### 1. core-proto

| Task | Details |
|------|---------|
| P1 | Remove naïve timestamp pattern – keep timezone aware datetimes. |
| P2 | Split proto package into `ameide-core-proto` (contracts) and `ameide-core-proto-tools` (code-gen helpers). |
| P3 | Introduce `buf` or `protoc` build to emit: <br>• gRPC stubs (python / ts) <br>• OpenAPI (grpc-gateway or `protoc-gen-openapiv2`). |
| P4 | Publish artefacts to a public registry (PyPI, npm). Version according to SemVer + `CHANGELOG.md`. |

### 2. core-domain

| Task | Details |
|------|---------|
| P1 | Replace duplicate mapping utils with a generated mapper layer (use `datamodel-codegen` or `pydantic-gen`) that consumes the proto schema. |
| P2 | Introduce factory/DSL for constructing domain objects in tests. |
| P3 | Add property-based tests (`hypothesis`) that mutate `GraphDomain` & `Workflow` objects → must always satisfy invariants (no cycles, duplicate IDs, etc.). |

### 3. core-api

| Task | Details |
|------|---------|
| P1 | Adopt **gRPC** internally; expose REST via auto-generated gRPC-gateway (or FastAPI using OpenAPI spec). |
| P2 | Replace SSE endpoint with WebSocket *or* gRPC streaming. |
| P3 | On app start, validate that implementation <span/>matches proto version (break glass if mismatch). |
| P4 | Remove global mutable `_dependencies`; wire dependencies through FastAPI `lifespan` and typed DI container. |

### 4. core-sdk

| Task | Details |
|------|---------|
| P1 | Scrap handwritten `AmeideClient`; regenerate Python client from OpenAPI using `openapi-python-client` (or gRPC stub). |
| P2 | Implement `TranspiledPackage.publish()` to push artefacts to OCI/ORAS registry. |
| P3 | Provide convenience façade that wraps generated client with high-level helpers (optional). |

## Acceptance Criteria

* Running `make generate-contracts` regenerates proto stubs, REST routes, and SDKs **without manual edits**.
* Deleted all custom mapping code under `core-domain/agents|workflows|orchestration/mappers.py`.
* `core-sdk` contains *only* thin wrappers around generated clients.
* End-to-end tests (E2E + integration) pass using the new transport & client stack.
* Publishing pipeline pushes versioned proto & SDK artefacts on every tag.

## Non-Goals (for now)

* Migrating existing databases – handled by a later migration ticket.
* Refactoring storage layer (SQL/graph) – outside this backlog item, but must remain compatible.

## Timeline & Milestones

1. **v0 – groundwork (1 wk)** – proto timestamp fix, CI pipeline for buf/protoc.
2. **v1 – contracts published (2 wks)** – gRPC & OpenAPI artefacts released, SDK auto-generated.
3. **v2 – API transport swap (2 wks)** – API routes delegated to gRPC implementation; REST gateway online.
4. **v3 – domain convergence (1 wk)** – generated mappers + property tests green, old mappers deleted.
5. **v4 – artifact registry (1 wk)** – `publish()` implemented, docs updated.

## Risks / Mitigations

* **Schema churn** – Adopt strict versioning + change-log; provide compatibility layer during migration.
* **gRPC learning curve** – Provide samples, dev-container with buf + grpcurl.
* **Tooling fragmentation** – Standardise on `buf` for proto, `openapi-python-client` for REST, `connect-web` for TS.

---

> After this refactor **every new feature starts with the proto**, and the rest of the stack follows automatically – no more silent drift.
