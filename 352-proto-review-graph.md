# 352 – Graph Proto Review Remediation

## Context
- Align the graph proto definitions with the actual service behaviour.
- Close the gaps called out during the proto review (graph service stubs, SDK/frontend assumptions, repository element metadata).

## Tasks
- [x] Restore element CRUD + query handlers so all GraphService RPCs are implemented.
- [x] Implement relationship CRUD/query handlers (including neighbourhood traversal).
- [x] Align repository element bootstrap with SDK expectations (`type_key = "graph"`, metadata).
- [x] Extend element listing filters to honour `element_types`.
- [x] Ensure clients provide or the service derives tenant/org context without breaking existing flows.
- [x] Update SDK/frontend adapters to match the corrected contracts where necessary.
- [x] Wire node CRUD/tree operations (create/update/move/delete/getTree/listChildren).
- [x] Implement element assignment handlers (assign/unassign/list) and node search/stats queries.
- [x] Add validation + event publishing scaffolding (buf.validate hints, outbox wiring).
- [x] Add / update automated tests covering the new behaviour.
- [ ] Stand up an outbox processor / event subscriber feeding governance + workflow services and document the contract.
- [ ] Publish regenerated SDKs (TS/Python/Go) after buf.validate changes and coordinate adoption windows with Org/Threads teams.
- [ ] Align graph visibility semantics with organizations/threads before exposing new enums or filters.
- [ ] Introduce explicit tenant/org identifiers on persisted graph entities (align with threads/org multi-tenant hardening).
- [ ] Provide SDK helpers and docs for consistently populating `RequestContext` and repository metadata across domains.
- [ ] Design structured metadata/annotation support for elements and relationships (parity with threads message parts) and surface in SDKs.
- [ ] Document access/visibility semantics (private/org/system) ahead of potential shared visibility expansion with Threads/Organizations.

## Progress Log
- Restored GraphService element + relationship handlers (CRUD, list, versions, neighbourhood) and wired shared helpers in `services/graph/src/graph/service.ts`.
- Normalised repository bootstrap metadata/type keys and updated the SDK/frontend adapter to pass org-scoped context; added plural type filtering support.
- Validated with `pnpm --filter @ameide/graph test` and `pnpm --filter @ameide/graph typecheck`; still need focused behavioural tests for the new handlers.
- Implemented repository node CRUD/move/tree operations with materialized-path updates and sibling ordering safeguards.
- Added element assignment management, node search, and repository statistics endpoints to close remaining GraphService gaps.
- Introduced buf.validate constraints across the graph proto and wired mutation handlers to publish graph events plus enqueue repository outbox entries.
- Backfilled integration coverage for node moves, assignment ordering, search, and stats, asserting emitted events and outbox change types.
- TODO: Coordinate cross-service event consumer rollout and SDK regeneration windows.

## Additional follow-ups

- Define shared event schema field names (`tenant_id`, `organization_id`, `repository_id`, `resource_kind`) for outbox consumers and update docs.
- Share RequestContext builders via the SDK (blocks org/threads backlog items) and update graph handlers/tests accordingly.
- Add buf.validate-driven runtime guards for relationship/node role enums once visibility taxonomy settles with Org/Threads stakeholders.
- Benchmark graph outbox volume and size to set retention/cleanup strategy before enabling downstream pipelines.
- Align repository/node slug validation with the organization key helper so identifiers stay consistent across services.
- Surface organization visibility + feature toggle context in Graph queries (e.g., include feature flags in repository/node views) to support cross-domain UIs.
- Enrich emitted graph events with organization metadata required by Threads/Org consumers (visibility, roles) and verify downstream projections.
- Plan cache invalidation hooks that react to organization lifecycle events so navigation trees stay current without aggressive polling.
