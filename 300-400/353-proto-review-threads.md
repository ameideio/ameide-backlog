# Threads Proto Review & Refactors (Backlog 353)

## Context
- Derive follow-up changes from the threads proto review (bug fixes and hardening work).
- Keep a running log of what was implemented vs. what remains out of scope.

## Tasks
- [x] Fix threads messages API to return string role values (not numeric enums) for frontend consumers.
- [x] Cap inference history loading in `ThreadsService.SendMessage` to avoid unbounded prompt growth.
- [x] Design structured message parts & metadata in proto/service and align persistence.
- [x] Introduce explicit tenant scoping (`tenant_id`) across thread proto + persistence layers.
- [x] Extend `AppendMessagesRequest` to carry metadata so we can drop hardcoded user email fallback.
- [x] Document/update API semantics for auto thread creation (empty `thread_id`) and capture visibility follow-up.
- [ ] Ship database migrations (tenant_id / metadata / archived_at) across environments and rerun integration suites.
- [ ] Publish updated SDKs (TS/Python/Go) so downstream consumers receive the new proto surface without type drift.
- [ ] Add cross-tenant isolation tests (unit + service) and structured message ingestion regressions.
- [ ] Harmonize `RequestContext` helpers with Organization/Graph clients and centralize in SDK.
- [ ] Define shared visibility taxonomy with Org/Graph stakeholders before expanding enum values.
- [ ] Apply organization slug/tenant validation helpers when auto-creating threads to avoid divergent identifiers.
- [ ] Enrich thread events with organization + graph metadata (repository ids, visibility) for downstream analytics.
- [ ] Gate threaded experiences via organization feature toggles (SDK plumbing + frontend adoption).
- [ ] Plan caching strategy for thread list/detail queries that respects organization lifecycle events and graph membership changes.
- [ ] Adopt `buf.validate` annotations for threads proto (parity with graph/org) and regenerate descriptors.
- [ ] Introduce threads outbox + event publishing mirroring graph so workflows/governance can subscribe to message lifecycle events.
- [ ] Align archival/visibility behaviour with graph repositories (shared status enums + audits).
- [ ] Coordinate message metadata schema with graph assignments (linking artefacts to conversations) and document cross-service usage.

## Notes
- Update this file as tasks complete (checked) or get re-scoped.
- Link additional context or issues here if work splits out.
- Visibility taxonomy expansion remains open; revisit after stakeholder feedback on sharing modes.
- Ensure SDK releases stay in lock-step with Org/Graph so RequestContext + visibility semantics remain consistent.
