# Processing & Flows

## 1) UI Gate & Apply (single user)
- **Rules Gate** (bpmn-js `RuleProvider` / `commandStack.*.canExecute`):
  - Gate: `shape.create`, `elements.move`, `connection.create`, `connection.reconnect`, `shape.resize`, `elements.delete`.
  - Default deny unless a short-lived **permit** exists (set upon accept).
- **Proposal**: Build CommandProposal (batch), send via gRPC-Web.
- **Pending UX**: ghost/preview only (no model write).
- **Ack Handling**:
  - **Applied**: remap IDs via `id_map`, then **programmatically apply** operations through the modeling API (one undo step).
  - **Rejected**: show toast with `reason_code`; do nothing to canvas.

## 2) Validation Pipeline (server)
1) **AuthZ & Tenancy** → reject early if needed.  
2) **Version check** (`expected_version == current`) → else `VERSION_MISMATCH`.  
3) **Semantic checks** (domain rules).  
4) **Projection (moddle)** to temporary model; materialize both semantic and layout.  
5) **Export XML**; **XSD validate** (authoritative).  
6) **Lint** (advisory); attach warnings.  
7) **Commit** to Event Store (semantic v+1; layout base_v=v+1); write snapshot if thresholds met.  
8) **Return AppliedTransaction** (final IDs/positions, version, warnings [,xml?]).

## 3) Ordering & Merge Rules
- **Within a transaction**:
  - Apply **semantic** first ⇒ `version = v+1`.
  - Apply **all layout** with `base_version = v+1`.
- **Across transactions**:
  - Consumers replay semantic by version asc; interleave layout when `base_version` matches.

## 4) Sequences (mermaid)

### A) Accept
```mermaid
sequenceDiagram
autonumber
actor U
participant UI as bpmn-js (Rules Gate)
participant W as Envoy (gRPC-Web)
participant S as Command Service
participant E as Event Store
participant P as Moddle Projection
participant X as XSD/Lint

U->>UI: Drag/Create (intent)
UI->>UI: canExecute? → DENY (no local commit)
UI->>W: CommandProposal
W->>S: forward
S->>S: AuthZ + Version + Domain
S->>P: Apply to moddle (temp)
P-->>S: Temp model
S->>X: XSD + Lint
X-->>S: OK (+warnings)
S->>E: Append semantic (v+1) + layout (base_v=v+1)
E-->>S: Committed
S-->>W: AppliedTransaction {version, id_map, applied[]}
W-->>UI: Ack
UI->>UI: Apply programmatically; one undo step
```

### B) Reject
```mermaid
sequenceDiagram
autonumber
U->>UI: Edit intent
UI->>W: CommandProposal
W->>S: forward
S->>S: AuthZ/Version/Domain/XSD
S-->>W: Rejection {reason_code, details}
W-->>UI: Rejection
UI->>UI: Show error; no change
```

## 5) Snapshotting & Retrieval
- **Write snapshot** when: events ≥ N, or Δt ≥ T, or major change.
- **GetSnapshot** returns `{xml, version, created_at}`.
- Optionally include snapshot in the **accept ack** when a boundary is crossed.

## 6) Agents (HITL)
- Agents send **CommandProposals** with `origin.type=agent`.
- Same validation and commit path.
- UI receives an **AppliedTransaction** only when accepted by reviewer (HITL gate is policy-controlled in server).

## 7) Failure Modes & UI Behavior
- **Network**: keep pending; allow cancel; timeouts show retry UI.
- **VERSION_MISMATCH**: UI fetches snapshot; prompts user to retry action.
- **XSD/Lint fail**: show rule message; no canvas change.

## 8) Performance Notes
- UI: propose on **dragend** or throttle to 150–300 ms; accept **last ack** only.
- Server: batch layout ops; compress waypoints; defer heavy lint rules behind size guard.