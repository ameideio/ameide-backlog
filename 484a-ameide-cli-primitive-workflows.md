# 484a – Ameide CLI: Primitive Workflows & TDD

> **Deprecation notice (520):** Any workflow here that uses `ameide primitive scaffold` as the inner-loop generator is deprecated. Canonical v2 uses **`buf generate`** (pinned plugins, deterministic outputs, generated-only roots, regen-diff CI gate). See `backlog/520-primitives-stack-v2.md`.

**Status:** Active
**Audience:** AI agents (primary), developers (secondary)
**Scope:** Primitive commands, TDD loop, agentic development sequence, verification checks

> **Parent document**: [484 – Ameide CLI Overview](484-ameide-cli.md)

## Grounding & contract alignment

- **Primitive CLI surface:** Defines the `ameide primitive` workflows for the Ameide primitives (Domain/Process/Agent/UISurface/Projection/Integration) introduced in `470-ameide-vision.md` and `472-ameide-information-application.md`. This 484a document focuses on the currently implemented CLI/operator surface (Domain/Process/Agent/UISurface) and uses the shared status/condition semantics from `502-domain-vertical-slice.md` and `495-ameide-operators.md`.  
- **Scrum/agent usage:** Provides the guardrail and verification flows that AmeideCoder uses internally in the agent stack (`504-agent-vertical-slice.md`, `505-agent-developer-v2*.md`) when operating on primitives, including the Scrum-related ones built on `300-400/367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, and `508-scrum-protos.md`.

---

## 1. Primitive Commands Overview

The `ameide primitive` subcommands support AI agents doing development work on primitives (the six-primitives model), but the implemented workflows here focus on Domain/Process/Agent/UISurface.

```bash
# OBSERVE: What exists, what's expected, what's the delta
ameide primitive describe --json
ameide primitive describe --kind domain --name orders --json

# OBSERVE: What's out of sync
ameide primitive drift --json

# PLAN: What work is needed
ameide primitive plan --kind domain --name orders \
  --proto-path packages/ameide_core_proto/src/ameide/orders/v1/orders.proto --json

# IMPACT: What consumers would be affected
ameide primitive impact --proto-path <path> --json

# VERIFY: Run tests/checks
ameide primitive verify --kind domain --name orders --json
ameide primitive verify --all --proto-path <path> --json

# SCAFFOLD (optional, conservative): Generate empty skeletons
ameide primitive scaffold --kind domain --name orders --lang go --dry-run --json
```

---

## 2. What CLI Does vs What Agents Do

The CLI has **three categories** of behavior:

| Category | CLI Role | Agent Role |
|----------|----------|------------|
| **Safe to Generate** | Directory skeletons, empty handlers, test harness, Dockerfile | — |
| **Verify Only** | Proto breaking, SDK freshness, test results, conventions | Decide how to fix issues |
| **Never Generate** | Business logic, test assertions, SQL schemas, prompts | Full ownership |

### 2.1 Safe to Generate (Scaffold)

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

**Primitive destination:** CLI scaffolds always land inside `primitives/{primitive_kind}/{name}` so the same guardrails apply to both human and agent workflows. `service_catalog/...` stays hand-authored (Backstage-focused) while we pivot the automation toward agentic coding only.

**Rules:**
- **One-shot**: Scaffold only when folder doesn't exist. Never overwrite existing files.
- **Generated marker**: `// CODEGEN: safe to delete, regenerate with 'ameide primitive scaffold'`
- **GitOps is optional**: Use `--include-gitops` flag; default is `false`
- **Tests must fail**: Scaffolded tests are not empty; they exercise the API and fail until implemented

### 2.2 Verify Only (Never Auto-Fix)

The CLI **diagnoses** but never **fixes**:

- Proto breaking changes (via Buf)
- SDK freshness (generated stubs match proto)
- Test presence per RPC
- Convention compliance (folder structure, required files)
- GitOps manifest validity (if present)
- Security checks (see §3.1)
- EDA reliability checks (see §3.3)

Output is always structured JSON describing *what's wrong*, not *fixing it*.

### 2.3 Never Generate (Agent's Job)

The CLI **never** generates:

- Business logic & invariants
- Test assertions & fixtures
- SQL schemas & migrations
- Agent prompts & orchestration
- Non-trivial UI components

These require **meaning**, which only agents (with Transformation context) can provide.

---

## 3. Verification Checks

### 3.1 Security Hooks in Verify

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

### 3.2 Command/Event Discipline Checks

`verify` enforces the "no imperative writes" principle from [472 §2.8.6](472-ameide-information-application.md):

| Check | Rule | Severity |
|-------|------|----------|
| **RPC naming** | Domain/Process RPCs must use business verbs, not CRUD generics | WARN |
| **Forbidden prefixes** | `Update*`, `Set*`, `Patch*`, `Modify*` on Domain services | WARN |
| **Event emission** | Commands that write should have corresponding event types | WARN |

**Allowed command prefixes** (business verbs):
`Create`, `Place`, `Cancel`, `Approve`, `Reject`, `Assign`, `Submit`, `Start`, `Complete`, `Archive`, `Qualify`, `Win`, `Lose`, `Reassign`, `Publish`, `Close`, `Reopen`...

**Forbidden generic prefixes**:
`Update`, `Set`, `Patch`, `Modify`, `Change` (too vague - doesn't express business intent)

**Exception**: `Delete` is acceptable for hard-delete semantics when business intent is clear (e.g., `DeleteDraftOrder`)

### 3.3 Event Reliability Checks (EDA)

`verify` enforces EDA reliability principles from [472 §3.3](472-ameide-information-application.md) and [496-eda-principles.md](496-eda-principles.md):

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

---

## 4. Agentic Verification in the Development Process

Principle verification is **not just a CI gate**—it's an integral part of the agentic development workflow. AI agents MUST run `verify` checks throughout their development cycle, not just before commit.

### 4.1 When Agents Run Verification

| Development Phase | Verification Action | Purpose |
|-------------------|---------------------|---------|
| **Before implementation** | `verify --check naming` | Validate proto RPC names follow business verb conventions |
| **After each handler** | `verify --check eda` | Ensure outbox wiring, event emission, idempotency |
| **After adding events** | `verify --check schema` | Validate Buf compatibility, event metadata |
| **Before PR** | `verify --all` | Full verification including security checks |
| **On consumer changes** | `verify --cascade` | Verify all affected downstream primitives |

### 4.2 Agentic Verification Loop

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

### 4.3 Example Agent Reasoning

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

### 4.4 Why Agentic Verification Matters

- **Shift left**: Catch EDA violations during development, not in code review
- **Self-correction**: Agents fix issues autonomously without human intervention
- **Consistent quality**: Every primitive follows the same patterns
- **Documentation-driven**: Agents learn patterns from backlog references
- **Reduced review burden**: Human reviewers focus on business logic, not boilerplate patterns

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

```bash
ameide transformation context --json
```

Returns design artifacts, requirements, ProcessDefinitions, AgentDefinitions.

**Step 1.2: Describe Current Repo State**

```bash
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
buf lint packages/ameide_core_proto
buf breaking --against .git#branch=main packages/ameide_core_proto
```

**Step 4.2: SDK Regeneration**

```bash
buf generate
ameide primitive verify --check sdk-freshness --json
```

**Step 4.3: Implementation (TDD Loop)**

The agent follows **strict TDD discipline**: RED → GREEN → REFACTOR.

```bash
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
ameide primitive verify --kind domain --name orders --test orders.cancel_order.success --json
# Expected: {"status": "FAIL", "reason": "CancelOrder not implemented"}

# Step 2: GREEN - Implement minimal code
# Agent implements CancelOrder handler
ameide primitive verify --kind domain --name orders --test orders.cancel_order.success --json
# Expected: {"status": "PASS"}

# Step 3: REFACTOR (if needed)
# Step 4: NEXT - Move to next test
```

**Step 4.4: Cascade Verification**

After the primitive is green, verify all consumers:

```bash
ameide primitive verify --all --proto-path orders/v1/orders.proto --json
```

### 5.6 Phase 5: Human UAT

Before merge, human verifies implementation meets requirements. Agent presents final summary:

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

---

## 6. Modifying Existing Protos (Cascading Effects)

When an AI agent modifies an **existing** proto (not creating new), the workflow must detect and verify all consuming services.

### 6.1 Impact Analysis

Before modifying a proto:

```bash
ameide primitive impact --proto-path packages/ameide_core_proto/src/ameide_core_proto/orders/v1/orders.proto --json
```

**Output:**
```json
{
  "proto_path": "ameide_core_proto/orders/v1/orders.proto",
  "consumers": [
    {"kind": "DOMAIN", "name": "orders", "path": "primitives/domain/orders", "imports": ["OrdersService", "Order"]},
    {"kind": "PROCESS", "name": "l2o", "path": "primitives/process/l2o", "imports": ["OrdersServiceClient"]},
    {"kind": "UISURFACE", "name": "www_platform", "path": "primitives/uisurface/www_platform", "imports": ["Order"]}
  ],
  "sdks_affected": ["ameide-sdk-go", "ameide-sdk-ts", "ameide-sdk-python"]
}
```

### 6.2 Breaking Change Detection

```bash
buf breaking packages/ameide_core_proto --against .git#branch=main
```

If breaking changes detected, `ameide primitive verify` will fail unless:
- The change is intentional AND
- All consumers are updated in the same PR

### 6.3 Cascade Verification

```bash
buf generate                          # Regenerates all SDKs
ameide primitive verify --all --json  # Verify ALL affected primitives
```

**Output:**
```json
{
  "summary": "fail",
  "proto_change": {
    "path": "ameide_core_proto/orders/v1/orders.proto",
    "breaking": false,
    "changes": ["added field Order.priority", "added rpc CancelOrder"]
  },
  "consumer_results": [
    {"name": "orders", "kind": "DOMAIN", "status": "PASS", "tests": 12},
    {"name": "l2o", "kind": "PROCESS", "status": "FAIL", "tests": 8, "failed": 2},
    {"name": "www_platform", "kind": "UISURFACE", "status": "PASS", "tests": 5}
  ],
  "next_steps": ["Fix l2o process to handle CancelOrder"]
}
```

### 6.4 Proto Modification Flow

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

---

## 7. Scaffolded Output per Primitive Kind

### 7.1 Domain Primitive

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

### 7.2 Process Primitive

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
```

### 7.3 Agent Primitive

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
```

### 7.4 UISurface Primitive

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
```

---

## 8. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [470-ameide-vision §8-13](470-ameide-vision.md) | EDA core invariants |
| [472-ameide-information-application §2.8.6, §3.3](472-ameide-information-application.md) | CQRS, outbox, idempotency patterns |
| [476-ameide-security-trust](476-ameide-security-trust.md) | Security verification hooks |
| [477-primitive-stack §5.2](477-primitive-stack.md) | Human-gated agentic workflow |
| [496-eda-principles](496-eda-principles.md) | Full EDA reference |
