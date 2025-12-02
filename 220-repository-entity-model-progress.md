# Implementation Progress (2025-10-20)

**Repository Service MVP**
- ✅ Created standalone `@ameide/graph` Connect service exposing RepositoryAdmin/Tree/Query/Assignment RPCs.
- ✅ Backed service with PostgreSQL tables (`repositories`, `graph_nodes`, `graph_assignments` – Alembic revisions 010 & 011).
- ✅ Handlers now execute real SQL queries, returning typed protobuf messages; create/update write to Postgres.
- ✅ Portal API routes use `AmeideClient` via Connect transport (defaulting to Envoy gateway); graph UI reads live data.
- ✅ Envoy Gateway GRPCRoute publishes graph admin/tree/query services at `api.{env}.ameide.io`.

**Artifacts & Stats**
- ✅ Artifact summaries aggregated from graph assignments with type/node/search filters.
- ✅ Repository stats computed via SQL aggregations for node/assignment counts and artifacts by kind.
- ⚙️ Lifecycle parsing normalizes ad-hoc strings to proto enums.

**Mock Infrastructure**
- ✅ SDK mock transport seeds repositories/nodes/artifacts for local/testing scenarios.
- ✅ React components still functional while service interaction moves to Postgres-backed APIs.

## Next Steps
1. **Command Enhancements**
   - Persist graph events (append-only/outbox) for downstream projections.
   - Implement full mutation support (nodes, assignments, placement rules) with optimistic concurrency.
2. **Artifact Integration**
   - Join with real artifact registry to surface titles, lifecycle state, owners.
   - Enforce placement/gov rules prior to assignment.
3. **Projection Foundations**
   - Introduce event store + projection workers to decouple reads/writes.
   - Emit sync hooks to knowledge graph/search pipelines.
4. **Portal Alignment**
   - Remove remaining mock tree data; fetch classification nodes via API.
   - Implement artifact detail/draft journeys on top of service.
5. **Ops & Observability**
   - Finalize Envoy deployment (service mesh routing, health probes, TLS) and ship Helm chart for `@ameide/graph`.
   - Add metrics/logging dashboards and alerting.
