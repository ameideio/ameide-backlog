# Architecture Decision Records (V3 Optimistic)

**Status**: Decisions Implemented (2025-08-21)
**Implementation**: Complete SDK + Platform Integration with Protobuf/gRPC/Connect stack

## ADR-001 — Optimistic UI (No Backend Gating)
**Decision**: bpmn-js executes commands locally; no server round-trip before paint.
**Rationale**: Lowest latency, simplest UX; aligns with single-user and AI-in-editor.
**Trade-offs**: No server authority until Save; validation errors discovered late.

## ADR-002 — Periodic Snapshots (Drafts)
**Decision**: Auto-snapshot current XML at configurable intervals: { intervalMs: 30000, idleMs: 5000, maxDraftAge: 7d }.
**Rationale**: Crash recovery and draft continuity without version churn.
**Implementation**: IndexedDB always, remote draft API optional.

## ADR-003 — Version Bump on Explicit Save
**Decision**: Only a user-initiated Save creates a new immutable version (v = v+1).
**Rationale**: Clean audit points; users control commits.
**Consequence**: Autosaves are drafts only, not versions.

## ADR-004 — Validation: Lint Live, XSD on Save
**Decision**: Run bpmnlint continuously in the editor; run authoritative XSD validation server-side on Save.
**Rationale**: Fast feedback + spec compliance at persistence boundary.
**Trade-off**: Invalid diagrams can exist locally until Save attempt.

## ADR-005 — In-Editor AI Collaboration
**Decision**: AI plugin runs inside the editor process; generates **structured operations**; applies via command stack (HITL default, Autopilot optional).
**Rationale**: Keep agent and human in the same undo/redo domain; progressive streaming for UX.
**Implementation**: ✅ Server-streaming GetAISuggestions RPC with typed Operation messages.

## ADR-006 — Optional Event Streams (Local Only)
**Decision**: Keep **typed command capture** client-side (for analytics/telemetry). Server receives only snapshots at Save; event sourcing is optional server-side.
**Rationale**: Reduce backend complexity; leave room for later ES adoption.
**Future**: Can add event streaming without breaking changes.

## ADR-007 — Persistence Model
**Decision**: Backend stores: versions (immutable XML), latest draft (optional), metadata. Read models (SQL/Graph) update on Save.
**Rationale**: Save-time consistency; avoid drift between projections and editor state.
**Tables**: diagram_versions, diagram_drafts, read projections.

## ADR-008 — Transport
**Decision**: Editor uses Connect-ES v2 (gRPC-Web/Connect protocol) for all operations; AI calls stream through backend.
**Rationale**: Type safety, binary efficiency, native streaming support.
**Services**: BPMNEditorService with SaveDraft, WatchDraft, GetAISuggestions RPCs.
**Implementation**: ✅ Proto defined in `bpmn_editor_service.proto`

## ADR-009 — IDs & Layout
**Decision**: Client owns IDs and layout; server does not rewrite on Save unless policy enforces normalization.
**Rationale**: Deterministic user experience; normalization can be an optional tool.
**Exception**: Server may enforce unique IDs if duplicates detected.

## ADR-010 — MVP Scope
**Decision**: MVP includes local drafts, server saves with XSD, basic AI, no collaboration.
**Rationale**: Deliver value in 4 weeks; establish foundation for enhancement.
**Defer**: Multi-tenancy, auth, remote drafts, graph projections, advanced AI.

## ADR-011 — AI Safety
**Decision**: HITL (Human-in-the-Loop) by default; Autopilot requires explicit allowlist.
**Rationale**: Prevent runaway AI edits; maintain user trust.
**Allowlist Examples**: add-labels, fix-disconnected, add-default-flows.

## ADR-012 — Conflict Resolution
**Decision**: Optimistic concurrency with expected_version/etag; conflicts return conflicting operations.
**Rationale**: Client can rebase operations intelligently.
**Implementation**: ✅ SaveDraftResponse includes ConflictInfo with operation history.

## ADR-013 — Operation Deltas (NEW)
**Decision**: Send granular operations instead of full XML for drafts.
**Rationale**: Smaller payloads (10-100x), better conflict resolution, natural for collaborative features.
**Implementation**: ✅ Operation message with typed oneofs (CreateElementOp, UpdateElementOp, etc.)

## ADR-014 — Streaming Architecture (NEW)
**Decision**: Use server-streaming for AI suggestions and draft watching.
**Rationale**: Progressive UI updates, cancellation support, real-time collaboration.
**Implementation**: ✅ WatchDraft and GetAISuggestions return streams.