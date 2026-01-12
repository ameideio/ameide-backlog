# 511 – Process Primitive BPMN Conformance Suite (v1)

> **DEPRECATED (2026-01-12):** This conformance suite validates the v1 “BPMN compiled/executed via Temporal” posture.  
> New direction: BPMN-authored processes execute on Zeebe; conformance shifts to Zeebe deploy + worker coverage + Camunda runtime semantics.  
> See `backlog/511-process-primitive-bpmn-conformance-suite-v2.md` and `backlog/511-process-primitive-scaffolding-v2.md`.

This backlog documents the **implemented** conformance suite that keeps Ameide’s BPMN authoring semantics **Flowable-shaped** (where supported) and **Temporal-honest** (where executed), with a hard rule: **the diagram must not lie**.

The suite is a repo-owned guardrail:

- It provides one “feature coverage” BPMN that uses every **supported** profile v1 construct.
- It provides negative BPMN fixtures that use **unsupported** constructs and must fail with stable errors.
- It runs both compile-time checks and runtime semantics checks (Temporal Go SDK testsuite).

## What exists in the repo (2026-01)

**Reference primitive (positive fixture)**

- `primitives/process/bpmn_conformance_v1/bpmn/process.bpmn`
  - Intentionally includes: `serviceTask`, `sendTask`, `intermediateThrowEvent` (message), message waits (`receiveTask` + `intermediateCatchEvent`), `userTask` (Temporal Update), `exclusiveGateway`, and `subProcess`.
  - Every executable step has `bpmn:documentation` with the functional intent and the v1 execution semantics (“workflow wait” vs “activity work”).

**Negative fixtures (must fail)**

- `primitives/process/bpmn_conformance_v1/bpmn/negative/*.bpmn`
  - Timer catch/start, boundary events, event subprocess, parallel gateway, terminate end event, etc.

**Compiler/profile enforcement**

- Repo verification compiles BPMN and fails on drift:
  - `ameide primitive verify --kind process --name <name> --mode repo`
  - `ameide verify` runs the workspace-wide gate (repo-wide + all primitives).

## What is tested (v1 semantics)

The conformance runtime test asserts:

- **Deterministic branching:** `exclusiveGateway` evaluates outgoing `conditionExpression` in BPMN document order (restricted deterministic subset).
- **Message wait correlation:** only an envelope with matching `(message_name, correlation_key)` can satisfy the current wait.
- **Message dedupe:** duplicate `message_id` is ignored (prevents one delivery from satisfying multiple waits).
- **UserTask gate behavior:** the Update handler accepts exactly one decision per gate; duplicates are rejected.

The conformance compile test asserts:

- The reference BPMN compiles and generated artifacts are stable (no drift).
- Each negative fixture fails with a stable, intentional error message (profile rejection).

## How to run (developer loop)

- Compile + runtime semantics (fast, no Kubernetes): `go test -tags=integration ./...`
- Regenerate the primitive outputs after editing BPMN/policies: `./ameide primitive scaffold --kind process --name bpmn_conformance_v1`

## Why this exists

This suite is the mechanism that prevents “BPMN semantics drift”:

- If we say a BPMN construct is supported, the suite proves the compiler and generated workflow behave accordingly.
- If we say a construct is unsupported, the suite proves we reject it early (lint/verify), so a diagram cannot imply semantics we don’t execute.
