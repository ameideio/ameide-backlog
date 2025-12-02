# ADRs — V3

## ADR-001: Server-Gated Commands (Pessimistic)
**Decision**: Client sends CommandProposal; server validates/commits; UI applies only AcceptedTransaction.  
**Rationale**: Eliminate local/remote divergence and rebase complexity.

## ADR-002: No Collaboration / No CRDT
**Decision**: Single-user workflows; no Yjs/OT.  
**Rationale**: Reduce moving parts; authority stays server-side.

## ADR-003: Dual Streams (Semantic vs Layout)
**Decision**: Semantic events advance `version`; layout events reference `base_version = version_at_commit`.  
**Rationale**: Clean separation of business model vs DI/waypoints.

## ADR-004: Version per Transaction
**Decision**: One atomic batch per user action increments version once.  
**Rationale**: Predictable undo unit and concurrency control.

## ADR-005: Spec Compliance via XSD
**Decision**: Python `lxml` validates BPMN XSD; `bpmnlint` is advisory.  
**Rationale**: XSD defines correctness; lint improves ergonomics.

## ADR-006: Moddle-Only Server Projection
**Decision**: Use `bpmn-moddle` sidecar for deterministic model manipulation/export.  
**Rationale**: No DOM; stays compatible with BPMN tooling.

## ADR-007: ID Authority & Rewrites
**Decision**: Server owns final IDs; ack returns `id_map` for client remap.  
**Rationale**: Prevent collisions, ensure consistency.

## ADR-008: Transport
**Decision**: Browser uses **gRPC-Web**; Envoy bridges to internal gRPC.  
**Rationale**: Strong contracts, streaming support, standardized edge.

## ADR-009: Read Models
**Decision**: SQL for denormalized lookups; Graph for traversal/compliance; both updated on accept.  
**Rationale**: Fit-for-purpose query latency.

## ADR-010: Agents HITL First
**Decision**: Agents submit proposals; humans approve. Same API path; server validates.  
**Rationale**: Safety, auditability, gradual autonomy later.