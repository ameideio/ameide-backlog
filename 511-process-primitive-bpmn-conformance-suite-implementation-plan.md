# 511 — BPMN Conformance Suite: Implementation & Test Plan

## Goal

Implement an end-to-end conformance gate proving that:

`BPMN (profile) → verify-bpmn → compiler → generated workflow code → Temporal execution`

is correct for every supported BPMN construct in the current profile.

## Implementation phases

### Phase 0 — Define the contract surface (docs-only)

- Add the conformance suite backlogs:
  - idea (why)
  - BPMN specification (what to model)
  - implementation plan (this document)
- Add/maintain an explicit coverage matrix in `bpmn_conformance_v1` README (supported vs rejected).

### Phase 1 — Add the conformance primitive

- Create `primitives/process/bpmn_conformance_v1/` using the standard Process scaffold.
- Add BPMN diagrams:
  - `bpmn/conformance.bpmn`
  - `bpmn/negative/*.bpmn`
- Ensure every BPMN node includes `bpmn:documentation` describing:
  - compile mapping
  - runtime observable

### Phase 2 — Compiler / verifier tests (fast)

Add tests at the compiler layer that:

- run `verify-bpmn` on:
  - `conformance.bpmn` (must pass)
  - `negative/*.bpmn` (must fail with stable messages)
- compile `conformance.bpmn` and assert IR invariants:
  - step count and step ids are stable
  - wait-step subscription rules are enforced
  - unsupported constructs are rejected (no silent ignores)
- assert `compile.lock.json` digest is deterministic (no timestamps, no nondeterministic ordering)

### Phase 3 — Temporal execution tests (end-to-end without Kubernetes)

In `primitives/process/bpmn_conformance_v1/tests/` (or equivalent), add Go tests using:

- `go.temporal.io/sdk/testsuite.WorkflowTestSuite`

These tests should register:

- the **generated workflow** from the conformance primitive (`*_gen.go`)
- mock Activities that record:
  - which activity ran
  - the input payload
  - and return deterministic outputs

Then assert:

- service task ordering
- XOR determinism
- message wait blocking/unblocking behavior
- user task wait completion behavior
- idempotency rules (repeated completion signals/updates)

This is the minimum “battle-tested” runtime gate without requiring a live Temporal server or Kubernetes.

### Phase 4 — CLI integration (developer ergonomics)

Make the conformance suite runnable as a single command in CI and locally.

Preferred shape (no new modes):

- `ameide primitive verify --kind process --name bpmn_conformance_v1`

This should:

- run `verify-bpmn` and fail fast on any unsupported construct
- validate generated artifacts match the BPMN (no drift)
- run the conformance primitive’s Go test suite (unit + integration-tagged where applicable)

## CI / “inner loop” placement

- Conformance suite must run in `ameide ci test` (fast, no cluster).
- It must not require Telepresence, k3d, or AKS.
- Cluster E2E can be optional/future, but is not the primary conformance signal.

## Acceptance criteria

- Adding a new supported BPMN construct requires adding:
  - a green conformance scenario
  - at least one negative fixture for common misuse
- Removing support (or changing semantics) must be reflected by:
  - updated conformance BPMN documentation
  - updated tests that prove the new semantics
- A compiler change that “silently ignores” a BPMN node must be caught by conformance tests.

## Cross-references

- 511 execution profile/spec: `backlog/511-process-primitive-scaffolding-spec.md`
- 511 compiler refactoring: `backlog/511-process-primitive-scaffolding-refactoring.md`
- Testing discipline: `backlog/537-primitive-testing-discipline.md`
- Test contract: `backlog/430-unified-test-infrastructure-v2-target.md`

