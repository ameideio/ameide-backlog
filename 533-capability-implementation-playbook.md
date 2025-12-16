# 533 — Capability Implementation Playbook (Agent Prompt as a DAG)

This is a **plan of activities** intended to map cleanly onto future orchestration (e.g. LangGraph nodes).

Each node lists:
- **Context (read only)**: the minimum backlog/docs to read
- **Actions**: what to run/do/investigate
- **Outputs**: concrete artifacts the node must produce
- **Exit criteria**: how to decide the node is “done”

This playbook is intentionally **capability-agnostic**. Capability docs define semantics; `520` defines platform guardrails.

## Implementation progress (repo)

- [x] Proven end-to-end on at least one capability slice (SRE): `buf lint` + `buf breaking` + codegen freshness + GitOps component checks exercised via `ameide primitive verify`.
- [x] CI/guardrails feedback loop validated: verification failures drove structural fixes (monorepo `buf breaking` `--against` ref, codegen mapping for `google/api/field_behavior.proto`, process/domain shape checks).
- [ ] Convert this playbook into a machine-executable workflow spec (LangGraph/Temporal) with stable node IDs and structured outputs.
- [ ] Add a “per-node verification command” appendix (exact `ameide primitive verify` invocations per node, plus expected failure classes).

## Clarifications requested (next steps)

- [ ] Define whether `Node 8` is allowed to fail due to unrelated repo-wide proto lint failures, or whether verification should be scoped to “touched packages only”.
- [ ] Decide canonical “definition of done” for **Projection** and **Integration** beyond scaffold shape (e.g., required inbox/outbox patterns, minimum query/read models).
- [ ] Decide whether “event catalog” is mandatory for every Domain/Process (currently verify treats absence inconsistently across primitives).
- [ ] Define the minimum acceptable “MCP adapter” compliance level at `Node 7C` (proto-derived tool schemas vs hand-authored stopgaps).

## DAG Overview (parallelizable)

```
                         +---------------------------+
                         | Inputs (given)            |
                         | - Capability spec         |
                         | - Requirement             |
                         +-------------+-------------+
                                       |
                 +---------------------+---------------------+
                 |                                           |
      +----------v----------+                     +----------v----------+
      | Node 0              |                     | Node 1              |
      | Load Guardrails     |                     | Validate Spec       |
      +----------+----------+                     +----------+----------+
                 |                                           |
                 +---------------------+---------------------+
                                       |
                          +------------v------------+
                          | Gate A                  |
                          | Guardrails + Spec ready |
                          +------------+------------+
                                       |
                          +------------v------------+
                          | Node 2                  |
                          | Design Contracts (Proto)|
                          +------------+------------+
                                       |
                          +------------v------------+
                          | Gate B                  |
                          | Protos compile          |
                          +------------+------------+
                                       |
      +------------------+-------------+------------------+------------------+
      |                  |             |                  |                  |
 +----v----+         +---v----+    +---v----+         +---v----+         +---v----+
 | Node 3A |         | Node 3B|    | Node 3C|         | Node 3D|         | Node 3E|
 | Scaffold|         | Scaffold|    | Scaffold|        | Scaffold|        | Scaffold|
 | Domain  |         | Projection    | Process |       | UISurface       | Agent   |
 +----+----+         +---+----+    +---+----+         +---+----+         +---+----+
      |                  |             |                  |                  |
      v                  v             v                  v                  v
 +----+----+         +---+----+    +---+----+         +---+----+         +---+----+
 | Node 4  |         | Node 5 |    | Node 6 |         | Node 7A|         | Node 7B|
 | Implement          | Implement    | Implement       | Implement       | Implement
 | Domain  |         | Projection    | Process |       | UISurface       | Agent   |
 +----+----+         +---+----+    +---+----+         +---+----+         +---+----+
      |                  |             |                  |                  |
      +------------------+-------------+------------------+------------------+
                                       |
                                +------v------+
                                | Node 8      |
                                | Verify &    |
                                | Package     |
                                +-------------+

Note: Node 3F (Scaffold Integration) and Node 7C (Implement Integration) follow the same pattern as 3D/7A and 3E/7B.
```

**Gates**
- **Gate A:** guardrails + spec readiness
- **Gate B:** contracts compile (protos lint/build)

**Fan-out parallelism**
- After **Gate B**, scaffold primitives in parallel (`Node 3A..3F`).
- After scaffolding, implement in parallel where safe; converge at `Node 8`.

## How This Relates to 505 and 506

- `backlog/505-agent-developer-v2.md`: provides the detailed Agent discipline that plugs into this DAG as `Node 7B` (proposal-only posture, safety/risk boundaries, how agents consume projections/queries and invoke domain/process intents).
- `backlog/506-scrum-vertical-v2.md`: is a concrete instantiation of this generic DAG for a Scrum vertical slice (contracts + seams + verification).

## Cross-Reference Index

- **CLI surface area (commands, workflows):** `backlog/484-ameide-cli-overview.md`, `backlog/484a-ameide-cli-primitive-workflows.md`
- **Platform guardrails (what must be true):** `backlog/520-primitives-stack-v2.md`, `backlog/514-primitive-sdk-isolation.md`, `backlog/509-proto-naming-conventions.md`
- **Agent memory/DAG alignment:** `backlog/520-primitives-stack-v2-research-agent.md`, `backlog/512-agent-primitive-scaffolding.md`
- **Primitive scaffolding specs (shape per kind):** `backlog/510-domain-primitive-scaffolding.md`, `backlog/511-process-primitive-scaffolding.md`, `backlog/512-agent-primitive-scaffolding.md`, `backlog/513-uisurface-primitive-scaffolding.md`
- **Verification discipline:** `backlog/415-implementation-verification.md`, `backlog/448-per-environment-service-verification.md`
- **Testing discipline:** `backlog/537-primitive-testing-discipline.md` (RED→GREEN TDD, per-primitive invariants, CI enforcement)
- **Operators (control-plane responsibilities):** `backlog/498-domain-operator.md`, `backlog/499-process-operator.md`, `backlog/500-agent-operator.md`, `backlog/501-uisurface-operator.md`
- **Scaffolding/codegen change logs:** `backlog/521c-internal-generation-improvements.md`, `backlog/521d-external-generation-improvements.md`

---

## Node 0 — Load Guardrails (Platform Constitution)

**Context (read only)**
- `backlog/520-primitives-stack-v2.md`
- `backlog/514-primitive-sdk-isolation.md`
- `backlog/521d-external-generation-improvements.md`

**Actions**
- Identify the primitive set required (Domain is mandatory; others only if needed).
- Record the non-negotiables (canonical writers, event-driven seams, SDK-only imports, per-primitive DB+migrations).

**Outputs**
- A short “guardrails checklist” for this capability PR.

**Exit criteria**
- Guardrails checklist exists and is referenced by implementation work.

## Node 1 — Validate Spec (ArchiMate chain + acceptance seams)

**Context (read only)**
- `backlog/528-capability-definition.md`
- `backlog/530-ameide-capability-design-worksheet.md`
- Capability-specific docs (the capability spec itself)

**Actions**
- Confirm the ArchiMate chain exists: Capability → Value Streams → Business Processes → Application Services/Events → Application Components → Technology Services.
- Confirm identity/scope axes exist and at least one acceptance slice is defined.

**Outputs**
- “Spec readiness” notes (what’s missing; what’s locked).

**Exit criteria**
- Spec has enough shape to define contracts (`Node 2`).

## Node 2 — Design Contracts (Proto)

**Context (read only)**
- `backlog/509-proto-naming-conventions.md`
- `backlog/496-eda-protobuf-ameide.md`, `backlog/496-eda-principles.md`

**Actions**
- Define topic families and aggregator envelopes (domain intents/facts/process facts).
- Define query services needed by UISurfaces/Agents (read-only services).
- Keep semantics in the capability docs; keep envelopes/idempotency/metadata consistent with platform guardrails.

**Outputs**
- Proto proposal (packages/topics/envelopes/messages/services).

**Exit criteria**
- `buf lint`/`buf build` pass (Gate B).

## Node 7B — Implement Agent (LangGraph-aligned)

**Context (read only)**
- `backlog/505-agent-developer-v2.md`
- `backlog/520-primitives-stack-v2-research-agent.md`
- `backlog/512-agent-primitive-scaffolding.md`

**Actions**
- Enforce `thread_id` when persistence is enabled; map to `configurable.thread_id`.
- Keep agent state small; store large outputs as out-of-band artifact refs.
- Use reducers on all state fields; use `Send` for explicit fan-out/fan-in when parallelism is required.
- Treat side-effects (A2A task creation, domain intents, integrations) as idempotent under replay/interrupt.
- Standardize streaming outputs (prefer `updates` for machine-readable progress).

**Outputs**
- An Agent that can be restarted without losing truth (domain/process remain canonical).

**Exit criteria**
- Agent conformance checks pass (thread_id discipline, replay safety, artifact refs, proposal-only posture).

## Node 8 — Verify & Package

**Context (read only)**
- `backlog/537-primitive-testing-discipline.md` (testing requirements, RED→GREEN enforcement, per-primitive invariants)
- `backlog/520-primitives-stack-v2.md` (§7 CI gate checklist, §Conformance Checklists)
- `backlog/415-implementation-verification.md`, `backlog/448-per-environment-service-verification.md` (verification context)

**Actions**
- Run `ameide primitive verify` for each implemented primitive:
  - Domain: `--checks naming,eda,imports,tests`
  - Process: `--checks naming,shape,tests`
  - Projection: `--checks naming,imports,tests`
  - Integration: `--checks naming,imports,tests`
  - UISurface: `--checks naming,tests`
  - Agent: `--checks naming,imports,tests`
- Confirm no RED scaffold tests remain (any `AMEIDE_SCAFFOLD` markers cause `ameide primitive verify` to fail).
- Run `buf lint` and `buf breaking` on proto changes.
- Run regen-diff to ensure generated outputs are committed.
- Validate GitOps manifests (if `--include-gitops` was used).

**Outputs**
- All primitives pass verification checks.
- All tests pass (GREEN state).
- Proto contracts are lint-clean and backward-compatible.
- GitOps manifests are valid and deployable.

**Exit criteria**
- `ameide primitive verify` passes for all primitives in the capability.
- CI gates pass (tests, lint, breaking, regen-diff).
- Capability is ready for deployment to target environment.
