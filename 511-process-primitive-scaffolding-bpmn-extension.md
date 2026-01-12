# 511 – BPMN Extensions for Process Primitive Scaffolding (direction)

> **DEPRECATED (2026-01-12):** This document describes Ameide BPMN extensions as compilation inputs for a Temporal-backed Process primitive (v1).  
> New direction: BPMN-authored processes execute on Zeebe; Ameide extensions are design-time contracts for worker/agent side-effects and verification, not transpilation directives.  
> See `backlog/511-process-primitive-scaffolding-v2.md`.

This document captures the direction for an Ameide-owned **BPMN extension profile** that makes BPMN models **scaffoldable** into a **Temporal-backed Process primitive** with low ambiguity and strong validation.

The point of these extensions is not to fully specify the implementation. The point is to make the model a **compilable contract** so scaffolding (and a coding agent) can produce a consistent, testable skeleton.

Crossreference: the Process scaffold shape and invariants live in `backlog/511-process-primitive-scaffolding.md`.
CLI integration and the Go-only compiler refactor plan (implemented under `ameide primitive scaffold/verify`): `backlog/511-process-primitive-scaffolding-refactoring.md`.

---

## Goal (scaffolding + AI coding agent consumption)

Define a **tight, lintable extension profile** such that:

- the same BPMN yields the same scaffold shape (no heuristics),
- execution boundaries are explicit (determinism + idempotency),
- “wait for external outcomes” is modeled consistently (correlation + dedupe),
- the profile is small enough to stay stable, but strict enough to prevent “diagram-only” models.

**Scaffolding objective (what gets generated)**

- workflow skeleton(s) / state-machine shape,
- Activity stubs (side-effect boundaries),
- Update handlers for gates (user decisions),
- “await correlated fact/event” helpers and signal-routing stubs.

**AI coding agent consumption (what happens next)**

After scaffolding, a coding agent (or human) fills in the missing bodies and wiring:

- implement Activity bodies (domain write calls, external job requests, event emission),
- implement Update validation + gate state transitions,
- connect ingress routing to real event/fact topics + dedupe strategy,
- fill IO mappings and persistence of large artifacts (by reference),
- harden retries/timeouts and add tests within platform conventions.

---

## Stance on naming (not black-or-white)

We intentionally sit in the middle:

- **Camunda-inspired vocabulary/structure** where it reduces author friction,
- **Ameide-owned namespace + Ameide semantics** (we are not configuring a BPMN engine),
- “Camunda-like naming” MUST NOT be read as Zeebe/Camunda runtime compatibility.

---

## Camunda alignment and intentional divergences (trackable)

This profile borrows heavily from Camunda’s modeling vocabulary because it is familiar and BPMN-tool friendly. However, the execution target is a **compiled Temporal runtime**, not a BPMN engine. The items below are explicitly tracked so “Camunda-ish naming” does not hide semantic differences.

Aligned (by intent)

- Message waits are explicit BPMN wait nodes (catch/receive) and create an explicit subscription/wait contract (same mental model as message subscriptions existing only when a process reaches a wait point).
- Correlation is frozen when entering a wait (matches the “subscription value is evaluated at activation and not updated” intuition).
- `bpmn:message@name` + `message_id` dedupe semantics exist as first-class concepts (name + dedupe).
- IO mapping is deterministic and ordered (Camunda-like “ioMapping” semantics without scripting).
- Static headers exist as a small, literal metadata surface (Camunda-like “task headers” idea).

Intentional divergences (v1)

- **Design-time compilation, not runtime engine config**: Ameide extensions are compiler directives for scaffolding Temporal code; they do not configure a BPMN engine’s runtime behavior.
- **No general expression language**: we forbid FEEL/scripting in v1 and allow only small templates. Reason: reduce nondeterminism/hidden computation and keep the model lintable.
- **Correlation uses a single key value**: we intentionally align with Camunda’s “single correlation key” mental model. In v1 this is expressed as `correlationKeyTemplate` (a deterministic template that resolves to one string). Reason: removes ambiguity and maps cleanly to systems that only support a single correlation key value; avoids “tuple encoding” drift.
- **Policy indirection (`policyRef`) instead of per-node numeric retries/timeouts**: we reference a named policy and require the scaffolder to resolve + materialize it into generated code. Reason: keep BPMN small while keeping Temporal behavior bounded and deterministic; ensure policy drift shows up in code diffs.
- **Human gates compile to Temporal Updates**: BPMN `userTask` defaults to an Update surface rather than engine-managed user task semantics. Reason: synchronous accept/reject semantics and a stable idempotency surface in Temporal; UI/assignment/form metadata is intentionally out-of-scope in v1.
- **Buffering/TTL is not an engine feature**: any “message arrived before subscription exists” buffering belongs to ingress/broker retention/retry, not the workflow itself. Reason: we do not run a BPMN engine with internal message buffering.

These divergences are not “forever decisions”; they are v1 constraints designed to keep scaffolding deterministic and small.

---

## Principles

1. **Predictability over flexibility**: defaults exist only where unambiguous; otherwise the validator rejects the model.
2. **BPMN types carry control-flow**: subprocesses, gateways, loops remain in BPMN; extensions bind nodes to execution surfaces.
3. **One binding per executable node**: every executable node has exactly one binding (or a single default rule applies).
4. **Waiting is a contract**: waits declare a correlation key and a dedupe key; correlation is frozen when entering the wait.
5. **Determinism + idempotency are mandatory**: side effects require explicit idempotency; Activities are bounded (timeouts exist via explicit fields or well-defined defaults).
6. **No hidden programming language**: keep expressions out of the model; allow only simple templates for identity/idempotency.
7. **Profiles separate concerns**: required execution profile first; optional UI/integration profile later, without changing execution semantics.

---

## Profiles

### Execution profile (required)

Defines what is needed to scaffold Process primitive code deterministically:

- the **execution binding** per node (how it compiles),
- the **await contract** for waits (what ends the wait + correlation + dedupe),
- optional but recommended IO mapping and static headers.

### UI profile (optional, out of scope here)

Forms/assignments/text for human tasks. Must not change execution semantics.

### Integration profile (optional, out of scope here)

Optional wiring hints (topic names, queue names, catalogs). Must remain small and environment-agnostic.

---

## Namespace and placement

- Namespace (versioned): `https://ameide.io/schema/bpmn/extensions/v1`
- Prefix: `ameide`
- All Ameide elements MUST appear under `bpmn:extensionElements`.
- XSD (namespace elements only): `backlog/511-process-primitive-scaffolding-bpmn.xsd`

Example:

```xml
<bpmn:definitions
  xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:ameide="https://ameide.io/schema/bpmn/extensions/v1">
```

---

## Required vs nice-to-have (v1)

**Required**

- `ameide:workflowDefinition` on the `bpmn:process`
- `ameide:taskDefinition` on executable automated work nodes
- `ameide:subscription` for explicit waits (MUST be attached to the referenced `bpmn:message` in v1)

**Nice-to-have**

- `ameide:ioMapping` (recommended)
- `ameide:taskHeaders` (optional)
- `ameide:updateDefinition` (optional; user tasks default to Updates)

Why those two are required:

 - `taskDefinition` is required because BPMN does not define the policy/type/idempotency key surface needed for deterministic Activity execution.
- `subscription` is required because BPMN wait nodes do not define a correlation key or a dedupe key; without them the scaffold cannot generate a correct “await correlated message” shape.

---

## BPMN element → required extension matrix (v1)

This matrix exists so validators and scaffolding do not drift into implicit heuristics.

| BPMN element type | Requires `ameide:taskDefinition` | Requires `ameide:subscription` | Notes |
| --- | --- | --- | --- |
| `bpmn:serviceTask` | Yes | No | Automated work boundary; compiles to a Temporal Activity. |
| `bpmn:sendTask` | Yes | No | Automated work boundary; compiles to a Temporal Activity. |
| `bpmn:intermediateThrowEvent` (message) | Yes | No | Executable “publish message” step; compiles to a Temporal Activity. |
| `bpmn:userTask` | No (default Update) | No | Compiles to Temporal Update by default; optional `ameide:updateDefinition@name`. |
| `bpmn:receiveTask` | No | Yes | Explicit wait boundary (receiveTask@messageRef → referenced `bpmn:message`). |
| `bpmn:intermediateCatchEvent` (message) | No | Yes | Explicit wait boundary (messageEventDefinition@messageRef → referenced `bpmn:message`). |

Notes:

- This is the execution profile only. UI/integration annotations can exist but must not change which nodes require the execution elements above.

---

## Core execution elements (v1)

### A) Process-level: `ameide:workflowDefinition` (required)

Placement: `bpmn:process/bpmn:extensionElements`

Purpose: provide the minimal metadata required to pin definition identity and derive deterministic workflow identity.

Requiredness:

- Required in BPMN for v1 compilation (the model is treated as the compiler contract).

Minimum fields:

- `definitionId` (stable identifier; version/checksum)
- `workflowType` (default = BPMN process id)
- `workflowIdTemplate` (deterministic identity template)

Notes:

- `workflowIdTemplate` is a template, not a general expression language.

### B) Work binding: `ameide:taskDefinition` (required on automated executable nodes)

Placement (examples): `bpmn:serviceTask`, `bpmn:sendTask`, message throw events.

Purpose: define the **execution binding** that the scaffolder must generate.

Requiredness:

- Required on automated nodes that represent executable work.
- If a BPMN node has no `ameide:taskDefinition` and is not covered by a single allowed default, the validator rejects the model.

Minimum fields:

- `type` (required): stable identifier for the binding (activity type name / logical external job type / command key)
- `idempotencyKeyTemplate` (optional): stable idempotency key derivation for the side-effect boundary (a deterministic default is applied if omitted)
- `policyRef` (required): reference to a named timeout/retry policy (see below)

Forbidden (v1):

- `ameide:taskDefinition@implementation` MUST NOT be present (all executable tasks compile to Temporal Activities in v1).

Completion semantics (normative for v1):

- All executable work nodes compile to Temporal Activities; long waits are modeled as explicit BPMN wait states and compiled to Workflow waits (Signals/Updates), not “blocked Activities”.

Timeouts/retries:

- The scaffold MUST ensure Activities are bounded (timeouts exist) and retry behavior is defined.
- `policyRef` is the v1 mechanism to avoid embedding numeric timeouts/retry knobs into BPMN while still keeping scaffolding deterministic.
- The scaffolder resolves `policyRef` to concrete Temporal ActivityOptions/RetryPolicy and MUST materialize the resolved values in generated code (so policy is reviewable and drift is visible in diffs).
- `policyRef` resolution MUST be deterministic given the pinned scaffolder/templates version; the scaffolding pipeline is responsible for pinning versions so “same BPMN + same toolchain inputs → same scaffold outputs” holds.

### C) Static config: `ameide:taskHeaders` (optional)

Placement: on the same node as `ameide:taskDefinition`.

Purpose: small, static key/value metadata passed to the Activity boundary (or recorded as step config).

Constraints:

- literal values only (no expressions),
- must not contain environment-specific endpoints.

### D) IO mapping: `ameide:ioMapping` (recommended)

Placement: on executable nodes (tasks) and call-activity bindings (if adopted later).

Purpose: deterministic data shaping from workflow state → request and response → workflow state.

Constraints:

- ordered evaluation,
- missing sources become `null`/unset,
- targets are created if absent,
- no scripts; only field-path selection and assignment.
- `source` and `target` values MUST follow the Path grammar below (no expressions).

### E) Wait correlation: `ameide:subscription` (required for explicit waits)

Placement:

- Message waits (`bpmn:intermediateCatchEvent` with `messageRef`): `ameide:subscription` MUST be on the referenced `bpmn:message/bpmn:extensionElements`.
- Receive waits (`bpmn:receiveTask`): `ameide:subscription` MUST be on the referenced `bpmn:message/bpmn:extensionElements` (receiveTask@messageRef).

Purpose: define how a waiting point correlates incoming events/facts to a workflow execution and dedupes deliveries.

Requiredness:

- Required on BPMN nodes that represent “wait for external outcome” where the workflow must correlate and dedupe.

Minimum fields:

- `correlationKeyTemplate` (required): deterministic template that resolves to a single correlation key value
- `messageIdPath` (required): MUST equal `message_id` (dedupe key in the normalized envelope)

Semantics:

- correlation values are computed once when entering the wait state and treated as fixed for that wait.
- `correlationKeyTemplate` is resolved once when entering the wait, producing a single correlation key value.
- Incoming events/facts are matched by comparing the inbound correlation key value to the captured expected correlation key value (string equality).
- `messageIdPath="message_id"` is used for dedupe; duplicate deliveries with the same message id MUST be ignored.

Inbound envelope requirement (normative for v1):

- The ingress layer MUST deliver inbound messages/facts to workflows in a normalized envelope that contains:
  - `correlation_key` (string): the inbound correlation key value used for matching waits
  - `message_id` (string): a stable idempotency key for dedupe
  - `message_name` (string): the logical message name/type (for message-based waits this SHOULD match `bpmn:message@name`)
- `correlationKeyTemplate` is resolved from workflow state when entering the wait and compared against the inbound `correlation_key`.
- In v1, the inbound correlation key value is always taken from `correlation_key` in the normalized envelope (it is not configurable per-subscription in BPMN).
- In v1, `messageIdPath` is fixed to `message_id` to keep wait dedupe semantics unambiguous.

Recommended BPMN modeling:

- Model message-based waits as BPMN message waits (`bpmn:intermediateCatchEvent` with a `bpmn:messageEventDefinition messageRef="..."`) and attach the subscription to the referenced `bpmn:message`.

---

## Human gates (execution core)

Default rule:

- `bpmn:userTask` compiles to a **Temporal Update** by default.
- If `ameide:updateDefinition` is not provided, the default Update name MUST be derived deterministically from the BPMN element id (e.g., `Update_<userTaskId>`).

Optional override:

- `ameide:updateDefinition` MAY be used to set an explicit Update name.

Notes:

- This file is about execution scaffolding only. Any “form/assignment/UI” metadata is a separate (optional) profile.

---

## Expressions and templates

To avoid hidden computation inside BPMN:

- allow only **templates** for identity/idempotency fields (e.g., `${process_instance_id}`, `${step_id}`)
- do not allow general expressions or scripting in v1.

### Template grammar (v1)

Templates are UTF-8 strings that may contain placeholders of the form `${...}`.

Allowed placeholders:

- `${process_instance_id}`
- `${process_run_id}`
- `${step_id}`
- `${step_instance_id}`
- `${state.<path>}` where `<path>` follows the Path grammar below

Rules:

- placeholders are substituted from workflow state that is already present in memory/history (no external reads)
- placeholders MUST be deterministic on replay
- escaping: `\${` is treated as a literal `${`

### Path grammar (v1)

Path strings are dot-separated identifiers:

- identifier: `[A-Za-z_][A-Za-z0-9_]*`
- path: `identifier ( "." identifier )*`

Constraints:

- v1 does not support array indexing; keep variable shapes map/object-like
- missing path resolution:
  - for `ioMapping` sources: missing resolves to `null`/unset
  - for templates used as idempotency keys: missing SHOULD cause validation failure (to avoid unstable keys)

---

## Lint rules (must be enforceable)

The validator should be able to reject models that cannot be scaffolded deterministically.

Required rules:

- The `bpmn:process` includes exactly one `ameide:workflowDefinition`.
- Every executable automated node has exactly one `ameide:taskDefinition`.
- Every `ameide:taskDefinition` has `type` and `policyRef` (and may include `idempotencyKeyTemplate`).
- Every explicit wait references a `bpmn:message` that has exactly one `ameide:subscription` with `correlationKeyTemplate` and `messageIdPath="message_id"`.
- Template placeholders MUST be from the allowed set, and `${state.<path>}` MUST use a `<path>` that follows the Path grammar.

Recommended rules:

- Prefer `ameide:ioMapping` on tasks that cross a side-effect boundary.

---

## Example snippets (abstract)

### Delegated external job + explicit wait for completion

```xml
<bpmn:serviceTask id="Task_DoWork" name="Do work">
  <bpmn:extensionElements>
    <ameide:taskDefinition type="example.external_job.v1"
                           idempotencyKeyTemplate="example/${process_instance_id}/job/${state.job_key}"
                           policyRef="default" />
    <ameide:ioMapping>
      <ameide:input source="state.input_ref" target="request.input_ref" />
      <ameide:output source="result.external_job_id" target="state.external_job_id" />
    </ameide:ioMapping>
  </bpmn:extensionElements>
</bpmn:serviceTask>

<bpmn:sequenceFlow id="Flow_1" sourceRef="Task_DoWork" targetRef="Wait_ExternalJobCompleted"/>

<bpmn:message id="Message_ExternalJobCompleted" name="ExternalJobCompleted">
  <bpmn:extensionElements>
    <ameide:subscription correlationKeyTemplate="${state.external_job_id}"
                         messageIdPath="message_id"/>
  </bpmn:extensionElements>
</bpmn:message>

<bpmn:intermediateCatchEvent id="Wait_ExternalJobCompleted" name="ExternalJobCompleted">
  <bpmn:messageEventDefinition messageRef="Message_ExternalJobCompleted"/>
</bpmn:intermediateCatchEvent>

<bpmn:sequenceFlow id="Flow_2" sourceRef="Wait_ExternalJobCompleted" targetRef="Next_Node"/>
```

---

## v1 decisions (closed)

1. Message-based waits use message-level subscriptions (`bpmn:message` + `messageRef`); wait-node-level subscriptions are not used for message waits in v1.
2. `ameide:workflowDefinition` is required in BPMN for v1 compilation.
3. `policyRef` resolves against a pinned catalog at compile time; the resolution mechanism is a scaffolder/CLI concern (out-of-band to BPMN).
