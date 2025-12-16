# 540 Sales — Agent component (Sales Copilot)

**Status:** Implemented (runtime stub; projection-backed skills next)
**Parent:** `backlog/540-sales-capability.md`

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Agent primitive), Application Services (query clients), Application Interfaces (typed intents).

## Responsibilities (520-aligned)

- Read via projections and synthesize user-facing assistance (summaries, recommendations).
- Propose typed intents/commands; do not write domain state directly.
- Keep prompts/policy out of proto; keep agent state small and auditable.

## Implementation status

- Primitive: `primitives/agent/sales-copilot`
- Current behavior is a stub LangGraph agent with tests; next step is projection-backed skills + typed intent proposal wiring.

## Implementation progress (checklist)

- [x] Agent primitive scaffold exists at `primitives/agent/sales-copilot` (LangGraph + packaging + Dockerfiles).
- [x] Basic unit tests exist under `primitives/agent/sales-copilot/tests/`.
- [x] `ameide primitive verify --kind agent --name sales-copilot --mode repo` passes.
- [ ] No projection-backed skills implemented yet (no SalesQueryService client, no pipeline/approval queue reads).
- [ ] No typed intent proposal surface implemented yet (no `SalesDomainIntent` proposal generation with confirmation flow).

## Next steps (checklist)

- [ ] Add a typed SalesQueryService client and implement read skills (pipeline board, quote status, approval queue, audit timeline).
- [ ] Add intent proposal generation (output `SalesDomainIntent` payloads) and ensure all writes remain mediated (UI/process/human confirm).
- [ ] Add safety rails: tenant pinning, “read-before-write”, and refusal for direct state edits without a typed intent.
- [ ] Add evaluation harness: golden prompts → deterministic intent proposals; regression tests for tool/skill routing.

## Clarification requests (checklist)

- [ ] What is the v0 skill set: read-only copilot, or also “propose approvals/acceptance” flows (and who confirms them)?
- [ ] What is the canonical transport for agent reads: gRPC direct to `SalesQueryService`, or via platform Graph/Repository gateway?
- [ ] What is the required safety posture: is any direct Domain command issuance allowed from the agent, or proposal-only always?
- [ ] Should the agent maintain a durable thread state (`thread_id`) and if so, where is it stored (Threads service vs local only)?
