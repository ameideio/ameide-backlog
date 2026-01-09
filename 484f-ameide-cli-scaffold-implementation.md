# 484f – Ameide CLI: Scaffold Implementation Reference

> **Deprecation notice (520):** This file describes a bespoke CLI scaffolder. That approach is deprecated. Canonical v2 uses **`buf generate`** (pinned plugins, deterministic outputs, generated-only roots, regen-diff CI gate). See `backlog/520-primitives-stack-v2.md`.

**Status:** Deprecated (superseded by 520)
**Audience:** AI agents (primary), CLI implementers (secondary)
**Scope:** Per-primitive scaffold outputs, templates, worked examples, cross-reference index

> **Parent document**: [484 – Ameide CLI Overview](484-ameide-cli-overview.md)

> **Update (2026-01): 430v2 contract**
>
> This doc is already deprecated for generation, but it also contains v1-era “integration pack / `INTEGRATION_MODE` / `run_integration_tests.sh`” assumptions. Treat `backlog/430-unified-test-infrastructure-v2-target.md` as authoritative for current test semantics.

---

## 1. Scaffold Command Reference

```bash
ameide primitive scaffold \
  --kind <domain|process|agent|uisurface> \
  --name <primitive-name> \
  --proto-path <path/to/proto.proto> \
  [flags]
```

### 1.1 Common Flags

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--kind` | Yes | — | Primitive kind: domain, process, agent, uisurface |
| `--name` | Yes | — | Primitive name (kebab-case) |
| `--proto-path` | Yes | — | Path to .proto file defining RPCs |
| `--lang` | No | go | Implementation language: go, ts, python |
| `--repo-root` | No | . | Repository root path |
| `--dry-run` | No | false | Preview files without writing |
| `--json` | No | false | Output as structured JSON |
| `--include-gitops` | No | false | Generate GitOps manifests |
| `--include-test-harness` | No | false | Generate test runner scripts |
| `--gitops-root` | No | ./gitops | Path to GitOps directory |

### 1.2 Agent-Specific Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--agent-owner` | group:platform-architecture | Backstage owner reference |
| `--agent-description` | Agent primitive for {name} | Agent description |
| `--agent-definition-ref` | — | AgentDefinition reference ID |
| `--agent-default-model` | gpt-4o-mini | Default LLM model |

---

## 2. Implementation Status by Primitive

| Kind | Languages | Templates | GitOps | Test Harness | Vertical Slice |
|------|-----------|-----------|--------|--------------|----------------|
| Domain | Go, TS, Python | Inline | ✅ | ✅ | [502](502-domain-vertical-slice.md) |
| Process | Go, TS, Python | Inline | ✅ | ✅ | TODO (506) |
| Agent | Python only | 5 .tmpl files | ✅ | ✅ | [504](504-agent-vertical-slice.md) |
| UISurface | Go, TS, Python | Inline | ✅ | ✅ | TODO (507) |

**Notes:**
- Agent primitives use dedicated Go templates for consistent wiring checklists
- Domain/Process/UISurface use inline code generation in `primitive_scaffold.go`
- All kinds support `--include-gitops` for Argo CD ApplicationSet integration
- Vertical slice docs own primitive-specific scaffold details (operator + CLI + GitOps end-to-end)

---

## 3. Scaffold Output per Primitive Kind

### 3.1 Domain/Process/UISurface (Go)

```
primitives/{kind}/{name}/
├── README.md                          # Proto path, implementation checklist
├── catalog-info.yaml                  # Backstage component metadata
├── go.mod                             # Go module declaration
├── Dockerfile                         # Multi-stage build
├── cmd/
│   └── main.go                        # Placeholder entrypoint
└── internal/
    ├── handlers/
    │   └── handlers.go                # RPC stubs returning codes.Unimplemented
    └── tests/
        └── {rpc}_test.go              # Per-RPC failing test (one per RPC)
```

With `--include-test-harness`:
```
└── tests/
    └── run_integration_tests.sh       # 430-compliant test runner
└── internal/tests/
    └── integration_mode.go            # Mode switching helper
```

### 3.2 Domain/Process/UISurface (TypeScript)

```
primitives/{kind}/{name}/
├── README.md
├── catalog-info.yaml
├── package.json                       # npm config with jest
├── tsconfig.json                      # TypeScript configuration
├── jest.config.cjs                    # Jest configuration
├── Dockerfile                         # Node.js image
├── src/
│   └── handlers.ts                    # Handler class stubs
└── tests/
    └── handlers.test.ts               # Failing test suite
```

### 3.3 Domain/Process/UISurface (Python)

```
primitives/{kind}/{name}/
├── README.md
├── catalog-info.yaml
├── pyproject.toml                     # Basic pytest config
├── Dockerfile                         # Python image
├── src/
│   └── handlers.py                    # Handler class stubs
└── tests/
    └── test_handlers.py               # Failing test suite
```

### 3.4 Agent (Python only)

Agent primitives use dedicated templates from `packages/ameide_core_cli/internal/commands/templates/agent/`:

```
primitives/agent/{name}/
├── README.md                          # From readme.md.tmpl (wiring checklist)
├── catalog-info.yaml                  # Backstage component
├── pyproject.toml                     # From pyproject.toml.tmpl
├── Dockerfile.dev                     # Dev container with hot reload
├── Dockerfile.release                 # Multi-stage production build
├── src/
│   ├── __init__.py                    # From package_init.py.tmpl
│   ├── main.py                        # FastAPI entrypoint (/healthz, /invoke)
│   ├── agent.py                       # AgentPrimitive skeleton (NotImplementedError)
│   ├── prompts/
│   │   └── default.md                 # From prompt.md.tmpl
│   └── tools/
│       └── __init__.py                # From tools_init.py.tmpl
└── tests/
    └── test_agent.py                  # Failing test (RED by design)
```

---

## 4. Template Files (Agent Only)

| Template | Generated As | Variables |
|----------|--------------|-----------|
| `readme.md.tmpl` | README.md | Name, HumanName, Description, Owner, Model, RelDir |
| `pyproject.toml.tmpl` | pyproject.toml | Name, Description |
| `package_init.py.tmpl` | src/__init__.py | HumanName |
| `tools_init.py.tmpl` | src/tools/__init__.py | (none) |
| `prompt.md.tmpl` | src/prompts/default.md | (none) |

**Template location:** `packages/ameide_core_cli/internal/commands/templates/agent/`

The agent README.md template includes:
- Metadata table (owner, model, runtime, directory)
- Repo context (guardrail workflow, devcontainer, LangGraph runtime mounts, GitOps)
- Files generated table (main.py, agent.py with PromptRenderer, tools/, prompts/, tests/, Dockerfiles)
- Business meaning TODOs (problem, inputs/outputs, dependencies, failure signals)
- 4-step wiring checklist (uv workspace, secrets/ExternalSecret, testing, GitOps)

---

## 5. GitOps Manifest Structure

When `--include-gitops` is specified, scaffolding creates:

```
gitops/primitives/{kind}/{name}/
├── values.yaml         # Helm-like configuration
├── component.yaml      # Argo CD ApplicationSet component descriptor
└── kustomization.yaml  # Empty resources list (ready for customization)
```

**values.yaml** includes:
- Image reference, replicas, port
- ExternalSecret configuration (for agents: secret store ref, remote key, mount path, template keys)
- Environment variables for runtime mounts

**component.yaml** includes:
- Component name, project, namespace
- Chart repository URL and path
- Sync policy (automated prune + self-heal)
- Destination server

---

## 6. Worked Example: Scaffolding a Domain Primitive

### Step 1: Preview with dry-run

```bash
ameide primitive scaffold \
  --kind domain \
  --name orders \
  --proto-path packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation_service.proto \
  --lang go \
  --dry-run \
  --json
```

### Step 2: Execute scaffold

```bash
ameide primitive scaffold \
  --kind domain \
  --name orders \
  --proto-path packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation_service.proto \
  --lang go \
  --include-gitops \
  --json
```

### Step 3: Verify RED state

```bash
ameide primitive verify --kind domain --name orders --json
```

Tests should **fail** at this point (RED state per TDD). The agent's job is to implement handlers to make tests pass (GREEN).

### Step 4: Implement and iterate

1. Replace `codes.Unimplemented` in handlers with real logic
2. Run tests until GREEN
3. Run `ameide primitive verify` to confirm guardrails pass

---

## 7. Cross-Reference Index

### 7.1 Vertical Slices (Primitive-Specific)

| Primitive | Vertical Slice | Scaffold Phase | Status |
|-----------|---------------|----------------|--------|
| Domain | [502-domain-vertical-slice.md](502-domain-vertical-slice.md) | CLI K | ✅ |
| Process | 506-process-vertical-slice.md | — | TODO |
| Agent | [504-agent-vertical-slice.md](504-agent-vertical-slice.md) | CLI K | ✅ |
| UISurface | 507-uisurface-vertical-slice.md | — | TODO |

### 7.2 Operator Docs (CRD + Controller)

| Primitive | Operator Doc | Reconciler Status |
|-----------|--------------|-------------------|
| Domain | [498-domain-operator.md](498-domain-operator.md) | ✅ |
| Process | [499-process-operator.md](499-process-operator.md) | ✅ |
| Agent | [500-agent-operator.md](500-agent-operator.md) | ⚠️ Partial |
| UISurface | [501-uisurface-operator.md](501-uisurface-operator.md) | ✅ |

### 7.3 CLI Series (484)

| Topic | Document | Section |
|-------|----------|---------|
| TDD philosophy & failing tests | [484a](484a-ameide-cli-primitive-workflows.md) | §2.1, §4 |
| Shape vs Meaning separation | [484](484-ameide-cli-overview.md) | §2 |
| Proto contract (ScaffoldRequest/Result) | [484b](484b-ameide-cli-proto-contract.md) | §3.3 |
| GitOps alignment (434-compliant) | [484c](484c-ameide-cli-repo-gitops.md) | §5 |
| Migration phases | [484d](484d-ameide-cli-migration.md) | §2.2 |
| Anti-patterns (JHipster syndrome) | [484e](484e-ameide-cli-industry-patterns.md) | §4 |
| When to scaffold (decision matrix) | [484e](484e-ameide-cli-industry-patterns.md) | §6 |

### 7.4 Supporting Docs

| Topic | Document |
|-------|----------|
| Agent wiring checklist | `templates/agent/readme.md.tmpl` |
| Primitive repo layout | [477-primitive-stack.md](477-primitive-stack.md) §2 |
| Test infrastructure | [430v2 unified test infrastructure](430-unified-test-infrastructure-v2.md) |
| Operator patterns | [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) |
| Agent developer guide | [505-agent-developer.md](505-agent-developer.md) |

---

## 8. Source Code Locations

| Component | Path |
|-----------|------|
| Scaffold implementation | `packages/ameide_core_cli/internal/commands/primitive_scaffold.go` |
| Agent templates | `packages/ameide_core_cli/internal/commands/templates/agent/` |
| Template renderer | `packages/ameide_core_cli/internal/commands/templates_agent.go` |
| Command registration | `packages/ameide_core_cli/internal/commands/primitive.go` |

---

## 9. Key Invariants

1. **One-shot scaffolding**: Scaffold only when folder doesn't exist. Never overwrite existing files.
2. **Tests must fail**: Scaffolded tests call real handlers and fail until implemented (RED state).
3. **No business logic**: CLI generates wiring only; agents write meaning.
4. **GitOps is optional**: Use `--include-gitops` flag; default is `false`.
5. **Generated marker**: Files include `// CODEGEN: safe to delete, regenerate with 'ameide primitive scaffold'`

> See [484e §4](484e-ameide-cli-industry-patterns.md#4-anti-patterns-to-avoid) for anti-patterns including regeneration hell and magic comments.
