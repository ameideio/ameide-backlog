# Ameide CLI – Proto-Aligned CLI for AI Agents

## 1. Purpose

The **Ameide CLI** is a proto-aligned command-line interface that serves as **guardrails for AI agents** doing autonomous development. It is designed to be consumed by:

- **AI Agents** (primary) – automating development work via structured JSON output
- **Developers** (secondary) – running commands interactively when needed

### 1.1 Design Philosophy: Shape, Not Meaning

The CLI knows **shape** (repo layout, proto structure, test conventions) but not **meaning** (business logic, validation rules, domain invariants). This separation is critical:

| CLI Owns (Shape) | Agent Owns (Meaning) |
|------------------|----------------------|
| Directory layout per primitive kind | Business logic & invariants |
| Empty handler/test skeletons | Test assertions & fixtures |
| Proto/SDK freshness checks | What the proto should contain |
| Convention enforcement | How to implement requirements |
| GitOps manifest structure | When/what to deploy |

### 1.2 Three Responsibilities

1. **Describe** – Introspect repo state, detect drift, report what exists vs what's expected
2. **Verify** – Run checks (tests, lint, breaking changes) and report structured pass/fail
3. **Scaffold** – Generate mechanical, proto-driven skeletons (optional, conservative)

The CLI does **no fuzzy reasoning**. All non-determinism lives in agents.

### 1.3 TDD as the Core Development Model

The CLI is built around **Test-Driven Development** principles:

```
RED → GREEN → REFACTOR
```

| Phase | What Happens | CLI Role |
|-------|--------------|----------|
| **RED** | Write failing test first | `plan` outputs test specs; `scaffold` creates failing test stubs |
| **GREEN** | Implement minimal code to pass | `verify` checks if test passes |
| **REFACTOR** | Clean up without breaking | `verify` ensures no regression |

**Key invariants:**
- Tests are **never optional** – every RPC must have tests
- Tests must **fail before implementation** – scaffolded tests call real handlers and fail
- Implementation is **test-driven** – agent writes code to make tests pass, not tests to validate code
- All tests must pass **before commit** – `verify --all` gates the workflow

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
# OBSERVE: What exists, what's expected, what's the delta
ameide primitive describe --json
ameide primitive describe --kind domain --name orders --json

# OBSERVE: What's out of sync (SDK staleness, missing tests, convention issues)
ameide primitive drift --json
ameide primitive drift --kind domain --name orders --json

# PLAN: What work is needed for a primitive
ameide primitive plan \
  --kind domain \
  --name orders \
  --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto --json

# IMPACT: What consumers would be affected by a proto change
ameide primitive impact --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto --json

# VERIFY: Run tests/checks, report structured pass/fail
ameide primitive verify --kind domain --name orders --json
ameide primitive verify --all --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto --json

# SCAFFOLD (optional, conservative): Generate empty skeletons
ameide primitive scaffold \
  --kind domain \
  --name orders \
  --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto \
  --lang go \
  --dry-run --json  # Preview without writing

ameide primitive scaffold \
  --kind domain \
  --name orders \
  --include-gitops \  # Optional: include GitOps manifests
  --json
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

### 4.0 What CLI Does vs What Agents Do

The CLI has **three categories** of behavior:

| Category | CLI Role | Agent Role |
|----------|----------|------------|
| **Safe to Generate** | Directory skeletons, empty handlers, test harness, Dockerfile | — |
| **Verify Only** | Proto breaking, SDK freshness, test results, conventions | Decide how to fix issues |
| **Never Generate** | Business logic, test assertions, SQL schemas, prompts | Full ownership |

### 4.0.1 Safe to Generate (Scaffold)

These are **mechanical, proto-driven, idempotent**:

- **Directory layout** per primitive kind
- **Entrypoints** (`cmd/main.go`, `package.json`)
- **Empty handler/activity files** with function signatures from proto (return `unimplemented` error)
- **Test file skeletons** with imports and **failing test stubs** (not empty - they call the handler and assert it fails)
- **Test harness** (`run_integration_tests.sh`, `__mocks__/`, mode switching)
- **Dockerfile**, `go.mod`/`package.json`, `README.md` stub

**TDD-aligned scaffolding:**
- Scaffolded handlers return `codes.Unimplemented` by default
- Scaffolded tests **call the handler and expect a real response** (not just compile)
- This means freshly scaffolded code **fails tests by design** → proper RED state
- Agent's job is to make tests pass (GREEN), not to write tests from scratch

**Rules:**
- **One-shot**: Scaffold only when folder doesn't exist. Never overwrite existing files.
- **Generated marker**: `// CODEGEN: safe to delete, regenerate with 'ameide primitive scaffold'`
- **GitOps is optional**: Use `--include-gitops` flag; default is `false`
- **Tests must fail**: Scaffolded tests are not empty; they exercise the API and fail until implemented

### 4.0.2 Verify Only (Never Auto-Fix)

The CLI **diagnoses** but never **fixes**:

- Proto breaking changes (via Buf)
- SDK freshness (generated stubs match proto)
- Test presence per RPC
- Convention compliance (folder structure, required files)
- GitOps manifest validity (if present)
- **Security checks** (see §4.0.2.1)

Output is always structured JSON describing *what's wrong*, not *fixing it*.

#### 4.0.2.1 Security Hooks in Verify

`verify` is the integration point for security tooling (aligning with [476-ameide-security-trust.md](476-ameide-security-trust.md)):

| Check | Tool | When |
|-------|------|------|
| **Secret scan** | gitleaks, trufflehog | Every verify run |
| **Dependency vulnerabilities** | `npm audit`, `govulncheck`, Snyk | Every verify run |
| **SAST** | Semgrep, CodeQL | `--check security` flag |
| **License compliance** | license-checker | `--check security` flag |
| **Policy checks** | OPA/Gatekeeper | For GitOps manifests |

Example output:

```json
{
  "security": {
    "secret_scan": {"status": "PASS", "tool": "gitleaks"},
    "dependency_vulns": {"status": "WARN", "critical": 0, "high": 2, "tool": "govulncheck"},
    "sast": {"status": "PASS", "tool": "semgrep"}
  }
}
```

**Default behavior:**
- Secret scan and dependency vulns run on every `verify` (fast, essential)
- SAST runs only with `--check security` (slower, optional locally, required in CI)

#### 4.0.2.2 Command/Event Discipline Checks

`verify` enforces the "no imperative writes" principle from [472 §2.8.6](472-ameide-information-application.md):

| Check | Rule | Severity |
|-------|------|----------|
| **RPC naming** | Domain/Process RPCs must use business verbs, not CRUD generics | WARN |
| **Forbidden prefixes** | `Update*`, `Set*`, `Patch*`, `Modify*` on Domain services | WARN |
| **Event emission** | Commands that write should have corresponding event types | INFO |

**Allowed command prefixes** (business verbs):
`Create`, `Place`, `Cancel`, `Approve`, `Reject`, `Assign`, `Submit`, `Start`, `Complete`, `Archive`, `Qualify`, `Win`, `Lose`, `Reassign`, `Publish`, `Close`, `Reopen`...

**Forbidden generic prefixes**:
`Update`, `Set`, `Patch`, `Modify`, `Change` (too vague - doesn't express business intent)

**Exception**: `Delete` is acceptable for hard-delete semantics when business intent is clear (e.g., `DeleteDraftOrder`)

Example output:

```json
{
  "command_event_discipline": {
    "rpc_naming": {
      "status": "WARN",
      "issues": [
        {"rpc": "UpdateOrder", "file": "orders.proto", "line": 42, "suggestion": "Use business verb like CancelOrder, AdjustOrderLines, ReassignOrder"}
      ]
    },
    "event_coverage": {
      "status": "PASS",
      "commands_with_events": 12,
      "commands_without_events": 0
    }
  }
}
```

**Rationale**: Commands should express **what the business is doing**, not which field is changing. The domain decides internally what "cancel" or "approve" means. See [472 §2.8.6](472-ameide-information-application.md) for the full CQRS pattern.

#### 4.0.2.3 Event Reliability Checks

`verify` enforces EDA reliability principles from [472 §3.3](472-ameide-information-application.md):

| Check | Rule | Severity |
|-------|------|----------|
| **Outbox wiring** | Domain primitives use outbox pattern for events | WARN |
| **Event emission** | Every state-mutating command emits ≥1 event | WARN |
| **Idempotency guard** | Event handlers check for duplicates (inbox pattern) | WARN |
| **Tenant validation** | Event handlers validate `tenant_id` context | WARN |
| **Schema versioning** | Events in `events/v1` package follow Buf rules | WARN |

> **Note**: All EDA reliability checks are WARN severity because they enforce the core invariants from [470 §8-13](470-ameide-vision.md). Violations indicate potential data loss, duplicate processing, or security issues.

Example output:

```bash
$ ameide primitive verify domain/sales
✓ Command/Query naming discipline
✓ Event emission coverage (8/8 commands emit events)
✓ Outbox pattern wiring
✗ Event handler idempotency
  → HandleOpportunityWon: missing inbox check (see 472 §3.3.2)
  → HandleQuoteApproved: missing inbox check

2 issues found.
```

**References**:
- Outbox pattern: [472 §3.3.1.1](472-ameide-information-application.md)
- Idempotency/Inbox: [472 §3.3.2](472-ameide-information-application.md)
- Multi-tenant isolation: [472 §3.3.7](472-ameide-information-application.md)
- Full EDA principles: [496-eda-principles.md](496-eda-principles.md)

#### 4.0.2.4 Agentic Verification in the Development Process

Principle verification is **not just a CI gate**—it's an integral part of the agentic development workflow. AI agents MUST run `verify` checks throughout their development cycle, not just before commit.

**When agents run verification:**

| Development Phase | Verification Action | Purpose |
|-------------------|---------------------|---------|
| **Before implementation** | `verify --check naming` | Validate proto RPC names follow business verb conventions |
| **After each handler** | `verify --check eda` | Ensure outbox wiring, event emission, idempotency |
| **After adding events** | `verify --check schema` | Validate Buf compatibility, event metadata |
| **Before PR** | `verify --all` | Full verification including security checks |
| **On consumer changes** | `verify --cascade` | Verify all affected downstream primitives |

**Agentic verification loop:**

```
┌─────────────────────────────────────────────────────────────────┐
│  1. WRITE CODE                                                  │
│     └─ Agent implements handler/consumer                        │
│                                                                 │
│  2. VERIFY IMMEDIATELY                                          │
│     └─ ameide primitive verify --kind domain --name orders      │
│     └─ Agent reads structured JSON output                       │
│                                                                 │
│  3. INTERPRET VIOLATIONS                                        │
│     └─ Agent reasons about each WARN/FAIL                       │
│     └─ Agent decides: fix now vs defer vs ask human             │
│                                                                 │
│  4. FIX OR ESCALATE                                             │
│     └─ If fixable: agent modifies code, returns to step 1       │
│     └─ If unclear: agent presents options to human              │
│     └─ If architectural: agent flags for design review          │
│                                                                 │
│  5. CONTINUE ONLY WHEN GREEN                                    │
│     └─ Agent does not proceed to next task until verify passes  │
└─────────────────────────────────────────────────────────────────┘
```

**Key invariant**: Agents treat `verify` output as **actionable feedback**, not just pass/fail gates. When verification fails, the agent:

1. Reads the structured JSON output
2. Identifies the specific violation (file, line, rule)
3. Looks up the referenced documentation (e.g., "see 472 §3.3.2")
4. Applies the documented pattern to fix the issue
5. Re-runs verification to confirm the fix

**Example agent reasoning:**

```
verify output:
{
  "idempotency_guard": {
    "status": "WARN",
    "issues": [
      {"handler": "HandleOrderShipped", "file": "handlers/shipping.go", "line": 42}
    ]
  }
}

Agent reasoning:
1. HandleOrderShipped lacks inbox check
2. Reference: 472 §3.3.2 describes inbox pattern
3. Action: Add inbox.IsProcessed() check before processing
4. Re-verify after fix
```

**Why agentic verification matters:**

- **Shift left**: Catch EDA violations during development, not in code review
- **Self-correction**: Agents fix issues autonomously without human intervention
- **Consistent quality**: Every primitive follows the same patterns
- **Documentation-driven**: Agents learn patterns from backlog references
- **Reduced review burden**: Human reviewers focus on business logic, not boilerplate patterns

> **Cross-reference**: The agentic development sequence is defined in [§5 Intelligent Agent Sequence](#5-intelligent-agent-sequence). Verification is part of the ACT phase (§5.5), specifically Step 4.3 (Implementation TDD Loop).

### 4.0.3 Never Generate (Agent's Job)

The CLI **never** generates:

- Business logic & invariants
- Test assertions & fixtures
- SQL schemas & migrations
- Agent prompts & orchestration
- Non-trivial UI components

These require **meaning**, which only agents (with Transformation context) can provide.

---

### 4.1 What Gets Scaffolded (When Requested)

> **Generation philosophy**: CLI scaffolds technical shape (Buf/Connect-style handler interfaces, Backstage-style repo structure), not business meaning. This follows the "shape, not meaning" principle from §1.1. For the full rationale and vendor pattern alignment, see [472 §2.9](472-ameide-information-application.md).

When `--scaffold` is used, each primitive kind produces:

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

## 5. Intelligent Agent Sequence

This section describes an **intelligent, reasoning-based** development sequence. The key insight: agents should **observe → reason → dry-run → act**, not blindly scaffold and hope.

> **Cross-reference**: This sequence aligns with [477-primitive-stack.md §5.2](477-primitive-stack.md) which defines the human-gated workflow (research → approval → proto → implement → UAT).

### 5.1 The Observe-Reason-Act Loop

```
┌─────────────────────────────────────────────────────────────────┐
│  1. OBSERVE                                                     │
│     └─ What exists? What's expected? What's the delta?          │
│                                                                 │
│  2. REASON                                                      │
│     └─ Research standards. Compare approaches. Form hypothesis. │
│                                                                 │
│  3. DRY-RUN                                                     │
│     └─ Simulate changes. Check for drift. Validate assumptions. │
│                                                                 │
│  4. HUMAN GATE (if significant)                                 │
│     └─ Present findings. Get approval before mutations.         │
│                                                                 │
│  5. ACT                                                         │
│     └─ Make minimal changes. Verify. Iterate.                   │
│                                                                 │
│  6. HUMAN UAT                                                   │
│     └─ Final acceptance before merge.                           │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Phase 1: Observe (Assess Current State)

Before any changes, the agent builds a complete picture of the current state.

**Step 1.1: Load Transformation Context**

Query the Transformation Domain for what *should* exist:

```bash
# What does the backlog say we need?
ameide transformation context --json
```

Returns design artifacts, requirements, ProcessDefinitions, AgentDefinitions.

**Step 1.2: Describe Current Repo State**

```bash
# What primitives exist? What's their state?
ameide primitive describe --json
```

**Output:**
```json
{
  "primitives": [
    {
      "kind": "DOMAIN",
      "name": "orders",
      "status": "EXISTS",
      "path": "primitives/domain/orders",
      "proto": "ameide/orders/v1/orders.proto",
      "drift": {
        "sdk_stale": true,
        "missing_tests": ["orders.cancel_order.success"],
        "gitops_missing": false
      }
    }
  ],
  "expected_but_missing": [
    {"kind": "PROCESS", "name": "l2o", "reason": "Referenced in ProcessDefinition but no primitive exists"}
  ]
}
```

**Step 1.3: Detect Drift**

```bash
# What's out of sync?
ameide primitive drift --json
```

**Output:**
```json
{
  "proto_sdk_drift": [
    {"proto": "orders/v1/orders.proto", "sdk": "ameide-sdk-go", "status": "STALE", "action": "Run buf generate"}
  ],
  "test_coverage_drift": [
    {"primitive": "orders", "rpc": "CancelOrder", "tests": 0, "expected": 1}
  ],
  "convention_drift": [
    {"primitive": "orders", "issue": "Missing README.md", "severity": "WARN"}
  ]
}
```

### 5.3 Phase 2: Reason (Research & Plan)

The agent now has data. Time to think.

**Step 2.1: Research Standard Solutions**

Before proposing changes, the agent researches:
- Industry patterns for similar problems
- Existing libraries/frameworks
- How other primitives in this repo solved similar issues

This is **agent reasoning**, not CLI. The CLI just provides data; the agent decides.

**Step 2.2: Form a Hypothesis**

Agent produces a plan:

```
"To add order cancellation:
1. Add CancelOrder RPC to orders.proto (additive, non-breaking)
2. Regenerate SDKs
3. Implement handler in orders domain
4. Add integration test
5. Update L2O process to handle cancellation events"
```

**Step 2.3: Validate Hypothesis with Dry-Run**

```bash
# What would happen if I modified this proto?
ameide primitive impact --proto-path orders/v1/orders.proto --json
```

**Output:**
```json
{
  "consumers": [
    {"kind": "DOMAIN", "name": "orders", "imports": ["OrdersService"]},
    {"kind": "PROCESS", "name": "l2o", "imports": ["OrdersServiceClient"]},
    {"kind": "UISURFACE", "name": "www_platform", "imports": ["Order", "OrderStatus"]}
  ],
  "cascade_tests_required": 25,
  "estimated_scope": "MEDIUM"
}
```

```bash
# Would this be a breaking change?
buf breaking --against .git#branch=main packages/ameide_core_proto
```

### 5.4 Phase 3: Human Review Gate

For significant changes (new primitives, proto modifications, architectural decisions), the agent **stops and presents findings**:

```
## Proposed Change: Add Order Cancellation

### Research Findings
- Saga pattern is industry standard for order cancellation
- Temporal compensation activities align with our Process primitive model
- Similar implementation exists in `primitives/process/onboarding`

### Impact Analysis
- 3 primitives affected (orders, l2o, www_platform)
- 25 tests will need to pass
- Non-breaking proto change (additive only)

### Recommended Approach
1. Add CancelOrder RPC with compensation metadata
2. Implement as Temporal activity with rollback
3. Emit OrderCancelled event for downstream consumers

**Awaiting approval to proceed.**
```

Human reviews and approves (or redirects).

### 5.5 Phase 4: Act (Minimal, Verified Changes)

Only after approval does the agent make changes. Each change is **small, tested, verified**.

**Step 4.1: Proto Changes**

```bash
# Agent edits proto
# Then validates:
buf lint packages/ameide_core_proto
buf breaking --against .git#branch=main packages/ameide_core_proto
```

**Step 4.2: SDK Regeneration**

```bash
buf generate

# Verify SDKs compile
ameide primitive verify --check sdk-freshness --json
```

**Step 4.3: Implementation (TDD Loop)**

The agent follows **strict TDD discipline**: RED → GREEN → REFACTOR.

```bash
# Get the test plan
ameide primitive plan --kind domain --name orders --json
```

**Output:**
```json
{
  "tests_to_create": [
    {"id": "orders.cancel_order.success", "description": "Cancelling valid order returns success"},
    {"id": "orders.cancel_order.already_shipped", "description": "Cancelling shipped order returns error"}
  ]
}
```

**TDD Loop (per test):**

```
┌─────────────────────────────────────────────────────────────────┐
│  1. RED: Write failing test first                               │
│     - Create test with real assertions (not empty)              │
│     - Run verify → MUST FAIL                                    │
│     - If test passes without implementation → test is wrong     │
│                                                                 │
│  2. GREEN: Write minimal code to pass                           │
│     - Implement only what's needed for THIS test                │
│     - No speculative features                                   │
│     - Run verify → MUST PASS                                    │
│                                                                 │
│  3. REFACTOR: Clean up (optional)                               │
│     - Improve code quality                                      │
│     - Run verify → MUST STILL PASS                              │
│                                                                 │
│  4. NEXT: Move to next test in plan                             │
└─────────────────────────────────────────────────────────────────┘
```

**Concrete sequence:**

```bash
# Step 1: RED - Write failing test
# Agent writes test assertion in orders.cancel_order.success test file

# Verify it fails (this is required!)
ameide primitive verify --kind domain --name orders --test orders.cancel_order.success --json
# Expected: {"status": "FAIL", "reason": "CancelOrder not implemented"}

# Step 2: GREEN - Implement minimal code
# Agent implements CancelOrder handler

# Verify it passes
ameide primitive verify --kind domain --name orders --test orders.cancel_order.success --json
# Expected: {"status": "PASS"}

# Step 3: REFACTOR (if needed)
# Agent cleans up code
# Verify still passes

# Step 4: NEXT - Move to next test
# Repeat for orders.cancel_order.already_shipped
```

**Key TDD invariants the CLI enforces:**

| Check | CLI Behavior |
|-------|--------------|
| Test exists before implementation | `plan` outputs tests_to_create; `verify` checks test presence |
| Test must fail first | Agent should run verify and confirm FAIL before implementing |
| Minimal implementation | Agent decides, but `verify` ensures no regression |
| All tests pass before commit | `verify --all` gates the commit |

**Step 4.4: Cascade Verification**

After the primitive is green, verify all consumers:

```bash
ameide primitive verify --all --proto-path orders/v1/orders.proto --json
```

This runs tests for **all affected primitives**, not just the one being modified.

### 5.6 Phase 5: Human UAT

Before merge, human verifies:
- Implementation meets requirements
- No unexpected side effects
- Code quality acceptable

Agent presents final summary:

```
## Implementation Complete: Order Cancellation

### Changes Made
- Added CancelOrder RPC to orders.proto
- Implemented handler with Saga compensation
- Added 2 integration tests (both passing)
- Updated L2O process to emit compensation on cancel

### Verification Results
- All 27 affected tests pass
- No breaking changes detected
- Lint clean
- Secret scan clean

**Ready for UAT.**
```

### 5.7 Summary: Intelligent Sequence

| Phase | Agent Action | CLI Support | Human Involvement |
|-------|--------------|-------------|-------------------|
| **Observe** | Gather state | `describe`, `drift` | None |
| **Reason** | Research, plan | `impact`, `plan` | None |
| **Review Gate** | Present findings | — | **Approval required** |
| **Act** | Implement, test | `verify`, `scaffold` | None |
| **UAT** | Present results | `verify --all` | **Acceptance required** |

Key differences from naive "scaffold → implement → commit":

1. **Drift detection first** – Don't assume clean state
2. **Research before code** – Agent thinks, not just executes
3. **Dry-run before mutate** – Understand impact before changes
4. **Human gates** – Approval for significant changes
5. **Cascade verification** – All consumers must pass, not just the target
6. **Small, verified steps** – Each change is tested before the next

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

> **Note:** For the authoritative repo layout and primitive/operator/GitOps split, see [477-primitive-stack.md §2](477-primitive-stack.md). This section provides CLI-specific context. For operator publishing & deployment flow, see [495-ameide-operators.md §6](495-ameide-operators.md).

### 9.1 Core Repo vs GitOps Repo Split

The CLI operates across **two logical repositories** (which may be consolidated or separate):

| Repo | Contains | CLI Focus |
|------|----------|-----------|
| **Core repo** | Code, proto, SDKs, operators, CRD *schemas* | `describe`, `drift`, `verify`, `scaffold` (code) |
| **GitOps repo** | CRD *instances*, environment values, ApplicationSets | `scaffold --include-gitops`, `verify --gitops` |

**Key distinction:**
- **CRD schemas/kinds + operators** live in the **core repo** (`config/crd/bases/`, `operators/`)
- **CRD instances + environment config** live in the **GitOps repo** (`gitops/primitives/`, `gitops/environments/`)

### 9.2 CLI Flags for Repo Roots

All `primitive` commands accept:

```bash
--repo-root <path>      # Core repo root (default: cwd or auto-detected)
--gitops-root <path>    # GitOps repo root (default: {repo-root}/gitops or separate repo)
```

Examples:

```bash
# Monorepo (gitops is a subdirectory)
ameide primitive describe --json
# Uses: repo-root=cwd, gitops-root=cwd/gitops

# Separate repos
ameide primitive describe --repo-root ~/ameide-core --gitops-root ~/ameide-gitops --json

# GitOps-only operations
ameide primitive verify --gitops-root ~/ameide-gitops --check gitops-only --json
```

### 9.3 Directory Structure

**Core repo** (where AI coder sandbox lives):

```
ameide/
├── packages/
│   ├── ameide_core_proto/           # Buf module (contracts)
│   ├── ameide_sdk_go/               # Generated clients
│   ├── ameide_sdk_ts/
│   └── ameide_sdk_python/
│
├── config/
│   └── crd/
│       └── bases/                   # CRD SCHEMAS (Domain, Process, Agent, UISurface)
│           ├── domain.ameide.io.yaml
│           ├── process.ameide.io.yaml
│           ├── agent.ameide.io.yaml
│           └── uisurface.ameide.io.yaml
│
├── operators/                       # Operator code (reconciles CRD instances)
│   ├── domain/
│   ├── process/
│   ├── agent/
│   └── uisurface/
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
```

**GitOps repo** (CRD instances and environment config):

```
ameide-gitops/  # or gitops/ subdirectory in monorepo
├── environments/
│   ├── dev/
│   │   └── argocd/                  # ApplicationSets for dev
│   ├── staging/
│   └── prod/
│
├── primitives/                      # CRD INSTANCES (actual runtime wiring)
│   ├── domain/
│   │   └── orders/
│   │       ├── domain.yaml          # Domain CRD instance
│   │       └── values.yaml          # Helm values
│   ├── process/
│   ├── agent/
│   └── uisurface/
│
└── sources/
    └── values/
        ├── dev/
        ├── staging/
        └── prod/                    # Environment-specific overrides
```

### 9.4 Key Points

- **CRD schemas** (what a Domain/Process/Agent/UISurface looks like) → **core repo**
- **CRD instances** (the actual orders Domain, the actual l2o Process) → **GitOps repo**
- **Operators** (code that reconciles CRs into Deployments) → **core repo**
- The CLI scaffolds **code** in core repo; **CRD instances** go in gitops repo (via `--include-gitops`)
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

## 14. Phased Implementation

Rather than building the full CLI at once, we implement in phases that deliver value incrementally.

### Phase 1: Describe & Verify Only (MVP)

**Goal:** Give agents visibility into repo state without any generation.

**Commands:**
```bash
ameide primitive describe --json    # What exists, what's expected, what's the delta
ameide primitive drift --json       # SDK staleness, test coverage gaps, convention issues
ameide primitive impact --json      # What consumers would be affected by a proto change
ameide primitive verify --json      # Run tests, lint, breaking checks
```

**What's NOT included:**
- `scaffold` command (agents create files manually)
- GitOps generation
- Any file mutations

**Value:** Agents can reason about the codebase with structured data. They write code themselves but have guardrails to validate correctness.

### Phase 2: Conservative Scaffolding

**Goal:** Generate safe, mechanical structures that agents would otherwise write identically – with **TDD-ready failing tests**.

**Commands:**
```bash
ameide primitive scaffold --kind domain --name orders --lang go --json
# Generates:
#   - Directory structure
#   - Handler files returning `codes.Unimplemented`
#   - Test files that CALL handlers and FAIL (proper RED state)
#   - Dockerfile, go.mod, README stub
```

**Flags:**
- `--include-gitops=false` (default) – No GitOps generation
- `--dry-run` – Show what would be created without writing

**TDD alignment:**
- Handlers return `codes.Unimplemented` by default
- Tests call the handlers and assert on expected behavior (not empty)
- **Freshly scaffolded code FAILS all tests** → agent starts in RED state
- Agent's job: make tests GREEN by implementing handlers

**Rules:**
- **One-shot only**: If folder exists, refuse to scaffold
- **No business logic**: Handlers return unimplemented, not real logic
- **Tests must fail**: Scaffolded tests exercise the API and fail until implemented

### Phase 3: GitOps & Test Harness (Optional)

**Goal:** For teams that want more automation, add GitOps and test infrastructure generation.

**Commands:**
```bash
ameide primitive scaffold --include-gitops --json
# Adds:
#   - gitops/primitives/{kind}/{name}/values.yaml
#   - gitops/primitives/{kind}/{name}/component.yaml

ameide primitive scaffold --include-test-harness --json
# Adds:
#   - __mocks__/ directory with client stubs
#   - run_integration_tests.sh (430-compliant)
#   - Mode-aware helper files
```

**Both remain optional flags**, not defaults.

### Phase 4: Transformation Integration (Future)

**Goal:** CLI can query Transformation Domain for context.

**Commands:**
```bash
ameide transformation context --json  # What should exist per design artifacts
ameide transformation sync --json     # Compare design artifacts to repo state
```

**Value:** Agents get "what should be built" from Transformation, not just "what exists" from repo.

### Implementation Timeline

| Phase | Scope | Complexity | Prerequisite |
|-------|-------|------------|--------------|
| 1 | describe, drift, impact, verify | Medium | Buf integration |
| 2 | scaffold (code only) | Low | Phase 1 |
| 3 | scaffold (gitops, test harness) | Low | Phase 2 |
| 4 | transformation context | High | Transformation APIs |

**Start with Phase 1.** It delivers the most value (visibility, guardrails) with the least risk (no mutations).

---

## 15. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [365-buf-sdks-v2](365-buf-sdks-v2.md) | SDK regeneration, Buf breaking rules, cascade verification |
| [430-unified-test-infrastructure](430-unified-test-infrastructure.md) | Test modes, folder structure, runner contracts |
| [434-unified-environment-naming](434-unified-environment-naming.md) | GitOps structure, namespace labels, tier mapping |
| [435-remote-first-development](435-remote-first-development.md) | Telepresence, cluster mode context |
| [471-ameide-business-architecture](471-ameide-business-architecture.md) | Transformation Domain as source of truth |
| [476-ameide-security-trust](476-ameide-security-trust.md) | Secret scan, security guardrails in verify |
| [467-backstage](467-backstage.md) | Shared templates with CLI scaffold; UI for human developers |
| [477-primitive-stack](477-primitive-stack.md) | Repo layout, operator structure, AI coder sequence |
| [481-service-catalog](481-service-catalog.md) | Backstage catalog integration |
| [482-adding-new-service](482-adding-new-service.md) | Manual checklist that `scaffold` automates |
| [495-ameide-operators](495-ameide-operators.md) | Operator CRD shapes, reconciliation logic, publishing & deployment |

---

## 16. Industry Patterns & Ameide Alignment

This section documents how Ameide's scaffolding approach relates to industry tools and patterns.

### 16.1 What Existing Tools Generate

| Tool | What It Generates | Ameide Equivalent | Reference |
|------|-------------------|-------------------|-----------|
| **Backstage Templates** | Repo skeleton, CI, Dockerfile, catalog file | `ameide primitive scaffold` | [467](467-backstage.md) §10 |
| **Buf/Connect** | Proto stubs + typed clients/servers | SDK generation via `buf generate` | [365](365-buf-sdks-v2.md) |
| **Kubebuilder** | CRD types + controller skeleton | `operators/*` in core repo | [495](495-ameide-operators.md) §0 |
| **JHipster/Context Mapper** | Full CRUD stack (entities, repos, controllers, frontend) | ❌ **Explicitly avoided** | — |

### 16.2 Why We Don't Do JHipster-Style Generation

JHipster and similar tools generate **full business logic** from domain models. This creates problems:

1. **Generated code becomes "untouchable"** — regenerating overwrites hand edits
2. **Domain invariants get smudged into templates** — two sources of truth
3. **Developers/agents think in generator capabilities** — not the actual domain model

Ameide's philosophy: **generate wiring, never meaning** (see §1.1).

### 16.3 Ameide Scaffolding Philosophy

| We Generate (Safe) | We Never Generate |
|--------------------|-------------------|
| Directory layout per primitive kind | Business logic & invariants |
| Empty handler signatures from proto | Test assertions & fixtures |
| Failing test stubs (proper TDD RED state) | SQL schemas & migrations |
| Dockerfile, go.mod/package.json | Prompts & agent reasoning |
| Test harness (`run_integration_tests.sh`) | Domain-specific validation |

This mirrors the **Connect/Buf pattern**: define service in proto, generate type-safe scaffolding, implement business logic by hand (or agent).

### 16.4 Mapping Vendor Patterns to Ameide Components

| Vendor Pattern | Ameide Component | How It Works |
|---------------|------------------|--------------|
| Backstage Software Templates | CLI `scaffold` + Backstage portal | Same templates, two consumption paths (CLI for AI, Backstage for humans) |
| Buf code generation | `buf generate` → SDK packages | Proto → TS/Go/Python stubs in workspace SDKs |
| Kubebuilder controller-gen | Operator implementations | CRD types + controller skeleton in `operators/*` |
| Connect server handlers | Domain primitive handlers | Proto service → handler interfaces + empty implementations |

### 16.5 Per-Primitive Scaffold Outputs

#### Domain Primitive (Go)

```text
primitives/domain/{name}/
├── cmd/main.go                      # Server bootstrap with AmeideClient
├── internal/
│   ├── handlers/                    # One file per RPC (returns Unimplemented)
│   ├── repository/                  # Data access interfaces
│   └── tests/                       # Failing test stubs per RPC
├── Dockerfile
├── go.mod
└── README.md

gitops/primitives/domain/{name}/
├── values.yaml                      # Helm values for Domain CRD
└── kustomization.yaml
```

**What CLI scaffolds**: Structure, handler signatures, failing tests
**What agent implements**: Business rules, queries, test assertions

#### Process Primitive (Go/Temporal)

```text
primitives/process/{name}/
├── cmd/main.go                      # Temporal worker registration
├── internal/
│   ├── workflow/workflow.go         # Stubbed workflow function
│   ├── activities/                   # One file per activity
│   └── tests/                       # Workflow test harness
├── Dockerfile
└── go.mod
```

**What CLI scaffolds**: Worker bootstrap, workflow/activity signatures
**What agent implements**: Orchestration logic, compensations

#### Agent Primitive

```text
primitives/agent/{name}/
├── cmd/main.go                      # Agent runtime harness
├── internal/
│   ├── tools/                       # Tool stubs (call SDK clients)
│   └── tests/                       # Agent test harness
├── prompts/                         # Empty prompt templates
├── Dockerfile
└── go.mod
```

**What CLI scaffolds**: Harness, tool stubs
**What agent implements**: Tool logic, prompts, reasoning

#### UISurface Primitive (Next.js)

```text
primitives/uisurface/{name}/
├── src/
│   ├── app/page.tsx                 # Basic page skeleton
│   ├── lib/api.ts                   # SDK client wiring
│   └── components/                  # Empty component stubs
├── package.json
├── Dockerfile
└── README.md
```

**What CLI scaffolds**: Next.js skeleton, SDK wiring
**What agent implements**: UI components, pages, styling

### 16.6 Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad | Ameide Approach |
|--------------|--------------|-----------------|
| Generate entity structs beyond proto | Creates second source of truth | Proto messages ARE the entities |
| Generate migration SQL | Schema is implementation detail | Agent writes migrations |
| Generate form layouts from proto | UI is not derivable from data model | Agent implements UI |
| Generate validation rules | Business invariants are domain logic | Agent implements in handlers |

### 16.7 References

- [Backstage Software Templates](https://backstage.io/docs/features/software-templates/)
- [Buf Code Generation](https://buf.build/docs/generate/)
- [Connect Getting Started](https://connectrpc.com/docs/go/getting-started/)
- [Kubebuilder Markers](https://book.kubebuilder.io/reference/markers)
- [Red Hat: 10 Tips for Better Backstage Templates](https://developers.redhat.com/articles/2025/03/17/10-tips-better-backstage-software-templates)
