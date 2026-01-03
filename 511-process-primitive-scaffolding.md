# 511 – Process Primitive Scaffolding (Go, opinionated)

This backlog defines the **canonical target scaffold** for **Process** primitives.

- **Audience:** AI agents, Go developers, CLI implementers
- **Scope:** One opinionated Temporal/EDA pattern, aligned with `backlog/520-primitives-stack-v2.md`, `backlog/496-eda-principles.md`, and `514-primitive-sdk-isolation.md` (SDK-only). The CLI orchestrates repo/GitOps wiring; `buf generate` + plugins handle deterministic generation (SDKs, generated-only glue).

---

## Primitive/operator alignment

- **Operator responsibilities (495, 499, 497):** The Process operator reconciles `Process` CRDs into runtime infrastructure for Temporal workers and ingress (Deployments, Services, Routes, secrets, HPA). It owns definition references, Temporal namespace/task queue wiring, and health conditions, but never implements workflow logic itself.  
- **Primitive responsibilities (this backlog, 496):** The Process scaffold is responsible for the Temporal-side behavior: workflow/ingress code, idempotency state, fact emission, and SDK-based communication with Domains and other primitives. It runs inside the image referenced by the Process CRD and is the only place that knows about business processes and derived state.  
- **Boundary:** Operators act as a **compiler from Process CRDs to Temporal + K8s objects**, while Process primitives remain **pure workflow services** that consume facts and emit process events. 511’s scaffold should make it impossible to “fix infra in code” (that’s the operator’s job) and equally impossible for the operator to absorb business logic. This separation is defined in `495-ameide-operators.md`, `497-operator-implementation-patterns.md`, and `499-process-operator.md`.

---

## General implementation guidelines (for CLI scaffolder)

- Exactly one fixed, opinionated scaffold per primitive kind; **do not introduce new CLI flags** beyond the existing `ameide primitive scaffold` parameters.  
- All user instructions for Process scaffolds must be provided via **template files** (README/templates, code comment templates), not inline string literals in the CLI code.  
- Scaffolded documentation (README + comments) must be **self‑contained** and must **not reference external backlogs**; implementers should be able to follow the scaffold without opening design documents.  

---

## Grounding & cross‑references

- **Primitive stack:** `477-primitive-stack.md` (Process primitives in `primitives/process/{name}`).  
- **EDA / idempotency:** `470-ameide-vision.md`, `472-ameide-information-application.md`, `496-eda-principles.md`.  
- **CLI workflows:** `484-ameide-cli-overview.md`, `484a-ameide-cli-primitive-workflows.md`, `484f-ameide-cli-scaffold-implementation.md`.  
- **Primitive/operator contract:** `495-ameide-operators.md`, `497-operator-implementation-patterns.md`.
- **Testing discipline:** `537-primitive-testing-discipline.md` (RED→GREEN TDD pattern, Temporal determinism/replay testing, CI enforcement).
- **Process operator / vertical slice:** `499-process-operator.md`, `506-scrum-vertical-v2.md`.  
- **Capability implementation DAG:** `backlog/533-capability-implementation-playbook.md` (Process scaffold + implementation nodes).

---

## 1. Canonical scaffold command (Process / Go)

For Process primitives we use a single, opinionated Temporal pattern. Scaffolds are SDK/shape‑based and do not require a proto path:

```bash
ameide primitive scaffold \
  --kind process \
  --name <name> \
  --include-gitops \
  --include-test-harness
```

This assumes:

- Workflows will be implemented in Go using Temporal.  
- The scaffold will produce both a **worker** and an **ingress router** within `primitives/process/<name>/`.  
- The implementation must **implicitly choose Go** as the language for Process scaffolds (`--lang` is effectively fixed to `go` and treated as a compatibility flag only).  

---

## 2. Generated structure (Process / Go)

The Process scaffold shape makes Activities a first-class, explicit surface and prefers per-workflow files over a single `workflow.go`.

```text
primitives/process/<name>/
├── README.md                            # Scaffold command, process checklist
├── catalog-info.yaml                    # Backstage component
├── go.mod                               # Go module with Temporal + ameide-sdk-go
├── Dockerfile                           # Multi-stage build (worker image)
├── cmd/
│   ├── worker/
│   │   └── main.go                      # Temporal worker bootstrap (register workflows + activities)
│   └── ingress/
│       └── main.go                      # Ingress router: bus → Temporal SignalWithStart
└── internal/
    ├── workflows/
    │   └── <name>_workflow.go           # Workflow stubs (entity or orchestration workflows)
    ├── activities/
    │   └── activities.go                # Activity stubs (side-effect boundary; idempotent)
    ├── ingress/
    │   └── router.go                    # Fact→Signal routing stubs
    ├── tests/
    │   ├── <workflow>_workflow_test.go  # Workflow tests (RED)
    │   └── integration_mode.go          # Mode switch for integration tests
    └── process/
        └── state.go                     # Helper structures for idempotency/derived state

primitives/process/<name>/tests/
└── run_integration_tests.sh             # 430‑aligned integration runner
```

GitOps with `--include-gitops`:

```text
gitops/primitives/process/<name>/
├── values.yaml                          # Worker + ingress Deployments, Temporal config
├── component.yaml                       # Argo CD component descriptor
└── kustomization.yaml                   # Kustomize stub
```

---

## 3. Opinionated Temporal / EDA pattern (Process)

Process scaffolds assume:

- **Ingress router**:
  - Subscribes to one or more **domain facts** topics (e.g., `scrum.domain.facts.v1`).  
  - For each fact, computes a deterministic workflow ID (e.g., `product/{product_id}/sprint/{sprint_id}`) and uses the Temporal client to call `SignalWithStart` on the **Temporal service** (not directly on workers).  
  - Pins `WorkflowIDReusePolicy` explicitly for `SignalWithStart` (do not rely on defaults) and includes a test that asserts the selected policy for the primary workflows.  
  - Contains all non‑deterministic concerns (bus client, JSON/proto parsing, logging) and remains outside workflow code.

- **Workflows**:
  - Are deterministic and signal‑driven (no direct network calls or access to mutable runtime config).  
  - Maintain only **derived, process‑local state**:
    - `lastSeenAggregateVersion` per aggregate,
    - boolean flags for “already emitted” process facts (`SprintBacklogReadyForExecution`, etc.).  
  - Emit **process facts** (e.g., `ScrumProcessFact`) via an injected port or Activity that writes to an outbox / event bus, not via direct broker clients in workflow code.
  - Emit a Kanban/timeline-friendly progress stream (phase-first) as process facts, aligned with `backlog/520-primitives-stack-v2.md`:
    - include `process_definition_id`, `process_instance_id` (Workflow Id), `process_run_id` (Run Id), and stable ordering fields (`run_epoch`, `seq`),
    - emit step-level facts only when explicitly required, and include both `step_id` and `step_instance_id` for BPMN-derived steps.
    - reuse the minimal capability-agnostic progress vocabulary from `backlog/509-proto-naming-conventions.md` (§4.1.1) unless there is a deliberate, documented exception.

- **Execution boundary (Activities; inline vs delegated)**:
  - Workflows MUST initiate all side effects via **Activities** (the side‑effect boundary); workflow code must not call external systems directly.
  - Activities have two allowed execution modes:
    - **Inline deterministic execution:** the Activity runs the deterministic work directly inside the Process worker (preferred when the toolchain/isolation fits the worker image).
    - **Delegated execution:** the Activity requests work across a runtime boundary (agentic coding, NiFi flows, devcontainer/heavy toolchains, external executors). Delegation MUST follow `496-eda-principles.md` semantics:
      - the Activity submits a **domain intent/command** (idempotent) to create a Domain-owned `WorkRequest` (audit), and the Domain emits:
        - lifecycle **facts** (audit/outcomes) via outbox on the fact spine, and
        - a point‑to‑point execution **intent** on an operational execution queue (not a fact topic).
      - executors consume execution **intents** and record outcomes back to the owning Domain write surface idempotently; the Domain emits completion facts.
      - workflow advancement is driven by **awaiting correlated facts/receipts**, not by direct runner responses.
  - Agent and Integration primitives remain separate DUs with their own runtimes (LangGraph, NiFi); Process delegates to them via Activities and advances based on correlated outcomes.

- **Idempotency**:
  - Workflows must ignore domain facts where `aggregate_version <= lastSeenAggregateVersion`.  
  - Activities that emit process facts must be **idempotent** (e.g., accept a stable idempotency key per fact), because Temporal may retry Activities.  
  - Process facts are emitted exactly once per logical condition by checking “already emitted” flags and using idempotent Activities/ports.

- **Envelope + trace propagation (EDA spine; required)**:
  - Ingress must treat all consumed facts as at-least-once and must propagate required metadata into workflow signals and downstream intents:
    - scope: `tenant_id` (and any other capability-defined scope ids),
    - traceability: `message_id`, `correlation_id`, `causation_id`,
    - tracing: `traceparent` (and optional `tracestate`).
  - Process facts emitted as orchestration evidence must carry the same correlation/trace context so projections can materialize audit-grade timelines.

- **Determinism and definitions**:
  - BPMN / ProcessDefinition artifacts from Transformation are canonical **design inputs**. In v1 they may be treated as “design knowledge”, but the target architecture supports **BPMN → Temporal transpilation**; therefore Process implementations MUST carry a stable `process_definition_id` (definition key + version/checksum) in workflow inputs and emitted process facts.
  - Workflows must not consult “latest definition from ConfigMap/registry” on each replayed step. Instead:
    - capture a **definition version/checksum** as workflow input when starting, and
    - treat that version as immutable per Workflow Execution, or
    - record any derived routing/decision data into workflow history via an Activity/SideEffect once, then use the recorded value.
  - Definition changes are handled at coarse‑grained boundaries (e.g., **Continue‑As‑New** or new Workflow Executions) rather than by changing behavior within a running execution based on live config.
  - When emitting BPMN‑derived step progress, include both `step_id` (definition element id) and `step_instance_id` (runtime occurrence id) to support loops, multi‑instance tasks, and parallel tokens.

- **Signal ordering / initialization**:
  - When using `SignalWithStart`, Temporal may deliver Signals before the workflow’s main logic has fully initialized. Scaffolded workflows are expected to:
    - initialize `State` and any required fields **before** entering the signal handling loop, and
    - treat Signals as potentially arriving at any time in the workflow’s lifetime.

- **Long‑running entity workflows**:
  - Process workflows are expected to behave like **entity workflows** (one workflow per business key) and may run for a long time. To avoid unbounded history and ease versioning:
    - scaffolds should plan for **Continue‑As‑New** once a history size or event count threshold is reached, or when a definition version changes.

Topic and event naming follow `509-proto-naming-conventions.md` and the relevant process proto (e.g., `ameide_core_proto.process.scrum.v1`).

---

## 4. Workflow and test semantics (Process / Go)

Scaffolded workflows:

- Live under `internal/workflows/**` (one file per workflow).  
- Provide a function signature like:

```go
func ExampleWorkflow(ctx context.Context, state *process.State) error
```

with placeholder state and TODOs for:

- initializing `process.State` before handling signals,  
- tracking `lastSeenAggregateVersion`,  
- scheduling timers,  
- emitting process facts via idempotent Activities/ports,  
- optionally performing **Continue‑As‑New** after configurable thresholds.

Scaffolded tests:

- Live under `internal/tests/**` (one file per workflow).  
- Are RED by default:
  - invoke workflows through the Temporal test harness,
  - assert TODO behavior,
  - provide comments pointing to `506-scrum-vertical-v2.md` / `496-eda-principles.md` for the expected behavior and Temporal’s determinism/idempotency constraints.

Implementers (humans or coding agents) are expected to:

1. Fill in workflow logic (timers, derived state, process fact emissions) in a way that respects Temporal determinism and idempotent Activities.  
2. Wire ingress router to call workflow signals based on domain facts using a Temporal client and deterministic workflow IDs.  
3. Update tests to assert correct behavior, idempotency, and (where appropriate) Continue‑As‑New behavior.
4. For any side‑effect step, implement the step as an Activity:
   - run deterministically inline when feasible, or
   - delegate via the WorkRequest + execution intent queue pattern when toolchain/isolation demands it,
   - and advance the workflow only on correlated facts/receipts (not synchronous runner responses).

---

## 5. Verification expectations

`ameide primitive verify --kind process --name <name>` is expected to check:

- Presence of `cmd/worker/main.go` and `cmd/ingress/main.go`.  
- Temporal worker registration of workflows/activities under `internal/workflows/**` and `internal/activities/**`.  
- Ingress router using deterministic workflow IDs and `SignalWithStart`.  
- Idempotency state present in workflow code (`lastSeenAggregateVersion`, flags).  
- Process facts published via a port or activity (not directly from workflows), consistent with EDA rules in `496-eda-principles.md`.
- Work execution posture is explicit in scaffold docs and stubs: deterministic steps run inline as Activities when feasible; delegated steps are requested via domain intents (execution intents on queues) and advanced by awaiting correlated facts/receipts.

Vertical slices like `506-scrum-vertical-v2.md` remain authoritative for **which workflows and facts** exist; this backlog only constrains the **scaffold shape and Temporal/EDA pattern** for Process primitives.
