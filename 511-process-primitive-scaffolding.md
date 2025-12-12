# 511 – Process Primitive Scaffolding (Go, opinionated)

**Status:** Draft  
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
- **Process operator / vertical slice:** `499-process-operator.md`, `506-scrum-vertical-v2.md`.  

---

## 1. Canonical scaffold command (Process / Go)

For Process primitives we use a single, opinionated Temporal pattern:

```bash
ameide primitive scaffold \
  --kind process \
  --name <name> \
  --proto-path <path/to/process_api.proto> \
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
  - For each fact, computes a deterministic workflow ID (e.g., `product/{product_id}/sprint/{sprint_id}`) and calls `SignalWithStart`.  
  - Contains all non‑deterministic concerns (bus client, JSON/proto parsing, logging).

- **Workflows**:
  - Are deterministic and signal‑driven (no direct network calls).  
  - Maintain only **derived, process‑local state**:
    - `lastSeenAggregateVersion` per aggregate,
    - boolean flags for “already emitted” process facts (`SprintBacklogReadyForExecution`, etc.).  
  - Emit **process facts** (e.g., `ScrumProcessFact`) via an injected port or activity that writes to an outbox / event bus, not via direct broker clients in workflow code.

- **Idempotency**:
  - Workflows must ignore domain facts where `aggregate_version <= lastSeenAggregateVersion`.  
  - Process facts are emitted exactly once per logical condition by checking “already emitted” flags.

Topic and event naming follow `509-proto-naming-conventions.md` and the relevant process proto (e.g., `ameide_core_proto.process.scrum.v1`).

---

## 4. Workflow and test semantics (Process / Go)

Scaffolded workflows:

- Live under `internal/workflows/<name>_workflow.go`.  
- Provide function signatures like:

```go
func <Name>Workflow(ctx workflow.Context, params Params) error
```

with placeholder state and TODOs for:

- tracking `lastSeenAggregateVersion`,  
- scheduling timers,  
- emitting process facts.

Scaffolded tests:

- Live under `internal/tests/<workflow>_workflow_test.go`.  
- Are RED by default:
  - invoke workflows through the Temporal test harness,
  - assert `Unimplemented` or TODO behavior,
  - provide comments pointing to `506-scrum-vertical-v2.md` / `496-eda-principles.md` for the expected behavior.

Agents/humans are expected to:

1. Fill in workflow logic (timers, derived state, process fact emissions).  
2. Wire ingress router to call workflow signals based on domain facts.  
3. Update tests to assert correct behavior and idempotency.

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

**Status:** Partially implemented; Process currently uses the generic Go scaffold, not the Temporal-specific shape described above.

- `ameide primitive scaffold --kind process`:
  - Shares the same code path as Domain/Go in `primitive_scaffold.go`:
    - Creates `primitives/process/<name>/go.mod`, `Dockerfile`, `catalog-info.yaml`.
    - Generates a single `cmd/main.go` that logs a placeholder and calls `internal/handlers.New()`.
    - Generates `internal/handlers/handlers.go` with one method per RPC returning `codes.Unimplemented`.
    - Generates per-RPC RED tests under `internal/tests/<rpc>_test.go` and an integration harness script.
  - Does **not** currently emit:
    - `cmd/worker/main.go` / `cmd/ingress/main.go`.
    - `internal/workflows/**`, `internal/ingress/router.go`, or `internal/process/state.go`.
    - Temporal-specific `go.mod` dependencies or worker bootstrap code.

- Templates vs inline strings:
  - Process-specific templates under `templates/process/**` do not yet exist.
  - Process README and comments are currently produced by the generic `buildReadmeContent` and handler/test builders, not 511-specific templates.

### 6.2 Verify behavior for Process primitives

**Status:** Generic checks only; no Temporal/Process-specific enforcement yet.

- `primitive verify --kind process --name <name>`:
  - Runs generic repo checks:
    - Naming (`checkNamingConventions`),
    - Security/SAST/secret scan,
    - Dependency vulnerabilities (for Go modules),
    - Tests (`go test` under the primitive),
    - Optional Buf / GitOps checks when configured.
  - Does **not** currently validate:
    - Presence of `cmd/worker` or `cmd/ingress` binaries.
    - Existence of `internal/workflows/**` or `internal/ingress/router.go`.
    - Idempotency helpers or Process fact emission patterns.
    - Temporal namespace/task queue configuration in code or GitOps.

### 6.3 Known gaps and next steps

- Scaffold:
  - Introduce a Process-specific scaffold path that:
    - Generates `cmd/worker/main.go` and `cmd/ingress/main.go` with Temporal bootstrap and ingress routing stubs.
    - Creates `internal/workflows`, `internal/ingress`, and `internal/process/state.go` as described in §2–§3.
    - Uses dedicated Process templates for README and code comments (similar to Domain’s template approach).
- Verify:
  - Add Process-specific checks to `primitive verify` to ensure:
    - Worker/ingress entrypoints and workflow/router files are present.
    - Temporal configuration is wired in GitOps (values.yaml).
    - Optional: enforce idempotency patterns and fact emission conventions via heuristics.
