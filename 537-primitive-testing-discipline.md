# 537 – Primitive Testing Discipline

**Status:** Normative (platform gate)
**Audience:** AI agents (primary), developers, CLI implementers
**Scope:** Testing requirements, RED→GREEN TDD pattern, CI enforcement, per-primitive invariants

---

## Purpose

This document is the **normative testing discipline** for all primitive types (Domain, Process, Projection, Integration, UISurface, Agent). It defines mandatory requirements that CI enforces:

1. The RED→GREEN TDD pattern scaffolded into all primitives
2. Per-primitive invariants that tests must verify
3. CI enforcement strategy (tests from day one, no exceptions)
4. Test infrastructure patterns per primitive type

**Non-negotiable:** All primitives must have tests. "No tests found" is a **FAIL**, not a warning.

---

## Cross-References

- [510-domain-primitive-scaffolding.md](510-domain-primitive-scaffolding.md) – Domain scaffold shape (§4 handler/test semantics)
- [511-process-primitive-scaffolding.md](511-process-primitive-scaffolding.md) – Process scaffold shape (§4 workflow/test semantics)
- [512-agent-primitive-scaffolding.md](512-agent-primitive-scaffolding.md) – Agent scaffold shape (§3.4 LangGraph discipline, §4 test semantics)
- [520-primitives-stack-v2.md](520-primitives-stack-v2.md) – Guardrails plane (§CI gate checklist)
- [533-capability-implementation-playbook.md](533-capability-implementation-playbook.md) – Node 8 (Verify & Package)
- [520-primitives-stack-v2-research-agent.md](520-primitives-stack-v2-research-agent.md) – LangGraph invariants research

---

## 1. RED→GREEN TDD Pattern

All primitive scaffolds generate **intentionally failing tests** (RED). This ensures:

- Implementers must replace placeholders with real logic (GREEN)
- Tests exist from day one, not as an afterthought
- CI catches unimplemented primitives

### Universal Scaffold Marker

All scaffold tests include the string `AMEIDE_SCAFFOLD` in their failure message. This marker is language-agnostic and scanned by `ameide primitive verify` (scaffold markers fail verification by default).

**Go (Domain / Process / Projection / Integration):**

```go
func TestExampleRPC(t *testing.T) {
    // ... test setup ...
    t.Fatalf("AMEIDE_SCAFFOLD: replace with real assertions (RED→GREEN→REFACTOR)")
}
```

**Python (Agent):**

```python
def test_agent_invoke():
    """AMEIDE_SCAFFOLD: replace with real assertions."""
    agent = AgentPrimitive()
    with pytest.raises(NotImplementedError):
        agent.run("test", thread_id="t/test/1")
```

**TypeScript/Node (UISurface):**

```typescript
test("server responds to /healthz", async () => {
  throw new Error("AMEIDE_SCAFFOLD: replace with real assertions (RED→GREEN→REFACTOR).");
});
```

**Important:** Do NOT use Go build tags (`//go:build scaffold`) as markers. Build tags change compilation behavior and can hide scaffold tests from normal `go test` runs, which defeats the purpose.

### TDD Lifecycle

1. **Scaffold** → Tests are generated in RED state with `AMEIDE_SCAFFOLD` marker
2. **Implement** → Replace scaffold failures with real assertions, remove marker
3. **Verify** → `ameide primitive verify --check tests` passes
4. **Refactor** → Improve implementation while keeping tests green

---

## 2. Per-Primitive Test Invariants

Each primitive type has specific invariants that tests must verify. These go beyond "does the code run?" to enforce architectural discipline.

### 2.1 Domain (Go)

| Invariant | Test Pattern | Location |
|-----------|--------------|----------|
| Handler returns correct response | Unit test per RPC | `internal/tests/<rpc>_test.go` |
| Outbox receives correct events | Mock `EventOutbox`, assert `Insert()` calls | `internal/tests/<rpc>_test.go` |
| Event envelope metadata complete | Assert `OutboxEvent` has tenant_id, idempotency key, trace context | `internal/tests/<rpc>_test.go` |
| No broker imports in handlers | Import policy check (verify command) | CI |
| Aggregate version tracking | Assert version increments on state changes | `internal/tests/<aggregate>_test.go` |

**Required Test Files:**
- `internal/tests/<rpc>_test.go` – Per-RPC handler tests
- `internal/tests/integration_mode.go` – Test harness mode switch

**Cross-Cutting Envelope Tests (Required):**
- Tenant scope present on emitted facts
- Idempotency key is set and dedupe works
- W3C trace context (`traceparent`/`tracestate`) present
- Correlation/causation IDs propagated

### 2.2 Process (Go / Temporal)

| Invariant | Test Pattern | Location |
|-----------|--------------|----------|
| **Static determinism policy** | AST/import guard: no banned imports/calls in workflow code | `internal/tests/determinism_policy_test.go` |
| **Behavioral idempotency** | Temporal testsuite: same input → same result | `internal/tests/<workflow>_workflow_test.go` |
| Replay determinism (optional) | Replay recorded history, assert no non-determinism error | `internal/tests/<workflow>_replay_test.go` |
| Activity idempotency | Call activity twice with same key, assert same result | `internal/tests/<activity>_test.go` |
| Signal handling | Send signal, assert state change | `internal/tests/<workflow>_workflow_test.go` |
| `WorkflowIDReusePolicy` explicit | Assert policy is set (not default) | `internal/tests/ingress_test.go` |
| Continue-As-New threshold | Assert CAN triggers after N events | `internal/tests/<workflow>_workflow_test.go` |
| Version tracking | Assert `lastSeenAggregateVersion` increments | `internal/tests/<workflow>_workflow_test.go` |
| **Process emits intents, not domain facts** | Assert domain changes go through intent emission | `internal/tests/<workflow>_workflow_test.go` |

**Static Determinism Policy Test (Required):**

Process workflows must not import or call non-deterministic APIs. Generate a static test that fails if workflow code:
- Imports `time`, `math/rand`, `os`, `net/http` (or equivalents)
- Calls `time.Now()`, `rand.*`, file I/O, network calls directly

```go
func TestWorkflowDeterminismPolicy(t *testing.T) {
    // Scan workflow package for banned imports/calls
    violations := checkDeterminismPolicy("./internal/workflows")
    require.Empty(t, violations, "Workflow code contains non-deterministic calls: %v", violations)
}
```

**Behavioral Idempotency Test (Required):**

```go
func TestWorkflowIdempotency(t *testing.T) {
    env := testsuite.NewTestWorkflowEnvironment()
    input := WorkflowInput{...}

    // First execution
    env.ExecuteWorkflow(ExampleWorkflow, input)
    result1 := env.GetWorkflowResult()

    // Reset and re-execute with same input
    env2 := testsuite.NewTestWorkflowEnvironment()
    env2.ExecuteWorkflow(ExampleWorkflow, input)
    result2 := env2.GetWorkflowResult()

    assert.Equal(t, result1, result2, "Workflow is not idempotent")
}
```

**Replay Tests (Optional/Bonus):**
Replay tests are useful but brittle (require stable recorded histories). Use them as additional coverage, not as the only determinism gate.

**Required Test Files:**
- `internal/tests/<workflow>_workflow_test.go` – Workflow tests using Temporal test env
- `internal/tests/determinism_policy_test.go` – Static determinism policy check
- `internal/tests/process_workflow_test.go` – Generic workflow scaffold test

### 2.3 Projection (Go)

| Invariant | Test Pattern | Location |
|-----------|--------------|----------|
| Event handler processes events | Send event, assert state change | `internal/tests/<handler>_test.go` |
| Idempotency key enforced | Process same event twice, assert no duplicate | `internal/tests/<handler>_test.go` |
| Offset/checkpoint persists | Process event, restart, assert offset preserved | `internal/tests/checkpoint_test.go` |
| Schema migration applies | Run migration, assert schema exists | `internal/tests/migration_test.go` |

**Required Test Files:**
- `internal/tests/<handler>_test.go` – Event handler tests
- `internal/tests/checkpoint_test.go` – Checkpoint store tests
- `internal/tests/integration_mode.go` – Mode switch

### 2.4 Integration (Go / NiFi)

| Invariant | Test Pattern | Location |
|-----------|--------------|----------|
| Port contract matches proto | Validate I/O schemas against proto | `internal/tests/<port>_test.go` |
| Parameter placeholders only | Assert no secrets in generated artifacts | `internal/tests/artifact_test.go` |
| Flow artifact valid | Validate NiFi flow JSON structure | `internal/tests/flow_test.go` |
| **Produces intents or integration facts, not domain facts** | Assert output types are intents or integration-scoped facts | `internal/tests/<port>_test.go` |

**Important:** Integrations translate external stimuli into **intents** (to Domains/Processes) or emit **integration facts** (ingress/outbound telemetry). Integrations do NOT emit domain facts—domains emit domain facts after accepting intents.

**Required Test Files:**
- `internal/tests/<port>_test.go` – Port contract tests
- `internal/tests/artifact_test.go` – Generated artifact validation

### 2.5 UISurface (TypeScript / Node)

| Invariant | Test Pattern | Location |
|-----------|--------------|----------|
| Server responds to health | GET /healthz returns 200 | `tests/server.test.ts` |
| Routes match proto contract | Validate route paths against proto | `tests/routes.test.ts` |
| SDK client wiring correct | Mock SDK, assert calls | `tests/sdk.test.ts` |

**Required Test Files:**
- `tests/server.test.ts` – Server lifecycle tests
- `tests/*.test.ts` – Route/component tests

### 2.6 Agent (Python / LangGraph)

#### Mechanical Invariants (LangGraph)

| Invariant | Test Pattern | Location |
|-----------|--------------|----------|
| `thread_id` required when persistence enabled | Invoke without thread_id, assert error | `tests/test_thread_id.py` |
| Checkpointer integration | Invoke twice, assert state persists | `tests/test_checkpointer.py` |
| Reducer annotations on state fields | Introspect state class, assert reducers | `tests/test_state.py` |
| Parallel writes merge correctly | Two nodes write, assert both values present | `tests/test_state.py` |
| Idempotent side-effects | Call with same dedupe key twice, assert single effect | `tests/test_idempotency.py` |
| Stream mode is "updates" | Capture config, assert `stream_mode="updates"` | `tests/test_stream_mode.py` |
| Tool output validated | Call tool, validate against schema | `tests/test_tools.py` |

#### Governance/Security Invariants (Required)

| Invariant | Test Pattern | Location |
|-----------|--------------|----------|
| **Tool grants enforcement** | Agent cannot call tools outside declared grants | `tests/test_tool_grants.py` |
| **Risk-tier behavior** | High-risk tool paths require explicit approval/steps | `tests/test_risk_tiers.py` |
| **Agent state is not business truth** | All durable changes go through domain/process APIs | `tests/test_state_discipline.py` |

**Tool Grants Test (Required):**

```python
def test_agent_cannot_call_undeclared_tools():
    """Agent must only call tools in its declared grants."""
    agent = create_agent(
        granted_tools=["search", "read_file"],
        available_tools=["search", "read_file", "write_file", "execute_code"]
    )

    # Should succeed
    result = agent.invoke({"messages": [HumanMessage(content="search for X")]})
    assert "search" in [call.tool for call in result.tool_calls]

    # Should fail or be blocked
    with pytest.raises(ToolNotGrantedError):
        agent.invoke({"messages": [HumanMessage(content="execute dangerous code")]})
```

**State Discipline Test (Required):**

```python
def test_agent_state_not_business_truth():
    """Agent must emit intents for durable changes, not store them in state."""
    agent = create_agent()
    result = agent.invoke({"messages": [HumanMessage(content="create order")]})

    # Agent state should NOT contain the order
    assert "order" not in result.state

    # Agent should have emitted an intent
    assert any(
        intent.type == "CreateOrderIntent"
        for intent in result.emitted_intents
    )
```

**Required Test Files:**
- `tests/test_agent.py` – Agent entrypoint tests (RED by default)
- `tests/test_state.py` – State/reducer tests
- `tests/test_thread_id.py` – Persistence discipline tests
- `tests/test_idempotency.py` – Side-effect idempotency tests
- `tests/test_tool_grants.py` – Tool access control tests
- `tests/test_risk_tiers.py` – Risk tier enforcement tests
- `tests/test_state_discipline.py` – State vs business truth tests

**LangGraph Test Infrastructure:**
- Use `langgraph.checkpoint.memory.MemorySaver` for unit tests
- Use `GenericFakeChatModel` for deterministic LLM responses
- Use `_StubTool` pattern for tool invocation capture

---

## 3. CI Enforcement Strategy

### 3.1 Non-Negotiable Rules

1. **"No tests found" = FAIL.** Scaffolds always generate tests. If a primitive has no tests, something is wrong.
2. **Scaffold markers must be replaced.** Any `AMEIDE_SCAFFOLD` marker in a test file causes `ameide primitive verify` to fail.
3. **CI runs the same commands as `verify`.** The CLI's `verify --check tests` runs the same test commands CI runs—it's a wrapper, not a competing gate.

### 3.2 Test Mode Semantics

Three modes, clearly defined:

| Mode | Env Var | Infrastructure | Use Case |
|------|---------|----------------|----------|
| `repo` | `INTEGRATION_MODE=repo` | Pure mocks only | Fast unit tests, no I/O |
| `local` | `INTEGRATION_MODE=local` | testcontainers (Postgres, Temporal, etc.) | Integration tests with local infra |
| `cluster` | `INTEGRATION_MODE=cluster` | Real cluster dependencies | E2E/probe tests, PostSync smoke jobs |

**Important:** testcontainers = `local` mode, NOT `cluster` mode. `cluster` is reserved for tests against real deployed infrastructure.

### 3.3 Verify Behavior

```bash
ameide primitive verify --kind domain --name orders --check tests
```

| Condition | Result |
|-----------|--------|
| All tests pass | **PASS** |
| Tests exist but some fail | **FAIL** |
| No tests found | **FAIL** (not WARN) |
| `AMEIDE_SCAFFOLD` marker present | **FAIL** |

### 3.4 CI Workflow Integration

```yaml
- name: Verify Primitive Tests
  run: |
    for kind in domain process projection integration uisurface agent; do
      for name in $(ls primitives/$kind/ 2>/dev/null); do
        ameide primitive verify --kind $kind --name $name --check tests
      done
    done
```

---

## 4. Test Infrastructure by Primitive Type

### 4.1 Domain / Projection / Integration (Go)

**Unit Test Setup (`repo` mode):**
```go
func TestSetup(t *testing.T) {
    // Create mock dependencies
    mockOutbox := &MockEventOutbox{}
    mockDB := &MockDB{}

    // Create handler with mocks
    handler := handlers.New(mockOutbox, mockDB)

    // Test
    resp, err := handler.ExampleRPC(ctx, req)
    assert.NoError(t, err)
    assert.Equal(t, expected, resp)

    // Verify mock interactions
    assert.Len(t, mockOutbox.InsertCalls, 1)

    // Verify envelope metadata
    event := mockOutbox.InsertCalls[0]
    assert.NotEmpty(t, event.TenantID, "tenant_id required")
    assert.NotEmpty(t, event.IdempotencyKey, "idempotency_key required")
    assert.NotEmpty(t, event.TraceParent, "traceparent required")
}
```

**Integration Test Setup (`local` mode with testcontainers):**
```go
func TestIntegration(t *testing.T) {
    if os.Getenv("INTEGRATION_MODE") != "local" {
        t.Skip("Skipping integration test (requires INTEGRATION_MODE=local)")
    }

    ctx := context.Background()

    // Start Postgres container
    pgContainer, err := postgres.RunContainer(ctx)
    require.NoError(t, err)
    defer pgContainer.Terminate(ctx)

    // Run migrations and test with real DB
    // ...
}
```

### 4.2 Process (Go / Temporal)

**Workflow Test Setup (`repo` mode):**
```go
func TestWorkflow(t *testing.T) {
    suite := testsuite.NewTestSuite(t)
    env := suite.NewTestWorkflowEnvironment()

    // Mock activities
    env.OnActivity(activities.EmitFact, mock.Anything, mock.Anything).Return(nil)

    // Execute workflow
    env.ExecuteWorkflow(workflows.ExampleWorkflow, input)

    // Assert completion
    require.True(t, env.IsWorkflowCompleted())
    require.NoError(t, env.GetWorkflowError())
}
```

**Static Determinism Policy Test (Required):**
```go
func TestWorkflowDeterminismPolicy(t *testing.T) {
    bannedImports := []string{"time", "math/rand", "os", "net/http", "io"}
    bannedCalls := []string{"time.Now", "rand.Int", "os.ReadFile", "http.Get"}

    violations := scanPackageForViolations(
        "./internal/workflows",
        bannedImports,
        bannedCalls,
    )

    require.Empty(t, violations, "Workflow code violates determinism policy:\n%s", strings.Join(violations, "\n"))
}
```

**Integration Test Setup (`local` mode with testcontainers):**
```go
func TestWorkflowIntegration(t *testing.T) {
    if os.Getenv("INTEGRATION_MODE") != "local" {
        t.Skip("Skipping integration test (requires INTEGRATION_MODE=local)")
    }

    // Start Temporal container
    temporalContainer, err := temporal.RunContainer(ctx)
    require.NoError(t, err)
    defer temporalContainer.Terminate(ctx)

    // Test with real Temporal
    // ...
}
```

### 4.3 Agent (Python / LangGraph)

**Unit Test Setup:**
```python
import pytest
from unittest.mock import MagicMock
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.language_models.fake import GenericFakeChatModel

@pytest.fixture
def agent_with_memory():
    checkpointer = MemorySaver()
    fake_llm = GenericFakeChatModel(messages=[
        AIMessage(content="Test response")
    ])
    return create_agent(llm=fake_llm, checkpointer=checkpointer)

async def test_thread_id_required(agent_with_memory):
    with pytest.raises(ValueError, match="thread_id is required"):
        await agent_with_memory.ainvoke(
            {"messages": [HumanMessage(content="hello")]},
            config={}  # No thread_id
        )
```

**Tool Grants Test:**
```python
def test_tool_grants_enforced():
    """Agent must respect tool grants."""
    stub_granted = _StubTool(name="search")
    stub_denied = _StubTool(name="execute_code")

    agent = create_agent(
        granted_tools=[stub_granted],
        # execute_code is available but not granted
    )

    # Attempt to use denied tool should fail
    with pytest.raises(ToolNotGrantedError):
        agent.invoke({
            "messages": [HumanMessage(content="execute some code")]
        })
```

**State Reducer Test:**
```python
def test_messages_field_has_add_messages_reducer():
    from typing import get_type_hints, Annotated
    from myagent.state import AgentState

    hints = get_type_hints(AgentState, include_extras=True)
    messages_hint = hints.get("messages")

    # Check it's Annotated with add_messages
    assert hasattr(messages_hint, "__metadata__")
    reducer = messages_hint.__metadata__[0]
    assert reducer.__name__ == "add_messages"
```

### 4.4 UISurface (TypeScript / Node)

**Server Test Setup:**
```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { createServer } from "../src/server";

describe("Server", () => {
  let server: Server;

  beforeAll(async () => {
    server = await createServer({ port: 0 });
  });

  afterAll(async () => {
    await server.close();
  });

  it("responds to /healthz", async () => {
    const res = await fetch(`http://localhost:${server.port}/healthz`);
    expect(res.status).toBe(200);
  });
});
```

---

## 5. Verification Command Integration

The `ameide primitive verify` command includes testing as a first-class check:

```bash
# Verify all checks including tests
ameide primitive verify --kind domain --name orders

# Verify only tests
ameide primitive verify --kind domain --name orders --check tests

# JSON output for CI
ameide primitive verify --kind agent --name scribe --check tests --json
```

**Verify Output Example:**
```json
{
  "kind": "agent",
  "name": "scribe",
  "checks": {
    "tests": {
      "status": "FAIL",
      "details": "2 tests passing, 1 test with AMEIDE_SCAFFOLD marker",
      "scaffold_tests": ["tests/test_agent.py::test_agent_invoke"]
    }
  }
}
```

---

## 6. Cross-Cutting Envelope Tests

All primitives that emit facts/events must include envelope metadata tests:

| Field | Required For | Test Pattern |
|-------|--------------|--------------|
| `tenant_id` | All facts | Assert non-empty on every emitted fact |
| `idempotency_key` | All facts | Assert set, assert dedupe works |
| `traceparent` | All facts (W3C Trace Context) | Assert valid format |
| `tracestate` | When propagating | Assert preserved from incoming context |
| `correlation_id` | Cross-primitive flows | Assert set and propagated |
| `causation_id` | Derived facts | Assert links to source event |

**Example (Domain):**
```go
func TestEventEnvelopeMetadata(t *testing.T) {
    mockOutbox := &MockEventOutbox{}
    handler := handlers.New(mockOutbox)

    ctx := context.WithValue(context.Background(), "tenant_id", "tenant-123")
    ctx = context.WithValue(ctx, "traceparent", "00-abc123-def456-01")

    handler.CreateOrder(ctx, &CreateOrderRequest{...})

    require.Len(t, mockOutbox.InsertCalls, 1)
    event := mockOutbox.InsertCalls[0]

    assert.Equal(t, "tenant-123", event.Metadata.TenantID)
    assert.NotEmpty(t, event.Metadata.IdempotencyKey)
    assert.Regexp(t, `^\d{2}-[a-f0-9]{32}-[a-f0-9]{16}-\d{2}$`, event.Metadata.TraceParent)
}
```

---

## 7. Implementation Status

### Implemented
- RED scaffold tests in all primitive scaffolds (510, 511, 512, 513, 514)
- `AMEIDE_SCAFFOLD` universal marker in scaffold test failures
- `go test` / `pytest` / `npm test` in CI for primitives
- `ameide primitive verify --check/--checks tests` runs language-appropriate tests and fails when no tests exist
- Scaffold markers fail `ameide primitive verify` by default (any test file containing `AMEIDE_SCAFFOLD`)
- `Codegen` drift gate (fails if generated outputs are stale)

### Pending
- Static determinism policy tests in Process scaffold
- LangGraph invariant tests in Agent scaffold
- Tool grants and risk-tier tests in Agent scaffold
- Cross-cutting envelope tests in Domain/Process scaffolds

---

## 8. Summary: Test Requirements by Primitive

| Primitive | Required Tests | Key Invariants |
|-----------|----------------|----------------|
| **Domain** | Handler, outbox, aggregate, envelope | Event emission, no broker imports, envelope metadata |
| **Process** | Workflow, activity, ingress, **determinism policy** | Static determinism, behavioral idempotency, intents not domain facts |
| **Projection** | Handler, checkpoint, migration | Idempotency, offset tracking |
| **Integration** | Port, artifact | Proto contract, no secrets, intents or integration facts only |
| **UISurface** | Server, routes | Health endpoint, SDK wiring |
| **Agent** | State, thread_id, tools, idempotency, **grants**, **risk-tier**, **state discipline** | LangGraph mechanics, governance, security |

---

## 9. CLI Scaffolding & Verification: Gap Analysis

This section documents gaps between this spec (537) and the current CLI implementation in `packages/ameide_core_cli`, along with required changes.

### 9.1 Current Implementation Summary

**Scaffold Entry Point:** `packages/ameide_core_cli/internal/commands/primitive_scaffold.go`

**Verify Entry Point:** `packages/ameide_core_cli/internal/commands/primitive.go`

**What's Working Well:**
- Template-driven scaffolding for all 6 primitive types
- Language enforcement (Go for Domain/Process, Python for Agent, Node for UISurface)
- Proto-driven handler generation with RPC stubs
- EDA pattern baked into Domain scaffolds (outbox/dispatcher/migrations)
- RED test pattern scaffolded with failure messages
- `go.work` auto-wiring for Go modules
- GitOps manifest generation (`--include-gitops`)
- Test harness script generation (`--include-test-harness`)

**Current Scaffold Test Patterns:**

| Language | Pattern | Current Message |
|----------|---------|-----------------|
| Go | `t.Fatalf(...)` | `"AMEIDE_SCAFFOLD: ... (RED→GREEN→REFACTOR)"` |
| Python | `pytest.fail(...)` | `"AMEIDE_SCAFFOLD: ... (RED→GREEN→REFACTOR)."` |
| TypeScript/Node | `throw new Error(...)` | `"AMEIDE_SCAFFOLD: ... (RED→GREEN→REFACTOR)."` |

**Current Verification Checks Implemented:**

| Check | Scope | Implementation |
|-------|-------|----------------|
| `WorkloadReady` | Cluster | Deployment replicas ready |
| `ConditionsHealthy` | Cluster | CRD conditions (Ready=True) |
| `DBReady` | Cluster/Domain | CNPG database readiness |
| `MigrationStatus` | Cluster/Domain | Migration job status |
| `Naming` | Repo | RPC names avoid CRUD verbs |
| `Security` | Repo | Regex scan for hardcoded secrets |
| `SecretScan` | Repo | `gitleaks detect` |
| `DependencyVulns` | Repo/Go | `govulncheck` |
| `SAST` | Repo | `semgrep --config=auto` |
| `EDA` | Repo/Domain | Outbox/dispatcher/migration files exist |
| `ProcessShape` | Repo/Process | Worker/ingress/workflow/state files exist |
| `EventCoverage` | Repo | events/ directory has definitions |
| `Tests` | Repo | Runs `go test` / `pytest` / `npm test` and **fails** if no tests exist |
| `Imports` | Repo | SDK-only policy enforcement |
| `GitOps` | Repo | Component.yaml + values.yaml exist |
| `BufLint` | Repo | `buf lint` |
| `BufBreaking` | Repo | `buf breaking --against main` |
| `MCPAdapter` | Repo/Integration | MCP adapter shape validation |
| `Codegen` | Repo | Validates generated output freshness (fails if generated outputs are missing; TS via temp-tree generation+diff; Go/Python via proto vs generated-file timestamps) |

### 9.2 Gaps Identified

| ID | Gap | Current State | Required by 537 |
|----|-----|---------------|-----------------|
| **G4** | Integration mode semantics inconsistent | Both `INTEGRATION_MODE` and `INTEGRATION_TEST_MODE` are in use; not all tests consistently skip based on `repo`/`local`/`cluster` semantics | Three modes: `repo`, `local`, `cluster` |
| **G5** | No per-primitive invariant checks | Generic test check runs, but no invariant validation | Static determinism policy for Process, reducer annotation checks for Agent |
| **G6** | No cross-cutting envelope tests | No validation of event metadata | Envelope tests required (tenant_id, traceparent, idempotency_key) for Domain/Process |
| **G8** | No agent governance tests | No tool grants enforcement, no risk-tier validation | Tool grants, risk-tier, state discipline tests required |
| **G9** | Codegen verification coverage incomplete | Codegen drift check does not yet cover internal Go outputs (`packages/ameide_core_proto/gen/go`) | Verify should cover all required generated artifacts, or document which are authoritative |

### 9.3 CLI Implementation Status

**Implemented (CLI gate is real):**
- Universal `AMEIDE_SCAFFOLD` marker in scaffold test failures (Go/Python/Node).
- `ameide primitive verify --check/--checks tests` runs language-appropriate tests and **fails** when no tests exist.
- Scaffold markers fail `ameide primitive verify` by default (any test file containing `AMEIDE_SCAFFOLD`).
- `Codegen` check validates generated output freshness (TS via temp-tree generation+diff; Go/Python via proto vs generated-file timestamps).

**Still needed (to make tests “meaningful”, not just present):**
- **Integration mode discipline**: consolidate on `INTEGRATION_MODE` (keep `INTEGRATION_TEST_MODE` only as compatibility) and ensure tests consistently skip/require infra based on `repo`/`local`/`cluster`.
- **Per-primitive invariants** (tests + optional verify checks):
  - Process: determinism policy (static) + idempotency behaviors (Temporal testsuite).
  - Domain/Process: envelope metadata tests (tenant/idempotency/trace context).
  - Agent: LangGraph invariants + tool grant enforcement + risk-tier governance tests.
- **Codegen coverage**: extend drift detection to (if authoritative) internal Go outputs under `packages/ameide_core_proto/gen/go`, or document why they’re not required.

### 9.4 Implementation Priority

| Priority | Item | Effort | Impact |
|----------|------|--------|--------|
| **P0** | G4: Integration mode alignment | 2h | High - prevents “tests pass in CI, fail locally” drift |
| **P1** | G9: Codegen coverage completeness | 2–4h | High - prevents SDK/generation drift |
| **P1** | G5: Process determinism policy checks | 4h | Medium - enforces Temporal determinism discipline |
| **P1** | G8: Agent governance tests | 4h | Medium - security/governance enforcement |
| **P2** | G6: Envelope test templates | 3h | Medium - metadata discipline |

### 9.5 Critical Files Summary

| File | Changes |
|------|---------|
| `packages/ameide_core_cli/internal/commands/primitive_scaffold.go` | Scaffold test markers (`AMEIDE_SCAFFOLD`), harness prefers `INTEGRATION_MODE` |
| `packages/ameide_core_cli/internal/commands/primitive_commands.go` | `--checks` alias; integration mode via env; scaffold markers fail verification by default |
| `packages/ameide_core_cli/internal/commands/primitive.go` | Cross-language tests, “no tests found” FAIL, scaffold marker scan, `Codegen` drift check |
| `packages/ameide_core_cli/internal/commands/templates/integration/mcp_adapter/internal/mcp/http_test.go.tmpl` | RED marker test for integration MCP adapter scaffolds |
