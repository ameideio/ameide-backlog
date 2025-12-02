# Transformation Initiatives Technology Architecture

## Technology Matrix
| Layer | Choice | Rationale | Integration Notes |
| --- | --- | --- | --- |
| Backend runtime | Python 3.11 (FastAPI + gRPC via ConnectRPC) | Aligns with graph services; shared middleware | New `services/transformation_transformations` service reuses auth, tenancy, tracing stack |
| Data store | PostgreSQL 15 | Reuses tenant DB, supports JSONB + temporal queries | Tables defined in `162-transformation-transformations-information-architecture.md`; references graph assignment IDs for scenarios |
| Messaging | NATS JetStream / Kafka | React to graph governance & artifact events | Subscribe to `graph.*`, emit `transformation.*` updates (role changes, milestone status) |
| Cache | Redis (clustered) | Store transformation dashboard snapshots, role resolution, scenario filters | Keys scoped by tenant and transformation; invalidated on mutation events |
| Frontend | Next.js 15 + React 19 | Shares SPA shell/components with graph | Initiative workspace screens live under `/transformations/:transformationId` |
| Client SDK | `@ameideio/ameide-sdk-ts` | Typed ConnectRPC clients | Extend SDK with transformation service stubs (`InitiativeService`, `InitiativeWorkspaceService`) |
| Workflow automation | Temporal / Argo Workflows | Drives periodic reminders, survey prompts, recertification flows | Workers consume transformation events and graph governance outcomes |
| Directory integration | Future Corporate Directory API | Source of truth for people & roles | Sync job maps directory principals to transformation role assignments |

## Service Architecture
- **InitiativeService (gRPC)**: CRUD for transformations, milestones, role assignments, artifact links; enforces tenancy, soft-delete rules, and directory lookups.
- **InitiativeWorkspaceService (gRPC for writes, GraphQL for aggregated reads)**: Provides board/table/roadmap datasets by joining transformation data with graph APIs. GraphQL is restricted to read-only fan-in queries; mutations stay on gRPC.
- **InitiativeEventsProcessor**: Listens to graph governance events (check runs, approvals, releases) and updates transformation read models. Emits `transformation.status.updated`, `transformation.role.assignment.updated`, `transformation.milestone.progressed`.
- **Reminder/Survey Worker**: Temporal workers trigger survey prompts, role reminders, recertification tasks based on milestone dates and governance status.

## API Design Considerations
- All RPCs receive `RequestContext` (tenant_id, user_id, request_id); operations validate graph ownership relationship.
- Role assignment endpoints enforce uniqueness and consult directory data when available; respond with normalized principal info.
- Artifact linking endpoints validate artifact/node existence via graph APIs and persist the canonical `assignment_id` (effective windows/scenario tags remain on graph assignments); transformation-specific metadata lives alongside the link.
- Workspace queries support As-is/To-be/Scenario filters by projecting graph assignment windows via the stored `assignment_id` and transformation metadata. Impacts rely on Graph API (graph `model_edges`).
- Read-only endpoints return governance status fields (required checks pending, approvals outstanding, queue position) retrieved from graph services.

## Data Synchronization
- Repository governance events (`governance.check_run.*`, `graph.transition.*`, `graph.release.published`) drive transformation status updates and refresh linked assignment metadata (checks, approvals, releases).
- Directory sync job resolves principal metadata; conflicts (e.g., missing user) trigger alert events and mark assignments as stale.
- Scheduled jobs recompute KPI aggregates (cycle time, completion %) for dashboards; results cached in Redis.

## Security & Compliance
- RBAC middleware ensures only authorized tenant members access transformation data; fine-grained scopes (transformations.admin / transformations.editor / transformations.viewer) map to graph permissions.
- Auditing captures all role assignments, milestone changes, and artifact link modifications with actor, timestamp, and diff.
- Soft-delete with legal-hold flags ensures regulatory retention; purge operations gated behind governance admin role.
- Sensitive integrations (directory, notifications) store credentials in Vault; rotation orchestrated via Temporal workflows.

## Observability
- OpenTelemetry instrumentation traces transformation RPCs, event processors, and Temporal workflows.
- Metrics: board query latency, milestone throughput, role assignment churn, event processing lag, survey completion rate.
- Logs: structured JSON with tenant_id, transformation_id; log forwarding to central logging stack (Loki or Elastic).

## Deployment
- Service packaged via `uv`/`poetry` container; Helm chart configures Postgres DSN, Redis, event topics, Temporal namespace, directory API endpoint.
- Local dev (Tilt) spins up transformation service, graph mocks, Temporal worker, directory stub, Next.js client.
- CI/CD: schema diff checks, API contract tests, event consumer integration tests, UI e2e flows (Playwright) verifying board/roadmap behavior.

## Dependencies & Interfaces
- **Repository Service**: artifact lookup, governance status, release info.
- **Directory API** (future): principal metadata, group membership.
- **Notification Service**: deliver role-based reminders, release announcements.
- **Analytics pipeline**: consumes transformation events for portfolio-level dashboards.

## Next Steps
- Finalize gRPC proto definitions for `InitiativeService` and `InitiativeWorkspaceService`.
- Define event schemas (CloudEvents or custom) for transformation notifications.
- Prototype Temporal workflows for reminder/survey automation.
- Establish directory sync contract with identity team (batch vs webhook).
