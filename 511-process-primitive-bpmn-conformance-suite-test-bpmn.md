# 511 — BPMN Conformance Suite: Test BPMN Specification

## Objective

Define the BPMN diagram(s) used by the Process conformance primitive so we can:

- validate the Ameide BPMN execution profile via `verify-bpmn`
- compile deterministically into Temporal workflow code
- execute the generated workflow under Temporal testsuite and assert behavior

The BPMN diagram is not “documentation only”: it is an executable contract and must be kept honest via tests.

## Location / naming

- Primitive: `primitives/process/bpmn_conformance_v1/`
- Primary diagram: `primitives/process/bpmn_conformance_v1/bpmn/conformance.bpmn`
- Optional additional diagrams (only if readability requires):
  - `primitives/process/bpmn_conformance_v1/bpmn/conformance_<topic>.bpmn`
- Negative fixtures (must be rejected):
  - `primitives/process/bpmn_conformance_v1/bpmn/negative/<case>.bpmn`

## Conformance coverage matrix (v1)

This suite must include at least one scenario for every construct supported by the current profile, and at least one negative fixture for every major unsupported family.

**Supported (profile v1) — must have green scenarios**

- `bpmn:process` with required `ameide:workflowDefinition`
- `bpmn:startEvent` (none start only)
- `bpmn:endEvent` (none end only)
- `bpmn:sequenceFlow`
- `bpmn:serviceTask` with `ameide:taskDefinition`
- `bpmn:userTask` (modeled wait state; completion via external update/signal)
- `bpmn:receiveTask` and/or `bpmn:intermediateCatchEvent` (message wait)
- `bpmn:message` with `ameide:subscription` (or node-local subscription where supported)
- `bpmn:exclusiveGateway` with deterministic conditions
- `bpmn:subProcess` (only if supported by compiler semantics)

**Unsupported (profile v1) — must have red fixtures**

- timer waits / timer start / timer boundary
- boundary events (message/timer/error)
- event subprocess
- parallel gateways and concurrency
- compensation / transaction / call activity
- end event definitions (terminate/error/message/etc)
- any BPMN element that the compiler does not explicitly support (the diagram must not be silently ignored)

## Required `bpmn:documentation`

Every node in the conformance diagram must include a `bpmn:documentation` element describing:

1. **Compile mapping** (what the compiler is expected to generate)
2. **Runtime observable** (what the tests will assert)
3. **Failure mode** (what must happen if inputs are invalid or the wrong signal arrives)

Example (service task):

- Compile mapping: “serviceTask → Temporal Activity `<TaskType>`”
- Runtime observable: “Activity invoked once; output merged into workflow-local state via io mapping”
- Failure mode: “Activity retry semantics follow policyRef; failures surface as workflow failure (until typed outcomes exist)”

## Canonical scenario set (recommended single-diagram layout)

This is a suggested layout for `conformance.bpmn` that stays readable and makes it easy to write tests.

### Scenario A — Minimal happy path

Start → serviceTask (activity) → end

Asserts:

- activity called exactly once
- workflow completes successfully

### Scenario B — Deterministic XOR conditions

Start → serviceTask “compute” → XOR gateway → (path1 | path2) → end

Asserts:

- given fixed inputs, the chosen path is deterministic
- no map-iteration ordering leaks into control flow

### Scenario C — Message wait correlation

Start → serviceTask “start job” → message wait → serviceTask “finalize” → end

Asserts:

- workflow blocks until message arrives
- duplicate message IDs are ignored (idempotency)
- wrong correlation key does not advance this workflow instance

### Scenario D — User task wait + completion

Start → userTask “approve” → serviceTask “post-approval work” → end

Asserts:

- workflow blocks at userTask
- completion via update/signal advances exactly once
- invalid completion (wrong step instance id) is rejected

### Scenario E — Subprocess scoping (only if supported)

Start → subprocess:

- serviceTask
- message wait

→ end

Asserts:

- step IDs remain stable and uniquely addressable
- entering/leaving subprocess does not break determinism

## Negative fixture design

Each negative BPMN file should contain exactly one “reason to fail”, so errors remain crisp and stable.

Examples:

- `negative/timer_catch_event.bpmn`: contains `intermediateCatchEvent` with `timerEventDefinition` → must fail.
- `negative/boundary_timer.bpmn`: contains timer boundary event → must fail.
- `negative/end_error_event.bpmn`: contains `endEvent` with `errorEventDefinition` → must fail.
- `negative/parallel_gateway.bpmn`: contains parallel gateway → must fail.

## Cross-references

- Execution profile/spec: `backlog/511-process-primitive-scaffolding-spec.md`
- Extension schema: `backlog/511-process-primitive-scaffolding-bpmn-extension.md`

