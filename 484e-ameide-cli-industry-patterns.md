# 484e – Ameide CLI: Industry Patterns & Rationale

**Status:** Active
**Audience:** CLI implementers, platform architects
**Scope:** Vendor alignment, what we do vs don't do, anti-patterns

> **Parent document**: [484 – Ameide CLI Overview](484-ameide-cli.md)

---

## 1. Industry Tool Comparison

The Ameide CLI draws inspiration from several mature tools while avoiding their pitfalls:

| Tool | What We Take | What We Avoid |
|------|--------------|---------------|
| **Backstage** | Service catalog metadata, `catalog-info.yaml` | UI scaffolding, heavy plugin system |
| **Buf/Connect** | Proto-first, breaking change detection, generated SDKs | N/A (we use Buf directly) |
| **Kubebuilder** | CRD scaffolding, operator patterns | Full controller generation, opinionated runtime |
| **JHipster** | Nothing | Full-stack generation, framework lock-in |
| **Nx/Turborepo** | Monorepo task orchestration, affected analysis | Build caching (handled by Tilt/CI) |

---

## 2. Design Rationale: Generate Wiring, Never Meaning

### 2.1 Core Philosophy

The CLI generates **mechanical structure** (wiring) but never **business logic** (meaning):

```
┌─────────────────────────────────────────────────────────────────┐
│                    What CLI Generates (Wiring)                  │
├─────────────────────────────────────────────────────────────────┤
│ • Directory layout per primitive kind                           │
│ • Empty handler stubs returning codes.Unimplemented             │
│ • Test files that call handlers and fail                        │
│ • Dockerfile.dev, Dockerfile.release                            │
│ • go.mod, pyproject.toml, package.json                          │
│ • GitOps manifests (values.yaml, component.yaml)                │
│ • catalog-info.yaml for service discovery                       │
│ • Health probe endpoints (unimplemented)                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│               What Agents Write (Meaning)                       │
├─────────────────────────────────────────────────────────────────┤
│ • Business logic in handlers                                     │
│ • Validation rules and domain invariants                        │
│ • Test assertions and fixtures                                  │
│ • Proto message definitions                                     │
│ • Event payloads and command structures                         │
│ • Database schemas and migrations                               │
│ • Integration with other primitives                             │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Why Not Generate Business Logic?

JHipster and similar tools generate full applications including:
- Entity relationships
- CRUD operations
- Form validation
- UI components

**Problems with this approach:**

1. **Regeneration hell** – Change the schema, regenerate, lose customizations
2. **Framework lock-in** – Generated code assumes specific patterns
3. **Hidden complexity** – Generated code is hard to understand and modify
4. **Stale patterns** – Generated patterns lag behind best practices
5. **AI displacement** – AI agents are better at writing business logic than templates

**Ameide approach:**

1. **One-shot scaffolding** – Generate once, never regenerate
2. **Framework-agnostic** – Generated stubs work with any Go/TS/Python patterns
3. **Transparent structure** – Generated code is minimal and obvious
4. **Agent-native** – AI writes business logic using current best practices
5. **TDD-aligned** – Scaffolded tests fail until agent implements handlers

---

## 3. Alignment with Industry Tools

### 3.1 Backstage Alignment

We adopt Backstage's service catalog conventions:

```yaml
# catalog-info.yaml (scaffolded)
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: orders
  annotations:
    ameide.io/primitive-kind: domain
spec:
  type: service
  lifecycle: production
  owner: platform-team
```

**What we use:**
- `catalog-info.yaml` for service metadata
- Component relationships for dependency tracking
- Annotations for Ameide-specific metadata

**What we skip:**
- Backstage UI (we use CLI + agent workflows)
- Backstage plugins (unnecessary complexity)
- TechDocs generation (agents write docs)

### 3.2 Buf/Connect Alignment

We use Buf directly for proto management:

```bash
# Buf handles proto, CLI handles Go/TS code
buf lint                         # Proto linting
buf generate                     # SDK generation
buf breaking --against           # Breaking change detection

# CLI consumes Buf output
ameide primitive describe --json # Shows proto freshness
ameide primitive drift --json    # Shows SDK staleness
ameide primitive verify --json   # Runs buf breaking
```

**Key insight:** We don't wrap Buf – we integrate with it. The CLI reads Buf's output and correlates it with repo state.

### 3.3 Kubebuilder Alignment

We adopt Kubebuilder's CRD and operator patterns:

```bash
# Kubebuilder-style CRD scaffolding
kubebuilder create api --group ameide.io --version v1 --kind Domain

# Ameide equivalent (but for primitives, not operators)
ameide primitive scaffold --kind domain --name orders --json
```

**What we use:**
- CRD schema patterns (`config/crd/bases/`)
- Operator reconciliation loop structure
- Controller-runtime patterns

**What we skip:**
- Full operator code generation (operators are special, manually written)
- kubebuilder markers (too magical)
- Admission webhooks (handled by operators)

### 3.4 Nx/Turborepo Alignment

We adopt monorepo task orchestration patterns:

```bash
# Nx-style affected analysis
nx affected:test                 # Run tests for affected projects

# Ameide equivalent
ameide primitive impact --json   # Show what's affected by proto change
ameide primitive verify --all --json  # Cascade verification
```

**What we use:**
- Affected file analysis (via git + proto imports)
- Task dependency graphs (proto → SDK → consumers)
- Parallel test execution

**What we skip:**
- Build caching (Tilt handles dev, CI handles prod)
- Task scheduling (CI/CD pipelines handle this)
- Project graph visualization (not needed for CLI)

---

## 4. Anti-Patterns to Avoid

### 4.1 JHipster Syndrome

**Anti-pattern:** Generate entire applications from entity definitions.

```bash
# DON'T: JHipster-style generation
jhipster entity Order --fields name,total,status --relationships customer:Customer

# Generated: controllers, services, repositories, DTOs, tests, UI, migrations...
# Problem: Now you have 50 files of generated code to maintain
```

**Ameide approach:**

```bash
# DO: Generate structure, let agent write logic
ameide primitive scaffold --kind domain --name orders --json

# Generated: directory structure, empty handlers, failing tests
# Agent: Writes business logic to make tests pass
```

### 4.2 Magic Comments

**Anti-pattern:** Use comments to control code generation.

```go
// DON'T: kubebuilder-style markers
// +kubebuilder:object:generate=true
// +kubebuilder:subresource:status
type Order struct {
    // ...
}
```

**Ameide approach:**

```go
// DO: Explicit proto-driven structure
// Proto defines the shape, Go code is just implementation
type OrdersServer struct {
    pb.UnimplementedOrdersServiceServer
    // ... dependencies
}
```

### 4.3 Regeneration Workflows

**Anti-pattern:** Regularly regenerate code and merge changes.

```bash
# DON'T: Continuous regeneration
jhipster entity Order --regenerate
git diff  # 500 lines of generated changes to review
```

**Ameide approach:**

```bash
# DO: One-shot scaffold, manual evolution
ameide primitive scaffold --kind domain --name orders --json
# If folder exists, scaffold REFUSES to run
# Evolution is handled by agents editing existing code
```

### 4.4 Hidden Abstractions

**Anti-pattern:** Generate code that depends on framework internals.

```typescript
// DON'T: Generated code with hidden dependencies
@Entity()
@ValidateIf((o) => o.status !== 'DRAFT')
export class Order extends BaseEntity {
    // Framework magic everywhere
}
```

**Ameide approach:**

```go
// DO: Plain Go code, explicit dependencies
type OrdersServer struct {
    db      *sql.DB
    outbox  outbox.Writer
    logger  *slog.Logger
}
```

---

## 5. Per-Primitive Scaffold Outputs

### 5.1 Domain Primitive

```
primitives/domain/{name}/
├── cmd/
│   └── main.go                  # Entry point with DI wiring
├── internal/
│   ├── server/
│   │   └── server.go            # gRPC server (handlers return Unimplemented)
│   └── domain/
│       └── {name}.go            # Domain types (empty, agent fills in)
├── __tests__/
│   ├── unit/
│   └── integration/
│       ├── run_integration_tests.sh
│       └── {name}_test.go       # Tests that CALL handlers and FAIL
├── Dockerfile.dev
├── Dockerfile.release
├── go.mod
├── README.md
└── catalog-info.yaml
```

### 5.2 Process Primitive

```
primitives/process/{name}/
├── cmd/
│   └── main.go
├── internal/
│   ├── workflows/
│   │   └── {name}.go            # Temporal workflow (returns Unimplemented)
│   └── activities/
│       └── {name}.go            # Temporal activities (empty)
├── __tests__/
│   └── integration/
│       └── {name}_test.go       # Temporal test framework tests (FAIL)
├── Dockerfile.dev
├── Dockerfile.release
├── go.mod
├── README.md
└── catalog-info.yaml
```

### 5.3 Agent Primitive

```
primitives/agent/{name}/
├── src/
│   ├── main.py                  # Entry point
│   ├── agent.py                 # Agent class (returns NotImplementedError)
│   └── tools/
│       └── __init__.py          # MCP tools (empty)
├── __tests__/
│   └── test_{name}.py           # Tests that FAIL
├── Dockerfile.dev
├── Dockerfile.release
├── pyproject.toml
├── README.md
└── catalog-info.yaml
```

### 5.4 UISurface Primitive

```
primitives/uisurface/{name}/
├── src/
│   ├── app/
│   │   └── page.tsx             # Next.js pages (placeholder)
│   └── components/
│       └── .gitkeep
├── __tests__/
│   └── {name}.test.tsx          # Tests that FAIL
├── Dockerfile.dev
├── Dockerfile.release
├── next.config.ts
├── package.json
├── README.md
└── catalog-info.yaml
```

---

## 6. Scaffolding Decision Matrix

| Scenario | Scaffold? | Rationale |
|----------|-----------|-----------|
| New Domain primitive | ✅ Yes | Mechanical structure is well-defined |
| New RPC in existing Domain | ❌ No | Agent adds handler method to existing file |
| New event type | ❌ No | Agent adds to proto, regenerates SDKs |
| New test case | ❌ No | Agent writes test in existing test file |
| New environment config | ❌ No | Agent creates values file in gitops |
| New CRD instance | ✅ Optional | `--include-gitops` flag adds manifest |
| Database migration | ❌ No | Agent writes migration SQL |
| Operator for new CRD kind | ❌ No | Operators are special, manually crafted |

---

## 7. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [365-buf-sdks-v2](365-buf-sdks-v2.md) | Buf integration philosophy |
| [477-primitive-stack](477-primitive-stack.md) | Primitive structure this aligns with |
| [482-adding-new-service](482-adding-new-service.md) | Manual checklist that scaffold automates |
| [495-ameide-operators](495-ameide-operators.md) | Why operators are NOT scaffolded |
