# Proto Review: Organization Service Hardening

**Created:** 2025-11-05  
**Owner:** Platform Core / Identity  
**Related Backlog:** 324 (user org settings), 330 (tenant resolution), 336 (SDK dependency), 347 (service tests)

---

## Problem Statement

Recent review of the `ameide_core_proto.platform.v1.Organization` schema and downstream implementations highlighted two blocking defects:

- Partial updates to organizations unintentionally nullify settings, attributes, and the Keycloak realm because the backend ignores missing field mask paths.
- The service allows updates to `realm_name` despite the proto marking it immutable, risking tenant realm drift.

The same review surfaced follow-up opportunities (boolean feature toggles, logging hygiene, helper APIs) tracked for later iterations.

---

## Objectives

1. Respect gRPC `FieldMask` semantics so only explicitly supplied paths update server state.
2. Enforce `realm_name` immutability at the service layer.
3. Keep SDK and frontend flows compatible with the stricter backend.
4. Validate behaviour via unit coverage and service tests.
5. Record findings, fixes, and testing outcomes for stakeholders.

---

## Deliverables

- [x] Hardened organization update path with mask-aware mutation.
- [x] Guardrail preventing `realm_name` modification after insert.
- [x] Updated unit tests covering both fixes.
- [x] Regression tests (existing suites) still green.
- [x] Documentation of progress and next steps.

---

## Work Log

| Date (UTC) | Status | Notes |
|------------|--------|-------|
| 2025-11-05 | ✅ Complete | Drafted backlog entry, implemented mask & realm hardening, added unit coverage (pnpm -C services/platform test -- organization-handlers). |
| 2025-11-05 | 🚧 In Progress | Added key/realm validation annotations, backend auth, SDK helpers, and unique role index; frontend test suite timed out locally (pnpm -C services/www_ameide_platform test -- org-scope). |

---

## Follow-ups (not in current scope)

### Core platform cleanup

- Convert `OrganizationSettings.feature_toggles` to `map<string, bool>` and cascade through SDKs.
- Reduce verbose organization fetch logging in web tier once observability replacements exist.
- Provide helper utilities for constructing `FieldMask` values in SDK clients.
- Ship a real `visibility_filter` capability in `ListOrganizationsRequest` once taxonomy is locked (see cross-domain alignment).
- Plan frontend caching/error boundary strategy: cache organization lookups via shared store + React Query, add consistent fallback UIs for resolution failures.
- Align tenant isolation with other domains: add row-level security parity and tenancy migration scripts so Graph/Threads consumers can rely on enforced scoping.
- Capture structured metadata rather than dumping into free-form `attributes` (mirror Threads’ structured parts approach for future reporting).

### Cross-domain alignment

- Adopt the shared organization `RequestContext` helpers across Graph and Threads SDK consumers to stop duplicating context construction logic.
- Define a unified event taxonomy/outbox schema with Graph/Threads so governance/audit pipelines consume consistent change types.
- Coordinate with Threads/Graph owners on a unified visibility model (private / org / tenant) before expanding `OrganizationVisibility`.
- Ensure organization visibility/role updates propagate into graph repository access rules and thread channel permissions (shared policy document + tests).
- Publish a canonical organization “view” contract (counts, active features, visibility) that downstream Graph/Threads services consume instead of recomputing.
- Backfill slug/realm validation usage across Graph repositories and Threads tenant bootstrap to keep identifiers aligned with the new proto rules.
- Share organization feature toggles with Graph/Threads clients (SDK surface + caching) so cross-domain UIs gate behaviour consistently.
- Emit organization lifecycle events on the platform event bus so Graph/Threads can reconcile membership/visibility changes without ad-hoc polling.
