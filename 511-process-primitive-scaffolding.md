# 511 – Process Primitive Scaffolding (Go, opinionated)

> **Status update (520/521):** This backlog specifies the Process scaffold produced by the Ameide CLI (`ameide primitive scaffold`). The consolidated approach is a split: the CLI orchestrates scaffolding + external wiring (repo layout, GitOps), and `buf generate` + plugins handle internal deterministic generation (SDKs, generated-only glue). See `backlog/520-primitives-stack-v2.md` and `backlog/521-code-generation-improvements.md`.

**Status:** Active reference (aligned with 520/521)  
**Audience:** AI agents, Go developers, CLI implementers  
**Scope:** Exact scaffold shape and patterns for **Process** primitives. One opinionated Temporal/EDA pattern, aligned with `514-primitive-sdk-isolation.md` (SDK-only, self-contained primitives), no new Process-specific CLI parameters beyond the canonical `ameide primitive scaffold` flags.

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

```text
primitives/process/<name>/
├── README.md                            # Scaffold command, process checklist
├── catalog-info.yaml                    # Backstage component
├── go.mod                               # Go module with Temporal + ameide-sdk-go
├── Dockerfile                           # Multi-stage build (worker image)
├── cmd/
│   ├── worker/
│   │   └── main.go                      # Temporal worker bootstrap (register workflows)
│   └── ingress/
│       └── main.go                      # Ingress router: bus → Temporal SignalWithStart
└── internal/
    ├── workflows/
    │   └── <name>_workflow.go           # Workflow stubs (entity or orchestration workflows)
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

- **Idempotency**:
  - Workflows must ignore domain facts where `aggregate_version <= lastSeenAggregateVersion`.  
  - Activities that emit process facts must be **idempotent** (e.g., accept a stable idempotency key per fact), because Temporal may retry Activities.  
  - Process facts are emitted exactly once per logical condition by checking “already emitted” flags and using idempotent Activities/ports.

- **Determinism and definitions**:
  - BPMN / ProcessDefinition artifacts from Transformation are **design knowledge only**; they are not compiled into Temporal at runtime. The Process primitive’s code, typically authored via agentic development, implements workflows that follow these designs.
  - Workflows must not consult “latest definition from ConfigMap/registry” on each replayed step. Instead:
    - capture a **definition version/checksum** as workflow input when starting, and
    - treat that version as immutable per Workflow Execution, or
    - record any derived routing/decision data into workflow history via an Activity/SideEffect once, then use the recorded value.
  - Definition changes are handled at coarse‑grained boundaries (e.g., **Continue‑As‑New** or new Workflow Executions) rather than by changing behavior within a running execution based on live config.

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

Scaffolded workflows (target shape and current scaffold):

- Live under `internal/workflows/workflow.go` in the current scaffold (per-process files like `<name>_workflow.go` remain the long-term goal).  
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

Scaffolded tests (target shape):

- Live under `internal/tests/process_workflow_test.go` in the current scaffold (per-workflow test files are the long-term goal).  
- Are RED by default:
  - invoke workflows through the Temporal test harness,
  - assert TODO behavior,
  - provide comments pointing to `506-scrum-vertical-v2.md` / `496-eda-principles.md` for the expected behavior and Temporal’s determinism/idempotency constraints.

Implementers (humans or coding agents) are expected to:

1. Fill in workflow logic (timers, derived state, process fact emissions) in a way that respects Temporal determinism and idempotent Activities.  
2. Wire ingress router to call workflow signals based on domain facts using a Temporal client and deterministic workflow IDs.  
3. Update tests to assert correct behavior, idempotency, and (where appropriate) Continue‑As‑New behavior.

---

## 5. Verification expectations

`ameide primitive verify --kind process --name <name>` is expected to check:

- Presence of `cmd/worker/main.go` and `cmd/ingress/main.go`.  
- Temporal worker registration of workflows/activities under `internal/workflows/**`.  
- Ingress router using deterministic workflow IDs and `SignalWithStart`.  
- Idempotency state present in workflow code (`lastSeenAggregateVersion`, flags).  
- Process facts published via a port or activity (not directly from workflows), consistent with EDA rules in `496-eda-principles.md`.

Vertical slices like `506-scrum-vertical-v2.md` remain authoritative for **which workflows and facts** exist; this backlog only constrains the **scaffold shape and Temporal/EDA pattern** for Process primitives.

---

## 6. Implementation progress (CLI & scaffold)

This section describes the current implementation status of 511 in the CLI (`packages/ameide_core_cli`) and repo scaffolds. It is descriptive; the rest of 511 remains the target spec.

### 6.1 Scaffolder behavior for Process primitives

**Status:** Implemented as a Process-specific Go scaffold shape (worker + ingress + workflows), SDK/shape-based and proto-path free.

- `ameide primitive scaffold --kind process`:
  - Uses a dedicated Process shape in the shared Go scaffold path (`primitive_scaffold.go`):
    - Creates `primitives/process/<name>/go.mod` with `go.temporal.io/sdk` and `github.com/ameideio/ameide-sdk-go` dependencies, a worker-focused `Dockerfile`, and `catalog-info.yaml`.
    - Generates `cmd/worker/main.go` – a worker entrypoint that:
      - reads `PROCESS_NAME`, `TEMPORAL_NAMESPACE`, `TEMPORAL_TASK_QUEUE`, and `TEMPORAL_ADDRESS`,
      - creates a Temporal client (`go.temporal.io/sdk/client`),
      - creates a worker (`go.temporal.io/sdk/worker`) for the configured task queue,
      - registers `workflows.ExampleWorkflow`, and
      - calls `worker.Run(worker.InterruptCh())`.
    - Generates `cmd/ingress/main.go` – an ingress entrypoint that:
      - reads `PROCESS_NAME`, `PROCESS_INGRESS_SOURCE`, `TEMPORAL_NAMESPACE`, `TEMPORAL_TASK_QUEUE`, and `TEMPORAL_ADDRESS`,
      - creates a Temporal client,
      - logs the configured source/namespace/task queue/address, and
      - includes TODO comments to subscribe to domain facts and call `SignalWithStartWorkflow` with deterministic workflow IDs.
    - Generates `internal/workflows/workflow.go` – an `ExampleWorkflow` stub that:
      - uses `workflow.Context` from `go.temporal.io/sdk/workflow`,
      - depends on `internal/process.State`,
      - includes TODO comments for timers, signals, idempotency, and Continue-As-New.
    - Generates `internal/ingress/router.go` – a `Router` stub with `HandleFact` placeholder.
    - Generates `internal/process/state.go` – a `State` struct with `LastSeenAggregateVersion` and TODOs for derived state/idempotency flags.
    - Generates a RED test `internal/tests/process_workflow_test.go` that references the workflow stub and reminds implementers to replace it with real assertions.
    - Adds a shared integration test harness when `--include-test-harness` is set.
  - Enforces Go as the canonical language:
    - `runScaffold` rejects `--lang` values other than `go` for Process scaffolds and defaults `--lang` to `go` when omitted, matching 514’s P4 rule for Process primitives.

- Templates vs inline strings:
  - Process-specific templates under `templates/process/**` now exist and are wired via `templates_process.go`:
    - README: `templates/process/readme.md.tmpl`.
    - Worker main: `templates/process/cmd/worker_main.go.tmpl`.
    - Ingress main: `templates/process/cmd/ingress_main.go.tmpl`.
    - Workflow stub: `templates/process/internal/workflows/workflow.go.tmpl`.
    - Ingress router stub: `templates/process/internal/ingress/router.go.tmpl`.
    - State struct: `templates/process/internal/process/state.go.tmpl`.
  - `buildReadmeContent` renders Process/Go READMEs from the template instead of inline strings, and `templates_process_test.go` includes `TestProcessReadmeTemplateHasNoBacklogIds` to ensure the template stays free of backlog ID references.

### 6.2 Verify behavior for Process primitives

**Status:** Generic checks plus a Process-specific shape check.

- `primitive verify --kind process --name <name>`:
  - Runs generic repo checks:
    - Naming (`checkNamingConventions`),
    - Security/SAST/secret scan,
    - Dependency vulnerabilities (for Go modules),
    - Tests (`go test` under the primitive),
    - Optional Buf / GitOps checks when configured.
  - Adds a Process-specific scaffold check:
    - `checkProcessShape(kind, serviceDir)` ensures the following files exist:
      - `cmd/worker/main.go`
      - `cmd/ingress/main.go`
      - `internal/workflows/workflow.go`
      - `internal/ingress/router.go`
      - `internal/process/state.go`
    - Fails when any of these are missing with a `ProcessShape` issue.

### 6.3 Known gaps and next steps

- Scaffold:
  - Enrich `internal/workflows` and `internal/ingress` stubs with stronger examples (e.g., example Activities, signal handlers, Continue-As-New patterns) once Temporal patterns are finalized.
- Verify:
  - Extend Process-specific checks to understand Temporal configuration and idempotency/fact emission conventions (currently out of scope for `checkProcessShape`).
