# 511 – Process Primitive Scaffolding Refactoring (BPMN → Temporal runner, v1)

This document closes the remaining “open questions” for the v1 BPMN extension profile and defines a single, consistent direction for refactoring the scaffolder into a **Go-only, self-contained compiler pipeline**.

Scope: **compile BPMN + `ameide:*` extensions into a generic Temporal runner skeleton** that a coding agent can complete (activity bodies, ingress bindings, domain semantics).

## Crossreferences

- `backlog/511-process-primitive-scaffolding.md` (canonical Process primitive shape and runtime posture).
- `backlog/511-process-primitive-scaffolding-bpmn-extension.md` (v1 execution profile: required `ameide:*` elements + semantics).
- `backlog/472-ameide-information-application.md` (explicit “BPMN → Temporal compilation”, pinned `process_definition_id`, no runtime BPMN interpretation).
- `backlog/496-eda-principles.md` (at-least-once, idempotency, envelope/trace propagation norms).
- `backlog/499-process-operator.md` and `backlog/495-ameide-operators.md` (operator/primitive boundary; infra vs business logic).
- `backlog/514-primitive-sdk-isolation.md` (Process primitives call other primitives via SDKs only).
- `backlog/554-comparative-process.md` (worker+ingress pattern, SignalWithStart posture, runner packaging comparisons).
- `backlog/602-image-pull-policy.md` (images referenced by digest; no local registries).

## Goals

- One source of truth for what the CLI accepts: **BPMN + `ameide:*` extensions (v1)**.
- A **Go-only** linter/compiler in the `ameide` CLI (no Python/XSD dependency at runtime).
- Generated output is “as far as possible” from the diagram while remaining generic.
- A scaffolded/compiled Process primitive is buildable as a **Temporal runner image** (worker + ingress).

## Non-goals (v1)

- Full BPMN engine semantics or runtime interpretation.
- Support for boundary events, event subprocess, timers, compensation, multi-instance, event-based gateways.
- “Expression language” in BPMN (beyond the template grammar already defined in the v1 profile).

## Closed v1 decisions (no longer open)

### 1) Normalized ingress envelope is the ABI

All external deliveries reaching workflows MUST be normalized by ingress into a stable envelope (the platform ABI).

Required fields (names are normative):

- `tenant_id` (string)
- `subject_id` (string): the business key the workflow is scoped to (entity-workflow posture)
- `message_name` (string)
- `message_id` (string): dedupe key
- `correlation_key` (string): wait-matching key

Allowed optional metadata (if present, MUST NOT affect routing/correlation semantics):

- `correlation_id`, `causation_id` (traceability)
- `traceparent`, `tracestate` (W3C trace context)
- `payload` (opaque JSON / bytes) and/or `payload_type` (string)

Clarification: `correlation_key` (wait routing) is distinct from `correlation_id` (traceability). They MUST NOT be conflated.

This follows the v2 “single authority” rule: ingress defines routing keys in the envelope; request inputs live in payload; environment config is GitOps/operator-owned. See `backlog/520-primitives-stack-v2.md` (“Configuration authority contract”).

### 2) Correlation extraction rule is fixed in v1

- Inbound wait matching uses **only** `envelope.correlation_key`.
- `ameide:subscription` does **not** define a `correlationKeyPath` in v1.
- `ameide:subscription@correlationKeyTemplate` is evaluated when entering the wait and compared to the inbound `correlation_key` (string equality).

### 3) Supported wait node types (v1 execution profile)

To avoid scope/timing foot-guns, v1 supports only:

- `bpmn:receiveTask`
- `bpmn:intermediateCatchEvent` with `bpmn:messageEventDefinition`

Other wait patterns (boundary message events, event subprocess message start, etc.) are explicitly **out of scope** for v1.

### 4) Subscription placement is not optional in v1

To prevent duplicated/ambiguous await contracts:

- For message waits (`bpmn:intermediateCatchEvent` + `messageRef`): `ameide:subscription` MUST be attached to the referenced `bpmn:message` (not on the catch event).
- For `bpmn:receiveTask`: `ameide:subscription` MUST be attached to the referenced `bpmn:message` (via `receiveTask@messageRef`).

### 5) `ameide:workflowDefinition` is required for compilation

For any BPMN intended to be compiled by the CLI, the `bpmn:process` MUST include exactly one `ameide:workflowDefinition`.

Minimum requirements:

- `definitionId` is required and MUST change when semantics change.
- `workflowIdTemplate` is required and MUST resolve to a deterministic workflow id (entity-workflow posture).

Recommended default:

- `workflowIdTemplate="<process-definition-prefix>/${state.tenant_id}/${state.subject_id}"` (the prefix is a product decision; v1 templates allow only `${state.<path>}` placeholders).

### 6) Supported control-flow constructs (v1 compiler)

The compiler operates on a constrained, deterministic subset:

- Allowed: `bpmn:process`, embedded `bpmn:subProcess`, `bpmn:startEvent`, `bpmn:endEvent`, `bpmn:sequenceFlow`, `bpmn:serviceTask`, `bpmn:sendTask`, `bpmn:userTask`, supported waits (above), `bpmn:intermediateThrowEvent` (message), `bpmn:message`, and `bpmn:exclusiveGateway` (with a deterministic condition subset on outgoing `sequenceFlow/conditionExpression`).
- Disallowed (compile error): `inclusiveGateway`, `parallelGateway`, `eventBasedGateway`, `complexGateway`, boundary events, event subprocess, timers, compensation, multi-instance, call activity.

Embedded subprocesses are supported as standard BPMN structure; v1 does not spawn separate workflow instances (no call activities). Subprocess entering/leaving is compiled via BPMN-faithful “enter container start → run internals → leave on container end” semantics.

### 7) IO mapping constraints (v1)

To keep IO mapping compilable and expression-free:

- `ameide:ioMapping` entries are path-to-path only (Path grammar).
- `input`: `source` MUST start with `state.` and `target` MUST start with `request.`
- `output`: `source` MUST start with `result.` and `target` MUST start with `state.`

### 8) Policies are resolved deterministically at compile time

- `ameide:taskDefinition@policyRef` MUST resolve against a pinned catalog at compile time.
- The compiler materializes resolved policy values into generated code (no runtime “latest policy” reads).

### 9) Generated-vs-handwritten boundary

- Compiler output is written to generated-only files (`*_gen.go`) plus `bpmn/compile.lock.json`.
- Humans/agents implement:
  - Activity bodies (side effects),
  - ingress transport bindings (Kafka/NATS/webhooks),
  - domain-specific semantics and payload shaping,
  - any projections, UI wiring, and policy catalog contents.

### 10) Process primitive ships as a Temporal runner image

The process primitive produces a single container image that includes:

- a **worker** entrypoint (Temporal worker registering workflows/activities)
- an **ingress** entrypoint (consumes events, normalizes envelope, routes via SignalWithStart)

GitOps deploys two Deployments referencing the same image digest with different command/args.

### 11) Process proto surface is ops/control only

- A Process primitive does not need a business/state API: Domains own durable state; Projections serve read/query APIs.
- If a Process exposes a proto-defined API at all, it MUST be limited to ops/control-plane concerns (e.g., ping/health/build/definition info), and MUST NOT become a read model.

## CLI integration (fits existing `ameide` structure)

This work is intentionally designed to fit the existing CLI lifecycle under `ameide primitive`:

- `ameide primitive scaffold --kind process …` remains the canonical “create/refresh the Process skeleton” entrypoint.
- Primitive-level repo verification uses `ameide primitive verify --kind process --name <name> --mode repo`. Workspace-wide verification uses `ameide verify`.
- `ameide primitive drift --kind process …` can detect “BPMN changed but generated code wasn’t regenerated”.
- Publish images via CI to GHCR; GitOps consumes digest-pinned refs.

## CLI contract (v1)

This work intentionally follows existing `ameide primitive` patterns (no new top-level `ameide bpmn …` command group).

Repository convention (v1):

- The canonical BPMN source of truth for a Process primitive lives at `primitives/process/<name>/bpmn/process.bpmn`.
- Compilation output is generated-only: `internal/*/*_gen.go` plus `bpmn/compile.lock.json`.

Commands:

- `ameide primitive scaffold --kind process --name <name> …`:
  - creates/refreshes the Process skeleton,
  - compiles `primitives/process/<name>/bpmn/process.bpmn` into `internal/*/*_gen.go` plus `bpmn/compile.lock.json`,
  - never overwrites non-generated code.
- `ameide primitive verify --kind process --name <name> --mode repo`:
  - lints `bpmn/process.bpmn` against the v1 execution profile,
  - verifies generated code is up-to-date (no drift between BPMN and generated outputs),
  - fails the repo gate on any error.

## Acceptance criteria

- The Go linter enforces exactly the v1 decisions above, including:
  - workflowDefinition required
  - wait node type + subscription placement rules
  - template/path grammar rules
- `backlog/527-transformation-e2e-sequence-v3.bpmn` passes the Go linter with no errors.
- `ameide primitive scaffold --kind process …` deterministically generates `*_gen.go` plus `bpmn/compile.lock.json` without needing external tools.
- The generated process primitive builds a runner image that can run worker or ingress via args/entrypoint.

## Follow-ups (v2 candidates)

- Parallel/event-based gateway semantics (token concurrency) with explicitly defined compilation semantics.
- Boundary message events and event subprocess (with strict scope/activation rules).
- Typed payload contracts for subscriptions (optional schema binding).
