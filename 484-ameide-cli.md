# Ameide CLI – Proto-Aligned CLI for AI Agents

## 1. Purpose

The **Ameide CLI** is a proto-aligned command-line interface that mirrors the SDK structure. Like `ameide-sdk-go`, `ameide-sdk-ts`, and `ameide-sdk-python`, the CLI provides access to Ameide capabilities – but from the command line.

It is designed to be consumed by:
- **AI Agents** (primary) – automating development work via structured JSON output
- **Developers** (secondary) – running commands interactively when needed

The CLI serves as **guardrails for AI agents** doing autonomous development. Agents use `plan`, `scaffold`, and `verify` commands to:
1. Understand what work needs to be done (plan)
2. Generate compliant code/tests/gitops (scaffold)
3. Check correctness before committing (verify)

Without the CLI, agents would generate non-compliant code. The CLI enforces repo conventions, test infrastructure patterns (430), GitOps structure (434), and proto alignment.

---

## 2. Core Principles

### 2.1 Proto-Aligned, Like SDKs

The CLI follows the same proto package structure as the SDKs:

```
ameide <proto-package> <action> [flags]
```

Examples:
```bash
ameide workflows list              # → ameide_core_proto.workflows_runtime.v1
ameide graph query                 # → ameide_core_proto.graph.v1
ameide primitive plan              # → ameide_core_proto.primitive.v1
ameide primitive scaffold          # → ameide_core_proto.primitive.v1
ameide primitive verify            # → ameide_core_proto.primitive.v1
```

Each command maps to a proto service/message. The CLI is generated from (or aligned with) the same protos that generate the SDKs.

### 2.2 Dual Output: Agent-First

Every command emits:

1. **Structured JSON** (default with `--json` flag)
   - Matches proto message shapes
   - Primary output for AI agent consumption
   - Deterministic, parseable, machine-readable

2. **Human-readable** (default stdout without `--json`)
   - Formatted text, tables, summaries
   - Secondary output for developer terminal use

AI agents should always use `--json`. The human-readable output exists for debugging and manual inspection.

### 2.3 Local-First, Service-Optional

The CLI can operate in two modes:

1. **Local mode** – works on filesystem, runs tests, parses protos directly
2. **Service mode** – connects to Ameide services via gRPC/Connect

Some commands (like `primitive`) are inherently local. Others (like `workflows`) require service connectivity.

---

## 3. Command Structure

### 3.1 Service Commands (gRPC clients)

These connect to Ameide services:

```bash
# Workflows (ameide.workflows.v1)
ameide workflows list [--status pending|running|completed]
ameide workflows runs [--definition-id <id>]
ameide workflows start <definition-id>

# Graph (ameide.graph.v1)
ameide graph query <query>
ameide graph mutate <mutation>

# Events (ameide.events.v1)
ameide events list [--aggregate-id <id>]
ameide events subscribe [--types <types>]
```

### 3.2 Primitive Commands (local, for agentic development)

These operate on the local filesystem for AI-assisted development:

```bash
# Plan - introspect repo + proto → testable work plan
ameide primitive plan \
  --kind domain \
  --name orders \
  --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto

# Scaffold - generate code/tests from proto
ameide primitive scaffold \
  --kind domain \
  --name orders \
  --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto \
  --lang go

# Verify - run tests/checks, report structured feedback
ameide primitive verify --kind domain --name orders

# Verify ALL consumers of a modified proto (cascade verification)
ameide primitive verify --all --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto

# Impact - list all consumers of a proto before modifying
ameide primitive impact --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto
```

### 3.3 Config Commands (CLI management)

```bash
ameide config init       # Create ~/.ameide/config.yaml
ameide config show       # Display current configuration
ameide config set <k> <v> # Update config value
```

---

## 4. Primitive Commands for Agentic Work

The `ameide primitive` subcommands are designed specifically to support AI agents doing development work on primitives (Domain, Process, Agent, UISurface).

### 4.0 What Gets Scaffolded

Each primitive kind has a different scaffold output:

#### Domain Primitive
```
primitives/domain/{name}/
├── cmd/main.go                      # Entrypoint
├── internal/
│   ├── handlers/                    # RPC handlers (one per proto RPC)
│   │   ├── create_{entity}.go
│   │   └── get_{entity}.go
│   ├── repository/                  # Data access interfaces
│   │   └── {entity}_repository.go
│   └── tests/                       # Test skeletons (failing by default)
│       ├── create_{entity}_test.go
│       └── get_{entity}_test.go
├── Dockerfile
├── go.mod
└── README.md

gitops/primitives/domain/{name}/
├── values.yaml                      # Helm values
└── kustomization.yaml
```

**Domain characteristics:**
- Owns entities and business logic
- Exposes gRPC/Connect APIs (from proto)
- Has repository interfaces (no direct DB in handlers)
- Emits domain events

#### Process Primitive
```
primitives/process/{name}/
├── cmd/main.go
├── internal/
│   ├── workflow/                    # Temporal workflow definitions
│   │   └── {name}_workflow.go
│   ├── activities/                  # Temporal activities
│   │   ├── call_{domain}_activity.go
│   │   └── ...
│   └── tests/
│       └── {name}_workflow_test.go
├── Dockerfile
├── go.mod
└── README.md

gitops/primitives/process/{name}/
├── values.yaml
└── kustomization.yaml
```

**Process characteristics:**
- Orchestrates cross-domain workflows
- Uses Temporal for durability
- Calls Domain primitives via SDK
- No direct data ownership

#### Agent Primitive
```
primitives/agent/{name}/
├── cmd/main.go
├── internal/
│   ├── agent/                       # Agent definition
│   │   └── {name}_agent.go
│   ├── tools/                       # Tool implementations
│   │   ├── {tool1}_tool.go
│   │   └── {tool2}_tool.go
│   ├── prompts/                     # System prompts
│   │   └── {name}.md
│   └── tests/
│       └── {name}_agent_test.go
├── Dockerfile
├── go.mod
└── README.md

gitops/primitives/agent/{name}/
├── values.yaml
└── kustomization.yaml
```

**Agent characteristics:**
- LLM-powered reasoning
- Tools call Domain/Process primitives
- Structured prompts
- Human-in-the-loop support

#### UISurface Primitive
```
primitives/uisurface/{name}/
├── src/
│   ├── app/                         # Next.js app router
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/                  # React components
│   └── lib/                         # SDK client wrappers
│       └── api.ts
├── tests/
│   └── e2e/
│       └── {name}.spec.ts
├── Dockerfile
├── package.json
├── next.config.ts
└── README.md

gitops/primitives/uisurface/{name}/
├── values.yaml
└── kustomization.yaml
```

**UISurface characteristics:**
- Next.js/React frontend
- Calls backends via SDK (TypeScript)
- E2E tests with Playwright
- Server components for data fetching

---

### 4.1 `ameide primitive plan`

> *"What work needs to be done?"*

Introspects the repo + proto to produce a testable plan.

**Output (JSON):**
```json
{
  "summary": "New Orders domain primitive from proto ameide.orders.v1",
  "suggested_scaffolds": [
    {
      "kind": "DOMAIN",
      "name": "orders",
      "paths": {
        "service": "services/orders",
        "tests": "services/orders/internal/tests",
        "gitops": "gitops/sources/values/dev/apps/orders.yaml"
      }
    }
  ],
  "tests_to_create": [
    {
      "id": "orders.api.create_order.success",
      "description": "Creating a valid order returns 201 and persists order row.",
      "level": "integration"
    }
  ]
}
```

### 4.2 `ameide primitive scaffold`

> *"Generate the skeleton code and tests."*

Creates service stubs, empty test files, and GitOps manifests from proto.

**Backstage alignment:** `scaffold` uses the **same templates** as Backstage's Software Templates for Domain/Process/Agent/UISurface. The CLI runs these templates locally (for AI agents in devcontainer), while Backstage exposes them via UI (for human developers who prefer a portal). One template stack, two consumption paths.

**Output (JSON):**
```json
{
  "summary": "Scaffolded Go domain primitive 'orders'.",
  "files_created": [
    "services/orders/main.go",
    "services/orders/internal/handlers/create_order.go",
    "services/orders/internal/tests/create_order_test.go",
    "gitops/sources/values/dev/apps/orders.yaml"
  ],
  "tests_created": ["orders.api.create_order.success"],
  "next_steps": [
    "Implement handlers in services/orders/internal/handlers/",
    "Run `ameide primitive verify --kind domain --name orders`."
  ]
}
```

### 4.3 `ameide primitive verify`

> *"Is this safe to commit?"*

Runs tests, linters, and checks. Reports structured pass/fail.

**Output (JSON):**
```json
{
  "summary": "fail",
  "tests": [
    {"id": "orders.api.create_order.success", "status": "PASS"},
    {"id": "orders.api.create_order.validation", "status": "FAIL", "details": "Expected INVALID_ARGUMENT, got OK", "file": "services/orders/internal/tests/create_order_test.go", "line": 42}
  ],
  "lint": [
    {"tool": "buf-lint", "status": "PASS"},
    {"tool": "golangci-lint", "status": "WARN", "issues": 3}
  ],
  "next_steps": ["Fix validation in CreateOrder handler."]
}
```

---

## 5. Agent Sequence

The full agentic development sequence, assuming **no human developers** - only AI agents and this CLI:

### Step 0: Get Context (Transformation Domain)

The **Transformation Domain** (see [471-transformation-domain](471-transformation-domain.md)) is the source of truth for what needs to be built. Before any coding begins, the Transformation Agent:

1. **Reads design artifacts:**
   - `ProcessDefinition` entities (what processes exist, their states, transitions)
   - `AgentDefinition` entities (what agents exist, their tools, prompts)
   - Backlog items, epics, requirements

2. **Queries runtime context:**
   - Graph projections (current state of the system)
   - Runtime metrics (performance, errors, usage patterns)
   - Deployment state (what's deployed where)

3. **Produces a decision:**
   - "We need a new Orders Domain primitive" or
   - "We need to modify the L2O Process to handle cancellations" or
   - "We need a new UISurface for the admin dashboard"

This decision becomes the input to `ameide primitive plan`. The Transformation domain owns the **"what"** and **"why"**; the CLI enforces the **"how"**.

**Future integration:** Plan/verify results can be stored back as Transformation artifacts (e.g., `ImplementationPlan`, `VerifyReport`) for traceability and audit.

### Step 1: AI Codes the Proto

Coder Agent:
- Opens the Buf workspace (`packages/ameide_core_proto`)
- Creates or edits proto files (e.g., `ameide/orders/v1/orders.proto`)
- Calls `ameide primitive plan` to understand implications:

```bash
ameide primitive plan --kind domain --name orders \
  --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto --json
```

Response tells agent what to do next:
- Generate SDKs
- Scaffold primitive
- What tests will be created

### Step 2: Generate SDKs

Agent runs Buf to regenerate SDKs:

```bash
buf generate
```

Updates `ameide_sdk_go`, `ameide_sdk_python`, `ameide_sdk_ts` packages.

### Step 3: Scaffold Primitive + TDD

Agent scaffolds the primitive structure:

```bash
ameide primitive scaffold --kind domain --name orders \
  --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto \
  --lang go --json
```

This creates:
- Service skeleton (`services/orders/`)
- Repository interfaces
- Basic handlers wired to SDK
- **Failing test skeletons** (TDD - red by default)
- GitOps manifests (`gitops/sources/values/dev/apps/orders.yaml`)

The response contains the **test plan** - list of test IDs + descriptions that become the agent's work plan.

### Step 4: Implement via TDD Loop

Agent iterates:

```
1. Pick next failing test from plan
2. Implement minimal code to make it pass
3. Run: ameide primitive verify --kind domain --name orders --json
4. Check structured result:
   - If test passes → next test
   - If test fails → read error details, fix, retry
5. Repeat until all tests green
```

The verify output is machine-readable:
```json
{
  "summary": "fail",
  "tests": [
    {"id": "orders.create_order.success", "status": "PASS"},
    {"id": "orders.create_order.validation", "status": "FAIL", "details": "...", "file": "...", "line": 42}
  ]
}
```

### Step 5: Guardrail Checks

`ameide primitive verify` enforces more than just tests:

```bash
ameide primitive verify --kind domain --name orders --json
```

**Checks:**
- All tests pass
- Buf breaking rules (no breaking proto changes)
- Buf lint (proto style)
- Language linters (golangci-lint, eslint, ruff)
- Proto/SDK consistency (generated code up to date)
- GitOps manifests valid
- Basic project conventions
- Secret scan (no credentials in code/tests/manifests) - aligns with [476-security-and-trust](476-security-and-trust.md)

The agent must get **all checks green** before proceeding.

### Step 6: Git Commit & PR

Agent:
- Formats code
- Creates commit with verify summary
- Opens PR

GitOps + Argo CD takes it from there.

---

### Summary Flow (New Primitive)

```
0. Transformation decides what to build
1. Agent edits proto
2. Agent runs buf generate (SDKs)
3. Agent runs ameide primitive scaffold (code + tests + gitops)
4. Agent implements until ameide primitive verify is green
5. Agent commits & PRs
```

The CLI does **no fuzzy reasoning** - it just:
- Uses Buf descriptors
- Applies fixed templates
- Generates tests & checks

All non-determinism (how to implement, how to fix failures) lives in agents.

---

## 5b. Modifying Existing Protos (Cascading Effects)

When an AI agent modifies an **existing** proto (not creating new), the workflow must detect and verify all consuming services.

### Step 1: Impact Analysis

Before modifying a proto, agent runs:

```bash
ameide primitive impact --proto-path packages/ameide_core_proto/src/ameide_core_proto/orders/v1/orders.proto --json
```

**Output (JSON):**
```json
{
  "proto_path": "ameide_core_proto/orders/v1/orders.proto",
  "consumers": [
    {
      "kind": "DOMAIN",
      "name": "orders",
      "path": "primitives/domain/orders",
      "imports": ["OrdersService", "Order", "CreateOrderRequest"]
    },
    {
      "kind": "PROCESS",
      "name": "l2o",
      "path": "primitives/process/l2o",
      "imports": ["OrdersServiceClient"]
    },
    {
      "kind": "UISURFACE",
      "name": "www_ameide_platform",
      "path": "primitives/uisurface/www_ameide_platform",
      "imports": ["Order", "OrderStatus"]
    }
  ],
  "sdks_affected": ["ameide-sdk-go", "ameide-sdk-ts", "ameide-sdk-python"]
}
```

### Step 2: Breaking Change Detection

Agent runs Buf breaking check:

```bash
buf breaking packages/ameide_core_proto --against .git#branch=main
```

If breaking changes detected, `ameide primitive verify` will fail unless:
- The change is intentional AND
- All consumers are updated in the same PR

### Step 3: Regenerate All SDKs

After proto modification:

```bash
buf generate                          # Regenerates all SDKs
ameide primitive verify --all --json  # Verify ALL affected primitives
```

The `--all` flag runs verify on every consumer identified by impact analysis.

### Step 4: Cascade Verification

`ameide primitive verify --all` performs:

| Check | Description |
|-------|-------------|
| SDK compilation | All SDKs compile with proto changes |
| Consumer tests | Run tests for ALL consuming primitives |
| Integration tests | Cross-primitive integration tests |
| Type compatibility | No type errors in TS/Go/Python consumers |

**Output (JSON):**
```json
{
  "summary": "fail",
  "proto_change": {
    "path": "ameide_core_proto/orders/v1/orders.proto",
    "breaking": false,
    "changes": ["added field Order.priority", "added rpc CancelOrder"]
  },
  "sdk_results": [
    {"sdk": "ameide-sdk-go", "status": "PASS"},
    {"sdk": "ameide-sdk-ts", "status": "PASS"},
    {"sdk": "ameide-sdk-python", "status": "PASS"}
  ],
  "consumer_results": [
    {"name": "orders", "kind": "DOMAIN", "status": "PASS", "tests": 12},
    {"name": "l2o", "kind": "PROCESS", "status": "FAIL", "tests": 8, "failed": 2, "details": "CancelOrder not handled in workflow"},
    {"name": "www_ameide_platform", "kind": "UISURFACE", "status": "PASS", "tests": 5}
  ],
  "next_steps": ["Fix l2o process to handle CancelOrder"]
}
```

### Proto Modification Summary Flow

```
1. Agent identifies proto to modify
2. ameide primitive impact → list consumers
3. Agent modifies proto
4. buf generate → regenerate SDKs
5. ameide primitive verify --all → test ALL consumers
6. Agent fixes any failures in consumers
7. Repeat until all consumers green
8. Agent commits (proto + SDK + all consumer fixes in one PR)
```

### Proto Definitions (Extended)

Add to `ameide_core_proto.primitive.v1`:

```proto
message ImpactRequest {
  string proto_path = 1;
}

message ImpactResult {
  string proto_path = 1;
  repeated Consumer consumers = 2;
  repeated string sdks_affected = 3;
}

message Consumer {
  PrimitiveKind kind = 1;
  string name = 2;
  string path = 3;
  repeated string imports = 4;  // Which types/services are imported
}

message VerifyAllRequest {
  string proto_path = 1;  // Optional: only verify consumers of this proto
}

message VerifyAllResult {
  string summary = 1;
  ProtoChange proto_change = 2;
  repeated SdkResult sdk_results = 3;
  repeated ConsumerResult consumer_results = 4;
  repeated string next_steps = 5;
}

message ProtoChange {
  string path = 1;
  bool breaking = 2;
  repeated string changes = 3;
}

message SdkResult {
  string sdk = 1;
  string status = 2;
}

message ConsumerResult {
  PrimitiveKind kind = 1;
  string name = 2;
  string status = 3;
  int32 tests = 4;
  int32 failed = 5;
  string details = 6;
}
```

### Existing Tooling Alignment

This integrates with:
- **Buf breaking** (`buf.yaml` has `breaking: use: [FILE]`) - detects breaking changes
- **365-buf-sdks-v2** - SDK regeneration and publishing pipeline
- **430 test infrastructure** - consumer tests run in mock/cluster modes

---

## 6. Proto Definitions

The CLI commands map to proto messages in `ameide_core_proto.primitive.v1`.

This follows the existing proto package convention:
- `ameide_core_proto.graph.v1` → GraphService
- `ameide_core_proto.workflows_runtime.v1` → WorkflowService
- `ameide_core_proto.primitive.v1` → PrimitiveService (new)

```proto
// packages/ameide_core_proto/src/ameide_core_proto/primitive/v1/primitive.proto

syntax = "proto3";

package ameide_core_proto.primitive.v1;

option go_package = "ameide_core_proto/primitive/v1;primitivev1";

enum PrimitiveKind {
  PRIMITIVE_KIND_UNSPECIFIED = 0;
  PRIMITIVE_KIND_DOMAIN = 1;
  PRIMITIVE_KIND_PROCESS = 2;
  PRIMITIVE_KIND_AGENT = 3;
  PRIMITIVE_KIND_UISURFACE = 4;
}

message PlanRequest {
  PrimitiveKind kind = 1;
  string name = 2;
  string proto_path = 3;
  string repo_root = 4;  // Optional: repo root path (default: cwd)
}

message PlanResult {
  string summary = 1;
  repeated SuggestedScaffold suggested_scaffolds = 2;
  repeated TestToCreate tests_to_create = 3;
}

message SuggestedScaffold {
  PrimitiveKind kind = 1;
  string name = 2;
  map<string, string> paths = 3;  // service, tests, migrations, gitops
}

message TestToCreate {
  string id = 1;
  string description = 2;
  string level = 3;  // unit, integration, e2e
}

message ScaffoldRequest {
  PrimitiveKind kind = 1;
  string name = 2;
  string proto_path = 3;
  string lang = 4;  // go, typescript, python
  string repo_root = 5;  // Optional: repo root path (default: cwd)
  bool dry_run = 6;  // If true, return what would be created without writing files
}

message ScaffoldResult {
  string summary = 1;
  repeated string files_created = 2;
  repeated string tests_created = 3;
  repeated string next_steps = 4;
}

message VerifyRequest {
  PrimitiveKind kind = 1;
  string name = 2;
  string repo_root = 3;  // Optional: repo root path (default: cwd)
  string mode = 4;  // "mock" (default) or "cluster"
}

message VerifyResult {
  string summary = 1;  // "pass" or "fail"
  repeated TestResult tests = 2;
  repeated LintResult lint = 3;
  repeated string next_steps = 4;
}

message TestResult {
  string id = 1;
  string status = 2;  // PASS, FAIL, SKIP, TODO
  string details = 3;
  string file = 4;
  int32 line = 5;
}

message LintResult {
  string tool = 1;  // buf-lint, golangci-lint, eslint
  string status = 2;  // PASS, WARN, FAIL
  int32 issues = 3;
}
```

---

## 7. Current State & Migration

The existing `packages/ameide_core_cli` has legacy commands:

| Command | Status | Action |
|---------|--------|--------|
| `ameide dev` | Legacy | Remove (use Tilt directly) |
| `ameide generate` | Legacy | Remove (use `buf generate`) |
| `ameide model` | Legacy | Remove (stubbed, incomplete) |
| `ameide command` | Legacy | Remove (stubbed) |
| `ameide query` | Legacy | Remove (stubbed) |
| `ameide events` | Migrate | → `ameide events *` |
| `ameide workflows` | Migrate | → `ameide workflows *` |
| `ameide config` | Keep | Still useful |
| `ameide primitive *` | **New** | Implement |

---

## 8. Proto Alignment Without a Service (For Now)

The CLI uses proto-defined message shapes **without requiring a gRPC server**.

### Now: Local CLI

```
┌─────────────────────────────────────────────────────┐
│  Agent / Human                                      │
│       │                                             │
│       ▼                                             │
│  ameide primitive scaffold --json                   │
│       │                                             │
│       ▼                                             │
│  CLI parses flags → fills ScaffoldRequest           │
│       │                                             │
│       ▼                                             │
│  CLI does work on filesystem                        │
│       │                                             │
│       ▼                                             │
│  CLI emits ScaffoldResult as JSON to stdout         │
│       │                                             │
│       ▼                                             │
│  Agent parses JSON (matches proto shape)            │
└─────────────────────────────────────────────────────┘
```

- CLI runs locally in devcontainer
- JSON output matches `ameide_core_proto.primitive.v1` message shapes
- No network, no server
- Agent just runs command + parses JSON

### Later (Optional): Remote Service

```
┌─────────────────────────────────────────────────────┐
│  Agent / CI Bot                                     │
│       │                                             │
│       ▼                                             │
│  PrimitiveServiceClient.Scaffold(ScaffoldRequest)   │
│       │                                             │
│       ▼                                             │
│  gRPC/Connect to ameide-primitive-service           │
│       │                                             │
│       ▼                                             │
│  Service does work (remote builds, central codegen) │
│       │                                             │
│       ▼                                             │
│  Returns ScaffoldResult                             │
└─────────────────────────────────────────────────────┘
```

If needed later:
- Wrap same proto in a gRPC/Connect server
- Use for remote builds, CI bots, central codegen
- CLI becomes thin client around the service
- **Same proto, same semantics** - just different transport

This gives proto alignment (shared types, consistent semantics) without blocking on network services.

---

## 9. Repo Layout (Target)

The CLI assumes a primitives-based repo structure:

```
ameide/
├── packages/
│   ├── ameide_core_proto/           # Buf module (contracts)
│   ├── ameide_sdk_go/               # Generated clients
│   ├── ameide_sdk_ts/
│   └── ameide_sdk_python/
│
├── primitives/
│   ├── domain/
│   │   ├── orders/
│   │   ├── transformation/
│   │   └── ...
│   ├── process/
│   │   ├── l2o/
│   │   └── onboarding/
│   ├── agent/
│   │   └── core_platform_coder/
│   └── uisurface/
│       └── www_ameide_platform/
│
└── gitops/
    ├── clusters/
    │   ├── prod/
    │   └── staging/
    └── primitives/                  # Domain/Process/Agent/UISurface CRDs
```

**Key points:**
- `primitives/` organized by kind (domain, process, agent, uisurface)
- `gitops/primitives/` mirrors `primitives/` for CRDs/manifests
- CLI is aware of both code (`primitives/*`) and GitOps (`gitops/*`)
- Current `services/` would migrate to `primitives/domain/` or `primitives/process/`

**Migration from current structure:**
| Current | Target |
|---------|--------|
| `services/orders/` | `primitives/domain/orders/` |
| `services/inference/` | `primitives/domain/inference/` |
| `services/workflows_runtime/` | `primitives/process/workflows_runtime/` |
| `services/www_ameide_platform/` | `primitives/uisurface/www_ameide_platform/` |

---

## 10. Test Infrastructure Alignment (Backlog 430)

The CLI aligns with the unified test infrastructure defined in [430-unified-test-infrastructure.md](430-unified-test-infrastructure.md).

### Verify Modes

`ameide primitive verify` supports two modes:

| Mode | Flag | Behavior |
|------|------|----------|
| `mock` | `--mode mock` (default) | In-memory stubs, fast, local |
| `cluster` | `--mode cluster` | Real Kubernetes services via Telepresence |

```bash
# Fast local check (default)
ameide primitive verify --kind domain --name orders --json

# Against live cluster
ameide primitive verify --kind domain --name orders --mode cluster --json
```

### Scaffolded Test Structure

Scaffold outputs follow the 430 canonical structure:

```
primitives/domain/{name}/
├── __mocks__/                    # Mock implementations (required by 430)
│   ├── index.ts
│   ├── client.ts                 # createMockTransport()
│   └── fixtures.ts               # Typed fixture data
├── __tests__/
│   ├── unit/
│   └── integration/
│       ├── run_integration_tests.sh  # 430-compliant runner
│       ├── helpers.ts                # Mode-aware client factory
│       └── *.test.ts
```

### Runner Script Contract

Scaffolded `run_integration_tests.sh` follows the 430 contract:

```bash
#!/usr/bin/env bash
set -euo pipefail

source "tools/integration-runner/integration-mode.sh"
MODE="$(integration_mode)"
export INTEGRATION_TEST_MODE="${MODE}"

if [[ "${MODE}" == "cluster" ]]; then
  require_cluster_mode "${MODE}"
  for var in GRPC_ADDRESS; do
    [[ -z "${!var:-}" ]] && { echo "${var} required"; exit 1; }
  done
fi

source "tools/integration-runner/junit-path.sh"
JUNIT_PATH="$(resolve_junit_path {name})"
# ... run tests
```

### Verify Output Formats

Verify emits both JSON and JUnit:

```bash
ameide primitive verify --kind domain --name orders --json
# stdout: VerifyResult JSON
# artifacts/{name}/junit.xml: JUnit XML for CI
```

### Fail Fast

Per 430, verify must:
- Exit non-zero immediately on any failure
- Never skip tests silently
- Never fallback to defaults in cluster mode

---

## 11. GitOps Alignment (Backlog 434)

The CLI aligns with environment naming and GitOps structure from [434-unified-environment-naming.md](434-unified-environment-naming.md).

### Scaffolded GitOps Structure

```
gitops/primitives/{kind}/{name}/
├── values.yaml                   # Base values
├── kustomization.yaml            # Optional kustomize overlay
└── component.yaml                # For ApplicationSet discovery
```

The `component.yaml` allows the single parametrized ApplicationSet to discover new primitives:

```yaml
# gitops/primitives/domain/orders/component.yaml
name: orders
domain: apps                      # ArgoCD Project domain
kind: domain
```

### Required Labels

Scaffolded Helm values include 434-mandated labels:

```yaml
# gitops/primitives/domain/orders/values.yaml
labels:
  ameide.io/tier: backend         # Domain/Process/Agent = backend, UISurface = frontend
  ameide.io/primitive-kind: domain
```

### Tier Mapping

| Primitive Kind | Tier Label |
|----------------|------------|
| Domain | `backend` |
| Process | `backend` |
| Agent | `backend` |
| UISurface | `frontend` |

### No Per-Environment Files

Per 434's single parametrized ApplicationSet design, scaffold does NOT create per-environment values. Environment-specific overrides go in:

```
gitops/sources/values/{env}/primitives/{kind}/{name}.yaml  # Optional override
```

---

## 12. Implementation Notes

- CLI is in Go, using Cobra for command structure
- JSON output uses proto-generated structs (or compatible shapes)
- `--json` flag switches output mode
- Config stored in `~/.ameide/config.yaml`
- Service commands use gRPC with tenant metadata
- Primitive commands are local filesystem operations

---

## 13. Automating the New Service Checklist (Backlog 482)

Backlog [482-adding-new-service](482-adding-new-service.md) defines a manual checklist for spinning up new services. The `ameide primitive scaffold` command **automates** this checklist:

| 482 Checklist Item | `scaffold` Output |
|-------------------|-------------------|
| Decide service type | `--kind domain\|process\|agent\|uisurface` |
| Scaffold files (README, Dockerfiles, Tilt) | `cmd/`, `Dockerfile.dev`, `Dockerfile.release`, `README.md` |
| Define interfaces (proto) | Uses proto from `--proto-path` |
| Secrets & config | ExternalSecret template in gitops |
| Testing | `__mocks__/`, `__tests__/integration/`, runner script |
| GitOps deployment | `component.yaml`, `values.yaml`, kustomization |
| Observability | Health probe stubs, OTel wiring |
| Documentation | README with deps, rotation steps; `catalog-info.yaml` |
| CI/CD wiring | Adds service to workflow matrices |

**Before scaffold:** AI agent creates/edits proto, runs `buf generate`.
**After scaffold:** All 482 checklist items are generated; AI agent implements business logic.

This makes 482 the "what" and 484 the "how" (automated).

---

## 14. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [365-buf-sdks-v2](365-buf-sdks-v2.md) | SDK regeneration, Buf breaking rules, cascade verification |
| [430-unified-test-infrastructure](430-unified-test-infrastructure.md) | Test modes, folder structure, runner contracts |
| [434-unified-environment-naming](434-unified-environment-naming.md) | GitOps structure, namespace labels, tier mapping |
| [435-remote-first-development](435-remote-first-development.md) | Telepresence, cluster mode context |
| [471-transformation-domain](471-transformation-domain.md) | Source of truth for what to build (Step 0 context) |
| [476-security-and-trust](476-security-and-trust.md) | Secret scan, security guardrails in verify |
| [467-backstage](467-backstage.md) | Shared templates with CLI scaffold; UI for human developers |
| [481-service-catalog](481-service-catalog.md) | Backstage catalog integration |
| [482-adding-new-service](482-adding-new-service.md) | Manual checklist that `scaffold` automates |
