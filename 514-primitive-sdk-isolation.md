# 514 – Primitive SDK & Isolation Principles

**Status:** Draft  
**Audience:** Platform DX, CLI implementers, primitive authors, AI agents  
**Scope:** Canonical rules for how primitives (Domain, Process, Agent, UISurface) depend on SDKs, import other code, and interact with the Ameide CLI. This backlog constrains and aligns 477, 393, 404, 403, 395 and the scaffolding backlogs 510–513.

---

## 1. Grounding

- **Primitive stack & layout:** `477-primitive-stack.md` (what lives under `primitives/*` vs operators vs GitOps).  
- **SDK import policy:** `393-ameide-sdk-import-policy.md`, `404-go-buf-first-migration.md`, `403-ts-proto-first-migrations.md`, `403-python-buf-first-migration-plan.md`.  
- **Front-end SDK usage:** `services/www_ameide_platform/backlog/SDK_USAGE.md`.  
- **Scaffolding backlogs:** `510-domain-primitive-scaffolding.md`, `511-process-primitive-scaffolding.md`, `512-agent-primitive-scaffolding.md`, `513-uisurface-primitive-scaffolding.md`.  
- **Agent architecture:** `400-agentic-development.md`, `504-agent-vertical-slice.md`, `505-agent-developer-v2*.md`.  

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
    - Required: `--kind domain --name <name> --proto-path <service.proto>`.
    - Implied: `--lang go`.
    - Optional: `--include-gitops`, `--include-test-harness`.
  - Process:
    - Required: `--kind process --name <name> --proto-path <process_api.proto>`.
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
  - Require `--proto-path` only for Domain/Process kinds.
  - Accept `--include-gitops` and `--include-test-harness` uniformly.
  - For Agent/UISurface:
    - Ignore or deprecate `--proto-path` if passed.
    - Avoid agent-specific knobs that belong to Backstage/Transformation (owner, description, definitionRef, model) on the hot path.

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

