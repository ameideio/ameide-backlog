# Interfaces & Schemas (contract-level)

## 1) gRPC Service (proto sketch)
```proto
service BpmnCommandService {
  rpc Propose(CommandProposal) returns (AppliedOrRejected);
  rpc GetSnapshot(GetSnapshotRequest) returns (Snapshot);
  rpc GetEvents(GetEventsRequest) returns (GetEventsResponse); // debug/ops
}
```

## 2) Messages (essential fields only)

### CommandProposal (client → server)
- `tenant_id`, `diagram_id`, `actor_id`
- `expected_version` (uint64)
- `commands[]` (atomic batch)
  - Semantic: `CreateElement`, `UpdateProperties`, `CreateConnection`, `UpdateLabel`, `DeleteElements`
  - Layout: `MoveElement`, `ResizeElement`, `UpdateWaypoints`
- *Notes*: client IDs/positions are suggestions; server may rewrite.

### AppliedOrRejected (server → client)
- oneof:
  - **AppliedTransaction**
    - `version` (new semantic version)
    - `applied[]` (ordered list of canonical events: semantic first, then layout for that version)
    - `id_map {client_id -> server_id}`
    - `warnings[]` (lint or non-fatal notes)
    - optional `xml_snapshot` (size-guarded)
  - **Rejection**
    - `reason_code` (e.g., `XSD_ERROR`, `LINT_FAIL`, `SEM_RULE_VIOLATION`, `VERSION_MISMATCH`, `AUTHZ_DENIED`)
    - `details[]`

### EventEnvelope (store)
- `stream_id` = `tenant-{tid}:diagram-{id}:{semantic|layout}`
- `stream_seq` (monotonic per stream)
- `global_seq` (store total order)
- `semantic_version` (only for semantic)
- `base_version` (only for layout)
- `ts_unix_ms`, `origin {type: human|agent, id}`
- `payload` (typed event)

## 3) Canonical Event Types (payload)

**Semantic**
- `CreatedElement {id, type, parent_id, props{}}`
- `UpdatedProperties {id, changes{}}`
- `CreatedConnection {id, type, source_id, target_id, props{}}`
- `UpdatedLabel {id, text}`
- `DeletedElements {ids[]}`

**Layout**
- `MovedElement {id, x, y, parent_id?}`
- `ResizedElement {id, width, height}`
- `UpdatedWaypoints {id, points[]}`

## 4) Read Models (minimal schemas)

### SQL (PostgreSQL)
- `bpmn_elements(diagram_id, element_id, element_type, name, parent_id, properties JSONB, version, created_at, updated_at)`
- `bpmn_flows(diagram_id, flow_id, source_id, target_id, condition_expression, condition_language)`
- `bpmn_statistics(diagram_id PK, task_count, gateway_count, event_count, flow_count, last_updated)`

### Graph (Neo4j/AGE)
- Nodes: `(:Process) (:Task) (:Gateway) (:Event) (:DataObject)`
- Rels: `(:A)-[:SEQUENCE_FLOW {id, condition}]->(:B)`, `(:Event)-[:BOUNDARY_OF {interrupting}]->(:Task)`, etc.

## 5) Validation Outputs
- **XSD**: `valid: bool`, `errors[] {line, col, message}`
- **Lint**: `errors[] {rule, message, element_id}`, `warnings[]`

## 6) Security / Tenancy (header fields)
- `x-tenant-id`, `x-actor-id`, `x-request-id`, `x-run-id` (loop prevention)