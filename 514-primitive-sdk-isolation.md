# 514 – Primitive SDK & Isolation Principles

**Status:** Draft  
**Audience:** Platform DX, CLI implementers, primitive authors, AI agents  
**Scope:** Canonical rules for how primitives (Domain, Process, Agent, UISurface) depend on SDKs, import other code, and interact with the Ameide CLI. This backlog constrains and aligns 477, 393, 404, 403, 395 and the scaffolding backlogs 510–513.

---

## Primitive/operator alignment

- **Operator side (495, 497, 498–501):** Operators are the **control plane** for primitives: they reconcile `Domain`, `Process`, `Agent`, and `UISurface` CRDs into Deployments, Jobs, Routes, secrets, HPAs, and monitoring resources, exposing readiness via conditions. Operators never encode business logic, workflows, prompts, or UI layout; they act as compilers from CRDs to cluster/external infra.  
- **Primitive side (510–513):** Scaffolds are the **behavior plane**: they produce SDK-based runtime services (gRPC, Temporal workers, agents, UI servers) that implement aggregates, workflows, prompts, and routes inside the container images referenced by the CRDs. They know nothing about controller-runtime or operator internals.  
- **Role of 514:** This backlog defines the **shared isolation rules** (SDK-only imports, self-contained modules, CLI-out-of-band, opinionated scaffolds) so that primitive scaffolds (510–513) and operators (498–501) work in harmony:
  - Operators own infra and health.
  - Primitives own SDK-based runtime behavior.
  - The CLI/scaffolder stitches them together without collapsing these boundaries.
  - Codegen plugin evolution is tracked in `backlog/521-code-generation-improvements.md`; CLI/orchestration evolution is tracked in `backlog/522-cli-orchestration-improvements.md`.

---

## 1. Grounding

- **Primitive stack & layout:** `477-primitive-stack.md` (what lives under `primitives/*` vs operators vs GitOps).  
- **SDK import policy:** `393-ameide-sdk-import-policy.md`, `404-go-buf-first-migration.md`, `403-ts-proto-first-migrations.md`, `403-python-buf-first-migration-plan.md`.  
- **Front-end SDK usage:** `services/www_ameide_platform/backlog/SDK_USAGE.md`.  
- **Scaffolding backlogs:** `510-domain-primitive-scaffolding.md`, `511-process-primitive-scaffolding.md`, `512-agent-primitive-scaffolding.md`, `513-uisurface-primitive-scaffolding.md`.  
- **Agent architecture:** `400-agentic-development.md`, `504-agent-vertical-slice.md`, `505-agent-developer-v2*.md`.  
- **Capability implementation playbook:** `backlog/533-capability-implementation-playbook.md` (end-to-end activity DAG; 514 is one of the guardrails inputs).

This backlog is **normative** for new primitives and for CLI/scaffold behavior. Where older docs conflict, 514 takes precedence.

---

## 2. Principles

### P1 – SDK-only cross-primitive communication

- Runtime primitives **never** import Buf/proto packages directly for outbound calls:
  - No `@ameide/core-proto` imports in TS/Node runtime code.
  - No `packages/ameide_core_proto` imports in Go/Python runtime logic (beyond server/handler stubs generated from the primitive’s own proto).
- Cross-primitive communication is **always** via the Ameide SDKs:
  - Go: `github.com/ameideio/ameide-sdk-go/...`.
  - TypeScript: `@ameideio/ameide-sdk-ts` (and service-specific barrels like `@/lib/sdk/core`).
  - Python: `ameide_sdk_python` packages.
- **Codegen, not bespoke JSON:** SDKs for Go/TS/Python are the **only** supported way to consume primitives (Transformation domains, Process workers, agents, UISurfaces/portal). There must be no bespoke JSON contracts for cross-primitive traffic at runtime; when JSON is needed, it must be defined in proto and consumed via the generated SDK clients.
- Tests may import proto types directly where necessary, but production handler/agent/UISurface code relies on SDK clients.

### P2 – Self-contained primitives

- Each primitive lives under `primitives/{domain,process,agent,uisurface}/{name}` and is designed as a **self-contained service**:
  - One Go module / Python project / Node package per primitive.
  - No imports from other primitives’ code or unrelated workspace packages; the **only** cross-component dependencies are:
    - Ameide SDKs (TS/Go/Python).
    - A minimal, explicitly-whitelisted infra set (logging/telemetry/common HTTP helpers) to be listed in 395/SDK policy docs.
- Operators and shared infra packages depend on primitives, **not** the other way around.

### P3 – CLI is out-of-band, not a runtime dependency

- Runtime primitives **do not**:
  - Invoke `ameide` CLI commands as part of request handling.
  - Depend on devcontainer tooling or `develop_in_container` to serve traffic.
- The Ameide CLI (`ameide primitive describe/drift/plan/impact/scaffold/verify/prompt`) is:
  - A **guardrail and scaffolding tool** for humans and AmeideCoder.
  - Used in devcontainers, CI, or transformation flows — **not** inside production request paths of primitives.
- The **AmeideCoder** devcontainer/A2A agents are the only place where the CLI guardrail loop is orchestrated programmatically (per 504/505). Generic Agent primitives use SDKs and external APIs but stay CLI-agnostic.

### P4 – Opinionated, minimal scaffolders

- For each primitive kind we support **one canonical language/shape** and **minimal flags**:
  - Domain:
    - Required: `--kind domain --name <name>`.
    - Implied: `--lang go`.
    - Optional: `--include-gitops`, `--include-test-harness`.
  - Process:
    - Required: `--kind process --name <name>`.
    - Implied: `--lang go`.
    - Optional: `--include-gitops`, `--include-test-harness`.
  - Agent:
    - Required: `--kind agent --name <name>`.
    - Implied: `--lang python`.
    - No `--proto-path` requirement; Agent behavior is driven by AgentDefinition + SDKs.
  - UISurface:
    - Required: `--kind uisurface --name <name>`.
    - Implied: `--lang ts`.
    - No `--proto-path` requirement; UISurface always calls backend via SDKs.
- Scaffolders refuse non-canonical language selections for each kind and avoid “kitchen sink” flags; richer metadata (owner, description, definitionRef, model config) comes from Backstage/Transformation, not the basic scaffold command.

---

## 3. Alignment for 510–513

This section describes how the existing primitive-scaffolding backlogs must be interpreted under 514.

### 3.1 Domain (510)

- 510’s outbox/dispatcher pattern remains the canonical shape:
  - `internal/ports/outbox.go`, `internal/adapters/postgres/outbox.go`, dispatcher + dispatcher `cmd` entrypoint stay required.
- Additional constraints:
  - Any **outbound** calls from Domain to other primitives must be made via SDK clients, not imported proto types.
  - Domain handler packages (`internal/handlers/**`) must not import:
    - Other primitives’ modules.
    - Proto packages other than their own service stubs.
  - `ameide primitive verify` is expected (future work) to:
    - Confirm presence of outbox/dispatcher files (510 rules).
    - Scan for forbidden proto imports and cross-primitive imports.

### 3.2 Process (511)

- 511’s Temporal worker + ingress router shape remains:
  - `cmd/worker`, `cmd/ingress`, `internal/workflows`, `internal/ingress`, `internal/tests` with integration harness.
- Additional constraints:
  - Process workflows and ingress routers call Domains via SDKs; they do not construct gRPC channels or import raw proto client stubs themselves.
  - Any direct proto usage in Process runtime code is limited to its **own** service definition and IDL-level helpers.

### 3.3 Agent (512)

- 512 continues to define the scaffold shape:
  - Python + FastAPI + LangGraph runtime in `src/agent.py` and `src/main.py`.
  - Prompts under `src/prompts/default.md` and tool registration under `src/tools/__init__.py`.
  - RED `tests/test_agent.py` at scaffold time.
- Under 514:
  - Generic Agent primitives:
    - Do **not** embed Ameide CLI workflows into their runtime logic.
    - Use SDK clients and external APIs to implement AgentDefinition behavior.
    - May still reference CLI in documentation/checklists, but production code remains CLI-agnostic.
  - The “Coder agent” / devcontainer pattern (504/505) is **not** used as the default Agent scaffold:
    - AmeideCoder’s devcontainer loop and `develop_in_container` tool stay documented in 504/505 and in the concrete `primitives/agent/ameide-coder` implementation.
    - 512’s templates should be updated so that prompts and README focus on SDK usage and AgentDefinition wiring, not on `ameide primitive` commands.

### 3.4 UISurface (513)

- 513 already encodes the SDK-first policy:
  - `src/proto/index.ts` wraps SDK clients, and runtime code avoids `@ameide/core-proto` imports.
- Under 514:
  - That policy becomes a **general primitive rule** (P1/P2); UISurface is the reference example.
  - UISurface scaffolds should not acquire additional flags beyond `--kind`, `--name`, `--include-gitops`, `--include-test-harness`.

---

## 4. CLI & verification expectations

### 4.1 Scaffold command behavior

- `ameide primitive scaffold` must:
  - Enforce canonical language per kind (P4).
  - Accept `--include-gitops` and `--include-test-harness` uniformly.
  - Avoid introducing proto-path dependencies into primitive scaffolds:
    - Domain/Process scaffolds are SDK- and shape-only; proto analysis lives in plan/impact/verify, not scaffold.
    - Agent/UISurface scaffolds remain SDK/HTTP-focused and do not accept proto-path as a required flag.

### 4.2 Prompt command and Coder agents

- `ameide primitive prompt` and the shared agent prompt profiles under `prompts/agent/*.md` remain the **onboarding mechanism for Coder-style agents** and human developers.
- Generic primitives do **not** call `primitive prompt` at runtime:
  - This behavior is confined to AmeideCoder and other A2A dev agents, described in `504-agent-vertical-slice.md` and `505-agent-developer-v2*.md`.
  - 512’s scaffolded agent prompts should be updated to reference SDK usage and domain-specific workflows, not direct CLI guardrail execution.

### 4.3 Verify command and import checks

- `ameide primitive verify --kind <kind> --name <name> --mode repo` is expected (over time) to enforce:
  - P1: Fail if runtime code in `primitives/{kind}/{name}` imports forbidden proto modules or cross-primitive packages.
  - P2: Warn/fail if primitives depend on non-whitelisted workspace packages.
  - 510/511/512/513-specific shape checks remain in place (EDA outbox/dispatcher, Temporal wiring, Agent skeleton, UISurface SDK wiring).
- Enforcement may reuse existing SDK policy scripts and lint rules from `393-ameide-sdk-import-policy.md`, extended to cover the `primitives/*` tree.

---

## 5. Migration notes

Short-term expectations:

- Existing primitives that still import proto modules directly or embed CLI behavior in runtime code are **grandfathered**, but:
  - New primitives must follow 514 from day one.
  - Refactors should treat 514 as the north star and gradually align older primitives.
- CLI and scaffolding backlogs (510–513) must be updated to:
  - Reference this document for SDK/import rules, self-containment, and scaffold flags.
  - Remove or narrow any text that suggests primitives themselves orchestrate `ameide` CLI guardrails at runtime.

Longer term:

- `primitive verify` becomes the primary enforcement point for 514.
- SDK import policy scripts from 393 expand to cover primitives and fail CI on violations.
- AmeideCoder and related A2A agents continue to refine the devcontainer + CLI orchestration loop, but that concern stays separate from primitive scaffolding backlogs.

---

## 6. Implementation progress (CLI, verify & scaffolds)

This section describes how much of 514 is currently enforced by the CLI (`packages/ameide_core_cli`) and scaffolds. It is descriptive; the rest of 514 remains normative.

### 6.1 P1 – SDK-only cross-primitive communication

**Status:** Partially implemented, strongest for Go/Domain primitives but now enforced across Go/TS/Python at import level.

- **Go (Domain/Process)**
  - The scaffolder derives Go import paths for service types from the proto path via `deriveImportPath`, mapping `packages/ameide_core_proto/src/...` to `github.com/ameideio/ameide-sdk-go/gen/...`.
  - Domain/Process handlers generated by `buildHandlersContent`:
    - Import the SDK package and alias it (e.g., `scrumv1`).
    - Use SDK-generated request/response types for RPC signatures.
    - Do **not** import `packages/ameide_core_proto` directly.
  - `primitive verify` includes an `Imports` check for Go primitives:
    - Fails when runtime (non-test) `.go` files under `primitives/*/<name>` contain `packages/ameide_core_proto` or `buf.build/gen/go`, enforcing “no direct proto imports” in runtime Go.

- **TypeScript & Python**
  - UISurface and Agent scaffolds currently:
    - Do not import `@ameide/core-proto` or `packages/ameide_core_proto` in runtime code by default.
    - Do **not** yet wire SDK clients aggressively (TS SDK usage is not scaffolded; Python SDK usage is still a TODO).
  - `primitive verify` now reuses the shared `Imports` check for TS/Python primitives:
    - Walks `.ts/.tsx/.cts/.mts` and `.py` files under a primitive.
    - Fails when runtime (non-test) files import `@ameide/core-proto`, `packages/ameide_core_proto`, or `buf.build/gen/*`.
    - Best-effort flags cross-primitive imports such as `primitives/{domain,process,agent,uisurface}/…` (TS) and `primitives.{domain,process,agent,uisurface}.…` (Python), nudging authors toward SDK-only usage.

### 6.2 P2 – Self-contained primitives

**Status:** Partially implemented for Go; other languages still rely on convention.

- **Module layout**
  - Scaffolders create one module/project per primitive where scaffolding exists:
    - Go: `go.mod` per Domain/Process primitive.
    - Python: `pyproject.toml` per Agent primitive.
    - TS: `package.json` per UISurface primitive created by earlier generic scaffolds (new UISurface scaffolds are currently disabled; see 513).
  - `go.work` is updated (best-effort) to include new Go primitives.

- **Cross-primitive imports**
  - Go `Imports` check in `primitive verify`:
    - Walks non-test `.go` files under `primitives/{domain,process,agent,uisurface}/{name}`.
    - Fails if runtime code imports proto packages (`packages/ameide_core_proto`, `buf.build/gen/go`), consistent with P1.
    - Additionally:
      - Parses `go.mod` to discover the primitive’s own module path.
      - Flags imports of `github.com/ameideio/ameide/primitives/...` that do not match the primitive’s own module as **cross-primitive imports**.
  - There is not yet a general “non-whitelisted workspace packages” list enforced across all primitives; enforcement focuses on proto and cross-primitive imports for Go.

- **Other languages**
  - Process/Agent/UISurface runtime code is now covered by the shared `Imports` check for:
    - Direct proto/core-proto imports in TypeScript/Python.
    - Obvious cross-primitive imports via `primitives/*` (TS) and `primitives.*` (Python).
  - There is not yet a central allowlist for shared infrastructure packages; enforcement remains focused on obvious proto/core-proto and cross-primitive imports, with Go having the strongest guarantees via module-path awareness.

### 6.3 P3 – CLI is out-of-band, not a runtime dependency

**Status:** Mostly enforced by convention and scaffolded structure; not yet checked automatically in `primitive verify`.**

- Scaffolds for Domain/Process/Agent/UISurface:
  - Do not generate runtime code that shells out to `ameide` or depends on devcontainer tooling for request handling.
  - Reference the CLI only in README/checklists (how to run `primitive verify`, `primitive plan`, etc.).
- The Coder/devcontainer pattern:
  - AmeideCoder and related A2A agents are implemented separately (under `primitives/agent` and `services/devcontainer_service`).
  - They orchestrate CLI guardrails as described in 504/505; this behavior is not baked into generic primitive scaffolds.

### 6.4 P4 – Opinionated, minimal scaffolders

**Status:** Implemented in CLI flag handling; scaffolders are SDK/shape-based and no longer require proto-paths.

- `primitive scaffold`:
  - Enforces canonical language per kind:
    - Domain/Process: defaults to Go; `--lang` values other than `go` are rejected.
    - Agent: uses the dedicated Python scaffold path; `--lang` values other than `py`/`python` are rejected.
    - UISurface: recognized as a kind, but scaffolding is currently **disabled**; `runScaffold` returns an explicit “UISurface scaffolding not yet implemented; see 513” error instead of generating the old generic TS shape.
  - Does **not** require `--proto-path` for any primitive kind:
    - Domain/Process scaffolds are SDK-only shapes (go.mod, Dockerfile, basic handlers/worker/ingress) with no proto dependency.
    - Agent scaffolds have never required proto-path and remain SDK/HTTP-only; any proto usage is via SDK clients.
  - Avoids per-kind “extra knobs” on the hot path:
    - Richer metadata and Backstage wiring live in templates and Backstage configs, not extra CLI flags.

### 6.5 Known gaps and next steps

- Extend `primitive verify` to enforce P1/P2 across:
  - TypeScript (UISurface) and Python (Agent) runtime code via import scanning.
  - Non-whitelisted workspace imports (shared infra allowlist).
- Grow the per-kind scaffolds (511–513) so they fully implement the shapes described in their backlogs while keeping SDK-only and CLI-out-of-band guarantees.
