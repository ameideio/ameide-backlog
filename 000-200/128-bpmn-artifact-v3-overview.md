# V3 — Server-Gated BPMN Editor (Single-User, No Collaboration)

## 1. Scope & Goals
- Build a **server-authoritative** BPMN 2.0 editor.
- **All edits are gated** by backend; UI applies only **accepted** changes.
- Maintain **XSD compliance** (OMG BPMN 2.0) and **bpmnlint** best-practice checks.
- Persist **event-sourced semantics** with **separate layout** stream.
- Expose clean **gRPC-Web → Envoy → gRPC** boundary.
- Provide **read models** (PostgreSQL + Graph) and **Agent HITL** suggestions.

## 2. Non-Goals
- No real-time multi-user collaboration (no CRDT/Yjs).
- No server DOM rendering; **BpmnModeler is never run on server**.

## 3. Principles
- **Backend is source of truth.**
- **Pessimistic gating**: UI never mutates model without server permit.
- **One user action = one atomic transaction** (version++ once).
- **Semantics and layout are distinct concerns** (dual stream).
- **Spec before style**: XSD is authoritative; lint is advisory.

## 4. High-Level Architecture
```
Browser (bpmn-js)
├─ Rules Gate (canExecute) → blocks local command
├─ CommandProposal (gRPC-Web)
▼
Envoy (gRPC-Web ↔ gRPC)
▼
Command Service (gRPC)
├─ AuthZ / Tenancy
├─ Validate (XSD + bpmnlint + domain)
├─ Event Store (semantic v+1; layout base_v)
├─ Moddle Projection → XML
├─ Snapshots
└─ Ack: AppliedTransaction {version, id_map, applied[], [xml?]}
    ▲
    └─ Agents (HITL proposals) share the same API
```

## 5. Data Views
- **Event Store**: append-only semantic/layout events.
- **Read Models**
  - SQL (elements, flows, stats) for simple queries.
  - Graph (Neo4j/AGE) for paths/compliance/context.

## 6. UX Contract (summary)
- While pending: **ghost/preview** only, no canvas commit.
- On **accept**: UI **applies** server-returned operations/IDs/positions.
- On **reject**: UI shows reason; **no change** to canvas.