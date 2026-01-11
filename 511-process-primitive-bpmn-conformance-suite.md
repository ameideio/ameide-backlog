# 511 — Process Primitive BPMN Conformance Suite (Idea)

## Problem

We are making BPMN a **canonical authoring format** for Processes, while executing on **Temporal** via a deterministic toolchain that produces workflow code and runs it.

The risk is permanent drift:

- the **diagram semantics** imply Flowable/BPMN behavior,
- but the **CLI-enforced runtime behavior** implements something else,
- and we only discover it when a real capability (e.g. Transformation) hits a corner case.

We need a way to make “diagram must not lie” testable and continuously enforced.

## Proposal

Create a dedicated **conformance Process primitive** whose only purpose is to exercise every **supported** BPMN construct (Ameide BPMN Execution Profile) end-to-end:

1. **Verify**: the CLI rejects anything outside the profile (`verify-bpmn` / `primitive verify`).
2. **Scaffold/refresh**: the CLI deterministically (re)writes generated workflow artifacts (e.g., `_gen.go`, `compile.lock.json`) as part of Process scaffolding.
3. **Execute**: the CLI runs the conformance primitive’s Temporal tests (Go `testsuite.WorkflowTestSuite`) and asserts observable behavior.

This becomes our **battle-tested** reference process for the CLI-enforced BPMN→Temporal contract.

## Why this belongs under 511

511 already defines:

- the Process scaffold contract,
- the BPMN execution profile,
- and the CLI responsibilities (scaffold/verify/test) that make the profile enforceable.

This conformance suite is the missing **verification artifact** that proves 511’s contract works under a real Temporal execution.

## Deliverables

1. New Process primitive (test-only):
   - `primitives/process/bpmn_conformance_v1/`
2. One or more BPMN files that cover the supported profile:
   - `primitives/process/bpmn_conformance_v1/bpmn/conformance.bpmn` (and optionally additional diagrams if readability requires)
3. Negative BPMN fixtures that must be rejected:
   - e.g. `primitives/process/bpmn_conformance_v1/bpmn/negative/*.bpmn`
4. Test suite:
   - CLI/toolchain verification tests (fast, pure Go)
   - Temporal execution tests (Go `testsuite.WorkflowTestSuite`) that execute the **generated workflow** and assert:
     - step ordering
     - wait-state semantics
     - message correlation behavior
     - explicit outcome semantics (success/failure/cancel) for supported outcomes

## Success criteria

- A single “coverage matrix” exists for the profile: each supported construct has at least one green scenario.
- Any unsupported construct has at least one red fixture: `verify-bpmn` fails with an actionable, stable error message.
- The conformance primitive is runnable in CI without Kubernetes:
  - no k3d, no AKS, no Telepresence
  - Temporal is exercised via the Go SDK testsuite (deterministic replay)
- A change that breaks the CLI-enforced runtime semantics must fail CI immediately.

## Scope (v1)

The conformance suite is versioned by the execution profile.

- **`bpmn_conformance_v1` covers “profile v1”** as defined by:
  - `backlog/511-process-primitive-scaffolding-spec.md`
  - `backlog/511-process-primitive-scaffolding-bpmn-extension.md`
- Future constructs (timers, boundary events, event sub-process, parallel gateways, etc.) are covered by **new conformance versions** once they become supported (e.g. `bpmn_conformance_v2`).

## Non-goals

- Full Flowable feature parity.
- UI E2E testing (Playwright).
- Cluster E2E (AKS/k3d) as the default conformance gate.
- Performance benchmarking.

## Cross-references

- 511 profile/spec:
  - `backlog/511-process-primitive-scaffolding.md`
  - `backlog/511-process-primitive-scaffolding-spec.md`
  - `backlog/511-process-primitive-scaffolding-refactoring.md`
- Testing discipline:
  - `backlog/537-primitive-testing-discipline.md`
  - `backlog/430-unified-test-infrastructure-v2-target.md`
- Process direction/examples:
  - `backlog/527-transformation-process.functional.normative.md`
