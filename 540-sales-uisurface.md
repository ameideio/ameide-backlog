# 540 Sales — UISurface component

**Status:** Implemented (scaffolded server; CQRS wiring ready)
**Parent:** `backlog/540-sales-capability.md`

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (UISurface primitive), Application Services (query + command clients).

## Responsibilities

- Present a UI/API surface for humans or external clients.
- Read via Projection (`SalesQueryService`); write via Domain (`SalesCommandService`) and/or intent topics.
- Do not embed business rules; keep policy and invariants in Domain/Process.

## Implementation status

- Primitive: `primitives/uisurface/sales`
- Serves a generated `internal/gen/index.generated.html` if present; otherwise a stable fallback response.
- Next: connect UI actions to `SalesCommandService` and read paths to `SalesQueryService`.

## Implementation progress (checklist)

- [x] UISurface scaffold exists and passes `ameide primitive verify --kind uisurface --name sales --mode repo`.
- [x] Non-root container configuration is in place (security baseline).
- [ ] No real UI workflow is implemented yet (no CQRS wiring to projection/domain; current surface is static scaffolding).

## Next steps (checklist)

- [ ] Implement minimal read-only UI: list opportunities, list quotes, approval queue, detail views.
- [ ] Add write flows as “proposal-first” (submit for approval, approve/reject, accept/decline) with explicit user confirmation.
- [ ] Add auth integration (tenant selection, user identity, role-aware affordances).
- [ ] Add smoke tests that assert CQRS boundaries: UI reads only from Projection; writes only via Domain commands/intents.

## Clarification requests (checklist)

- [ ] What is the target UI scope for v0 (seller pipeline only vs quoting/approval queue vs forecasting)?
- [ ] What is the canonical auth mechanism for UISurfaces (OIDC via platform gateway, direct JWT verification, or service mesh auth)?
- [ ] Do we need “audit timeline” and “diff view” UX as a must-have for governance?
