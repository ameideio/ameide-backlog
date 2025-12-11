# Technology Matrix
| Layer | Choice | Rationale | Integration Notes |
| --- | --- | --- | --- |
| Backend runtime | Python 3.11 (FastAPI + gRPC via ConnectRPC) | Matches existing Ameide core services; async SQLAlchemy support | New `services/core_graph` service reuses shared middleware for auth, tenancy, tracing |
| Data store | PostgreSQL 15 | JSONB + materialized path queries; tenant-safe isolation | Alembic migrations enable `ltree` + `btree_gist`; optional Row-Level Security per tenant |
| Messaging | NATS JetStream / Kafka (existing event bus) | Artifact lifecycle & assignment events | Emit `graph.assignment.*`, `governance.check_run.*`, `graph.transition.*`, `ruleset.updated` |
| Object cache | Redis (clustered) | Read-model caching (tree, stats) and workflows throttling | Cache tree snapshots, NodeOWNER inheritance docs, and ruleset digests |
| Frontend | Next.js 15 + React 19 + shadcn/ui | Aligns with platform SPA; SSR where needed | `RepositoryBrowser` and transformation surfaces consume SDK clients |
| Client SDK | `@ameideio/ameide-sdk-ts` | Typed clients generated from protobuf; consistent error handling | Expand to include graph services + lifecycle commands |
| Collaboration | Yjs (optional, per artifact) | Real-time editing for text/code artifacts | Agents and users share command bus origins; integrates with `ArtifactView` updates |
| Workflow automation | Temporal / Argo Workflows (existing) | Enforce lifecycle gates & recertification jobs | `TransitionQueue` worker evaluates approvals/checks before committing |
| Policy engine | OPA / Rego bundles + check registry | Externalize governance checks | Bundles versioned per tenant; registry locks check names/sources referenced by rulesets |
| Graph analytics | Neo4j or PostgreSQL `model_edges` table | Impact analysis & scenario planning | Event consumers populate edges with effective windows |

# Architecture Overview
```mermaid
flowchart LR
    subgraph Frontend
        UI[Next.js Repository Browser]
        InitiativesUI[Transformation Initiatives UI]
    end
    subgraph SDK
        TSClient[@ameideio/ameide-sdk-ts]
    end
    subgraph Services
        RepoSvc[Repository Service<br/>(Admin/Tree/Assignment/Query)]
        GovSvc[Governance Service<br/>(Owners/Approvals/Rulesets)]
        CheckSvc[Check Run Service<br/>(OPA / CI / SAST orchestration)]
        QueueSvc[Transition Queue Worker<br/>(Temporal)]
        ArtifactSvc[Artifact Service<br/>(event-driven + CRDT hybrids)]
    end
    subgraph Data
        PG[(PostgreSQL Repository Schema)]
        Redis[(Redis Cache)]
    end
    subgraph Bus
        NATS[(Event Bus)]
    end
    UI --> TSClient --> RepoSvc
    InitiativesUI --> TSClient
    RepoSvc --> PG
    RepoSvc --> Redis
    RepoSvc <--> ArtifactSvc
    RepoSvc --> NATS
    GovSvc --> NATS
    CheckSvc --> NATS
    QueueSvc --> NATS
    ArtifactSvc --> NATS
    NATS --> GovSvc
    NATS --> CheckSvc
    NATS --> QueueSvc
    GovSvc --> RepoSvc
    CheckSvc --> RepoSvc
    QueueSvc --> RepoSvc
```

# Backend Architecture
- **Service boundaries**:
  - `RepositoryService` keeps Admin/Tree/Assignment/Query APIs generated from `packages/ameide_core_proto/src/ameide_core_proto/graph/v1`.
  - `GovernanceService` manages NodeOWNERS, approval rules, check registry, rulesets, and resolves approver sets per transition.
  - `CheckRunService` schedules and records required checks, invoking OPA, CI, SAST, or manual attestations.
  - `TransitionQueueWorker` (Temporal) serializes lifecycle transitions, revalidates approvals/checks, then commits state.
  - `ArtifactService` continues owning artifact storage and lifecycle events consumed by graph projections.
- **Command bus alignment**: Lifecycle commands flow through a shared schema (`backlog/120-artifacts-north-star.md`). Governance and queue services subscribe to command events, update state, and publish outcomes (`graph.transition.completed`).
- **Storage implementation**: Async SQLAlchemy repositories cover core tables plus governance tables (owners, approvals, checks, releases, discussions, rulesets, check registry). Soft-delete fields (`deleted_at`, `legal_hold`) are enforced by default scopes. Transactions wrap node moves (updating `path`/`path_ltree`), check run persistence, release publication, and queue operations with idempotency keys.
- **Read models**: Redis caches tree snapshots, NodeOWNER inheritance, ruleset digests, check dashboards, and queue heads. Optional graph projections populate `model_edges` or Neo4j for impact analysis.
- **Event publishing**: Services emit `graph.assignment.*`, `graph.node.moved`, `graph.template.applied`, `governance.node_owner.*`, `governance.check_run.*`, `graph.transition.*`, `ruleset.updated`, and forwarded `artifact.lifecycle.transitioned`. Events include tenant context, ruleset version, approval sets, and diff payloads.
- **Database isolation**: Support optional PostgreSQL Row-Level Security policies keyed by `app.tenant_id` to satisfy regulated tenants while retaining application-level tenancy checks.
- **Protocol policy**: Repository surfaces only ConnectRPC/gRPC APIs (and their SDK bindings). Aggregated GraphQL read endpoints exist solely in the Transformation Initiatives workspace; graph write paths remain gRPC-only. Downstream consumers (including transformations) access graph data via the SDK.

# API Design Considerations
- **Context propagation**: All gRPC requests include `RequestContext` (`tenant_id`, `user_id`, `request_id`, `impersonation`). Governance APIs return the `ruleset_version` used, enabling audit replay.
- **Pagination & filtering**: Query APIs accept pagination, include filters for node type, artifact kind, lifecycle, transformation tags, effective windows, scenario tags, ruleset, approval state, check status, and queue state. Smart folders persist structured filters server-side.
- **Validation & errors**: Tree mutations guard against invalid node configurations. Lifecycle actions emit `FAILED_PRECONDITION` for unmet required checks or approvals, `ABORTED` if queue revalidation fails, and `PERMISSION_DENIED` when RBAC or rulesets block the actor.
- **Idempotency & provenance**: Lifecycle commands carry idempotency keys so duplicate submitters don’t create duplicate queue items. Responses include ruleset `{id, version, hash}` and check registry references to satisfy audit replays.
- **Webhooks & subscriptions**: Tenants can register webhooks for lifecycle, governance, release, and discussion events. SSE/WebSocket channels stream activity timelines, check run updates, queue status, and discussion threads.
- **Graph traversal APIs**: `RepositoryQueryService` exposes endpoints for impact analysis using `model_edges`, supporting queries like “systems impacted by capability X” or “transformations referencing deprecated artifacts.”
- **Check registry APIs**: Governance service publishes CRUD endpoints to register, retire, and audit canonical check definitions; `required_checks` references registry IDs and rejects unknown names.

# Frontend Architecture
- **Component hierarchy**: `RepositoryBrowser` orchestrates tree/list/grid views, assignment panels, releases, discussions, and governance sidebars. React Query fetches tree snapshots, NodeOWNER/resolution docs, required check status, and queue position concurrently.
- **Artifact detail surface**: Artifact drawer shows lifecycle state, owners, approvals, required check banner, queue status, releases tab (assets/downloads), discussions/ADR tab, and timeline of events (comments, check runs, approvals, releases).
- **Discussions & ADRs**: Markdown editors (Lexical/Tiptap) power discussions. ADR templates prefill decision records; accepted ADRs surface badges across graph nodes.
- **Time travel & scenarios**: Initiative workspace offers As-is/To-be/Scenario toggles that filter assignments by effective windows and scenario tags, with timeline scrubber for roadmap visualization.
- **Initiative integration**: Initiatives expose table/board/roadmap views (GitHub Projects style). Cards display linked artifacts, lifecycle, risk score, outstanding checks, and NodeOWNERS.
- **Collaboration cues**: Presence and comment threads use Yjs/WebSockets. Activity feed merges human and agent actions with origin metadata; watchers receive notifications via subscription service.

# Performance & Reliability
- **Indices**: Maintain composite indexes from migration `010_graph` plus governance tables (`idx_node_owner_node`, `idx_approval_scope`, `idx_check_runs_subject`, `idx_transition_queue_repo`). Evaluate GIST/ltree for `path` and partial indexes for queue lookups.
- **Caching strategy**: Cache tree structures, ownership inheritance, ruleset digests, check dashboards, and queue heads (60s TTL). Invalidate caches via events and background refresh.
- **Ruleset cache**: Propagate ruleset updates through Redis pub/sub to invalidate per-tenant caches immediately.
- **Scaling**: Repository, governance, and check services scale horizontally with asyncio + uvloop. Queue workers shard by graph and priority to avoid head-of-line blocking; graph projection consumers run separately to protect core throughput.
- **Resilience**: Enforce idempotency keys (request IDs, queue item IDs) and duplicate guards (unique queue index). Circuit breakers guard artifact-service, OPA, CI integrations. Queue workers retry with exponential backoff; dead-letter queue captures failed transitions for manual remediation.

# Security & Compliance
- **RBAC enforcement**: Middleware maps `RequestContext` roles to fine-grained permissions (`graph.admin`, `graph.editor`, `governance.reviewer`, `ruleset.manager`). High-risk operations (ruleset changes, queue overrides, release deletions) require MFA-verified sessions.
- **Audit logging**: Immutable logs capture lifecycle transitions, approvals, check runs (with results), ruleset versions, release publications, discussion decisions, and survey responses. Logs replicate to centralized compliance store.
- **Data residency**: Tenant region drives routing to localized clusters; ruleset bundles and OPA policies mirror regionally. Releases stored in region-specific object storage with encryption-at-rest.
- **Secrets & token management**: Governance/check services store integration secrets (CI tokens, OPA bundle credentials) in Vault; rotation policies enforced via Temporal workflows.
- **Row-level security & soft delete**: Optional PostgreSQL RLS policies enforce `tenant_id` isolation for regulated tenants; APIs respect `deleted_at`/`legal_hold` tombstones with admin-only purge flows.

# Deployment & DevOps
- **Packaging**: Containers built via `uv`/`poetry`; Helm charts configure Postgres DSN, Redis, event topics, OPA bundle URIs, Temporal endpoints, and ruleset cache TTLs.
- **Local development**: Tilt spins up graph, governance, check services, Temporal test worker, OPA dev server, Neo4j (optional), and Next.js frontend. Migration bundle includes new governance tables and seed rulesets/owners.
- **CI/CD**: Pipelines run schema diff checks, proto compatibility, policy tests (OPA), synthetic approval flows, and queue integration tests. ArgoCD rolls out charts with canary rules for governance services.
- **Observability**: OTLP tracing, Prometheus metrics (API latency, queue depth, check run durations, ruleset cache hit rate), Loki logs, Grafana dashboards. Alerting on failed check runs, queue SLA breaches, and ruleset cache miss spikes.
- **Rollout order**:
  1. Required checks + CheckRun service
  2. NodeOWNERS & Governance service
  3. Transition queue worker
  4. Artifact releases & protected tags
  5. Rulesets + policy bundles
  6. Graph read model & time-scoped assignments
  7. Discussions, surveys, transformation planning views
