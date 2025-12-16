# 521 — Primitive Generation Reference

The authoritative mapping of what each primitive type gets from scaffolding (CLI) and code generation (Buf).

See also:
- `521a-cli-scaffolding.md` — CLI scaffold behavior (external tooling)
- `521b-buf-generation.md` — Buf generation behavior (internal tooling)
- `533-capability-implementation-playbook.md` — Uses 521 as context for implementation

---

## Scope Boundary

| Concern | Tool | Creates | Location |
|---------|------|---------|----------|
| **Repo-owned files** | CLI (`ameide scaffold`) | go.mod, Dockerfile, handlers, README | `primitives/{kind}/{name}/` |
| **Generated-only files** | Buf (`buf generate`) | gRPC stubs, SDK types, registration glue | `internal/gen/`, `gen/` |

**Hard rule (520 §2b):** Everything derived from protobuf descriptors is reproducible by running `buf generate` in a clean checkout. Everything else is CLI scaffolding and checked-in scaffold templates.

---

## Opinionated Defaults (Strict)

- Scaffolding is **idempotent by default**: safe to rerun; creates missing files only and avoids overwriting implementation-owned code.
- Verification is **strict by default**: missing tests fail, `AMEIDE_SCAFFOLD` markers fail, and generation drift is treated as a hard error (see `backlog/537-primitive-testing-discipline.md`).
- MCP adapters are Integration primitives: protocol + policy at the edge, semantics in Domain/Projection; tool schemas are expected to be proto-derived (planned: `protoc-gen-ameide-mcp-catalog`).

---

## Overview Table

| Primitive | Language | CLI Command | Buf Template Pattern | Proto Required |
|-----------|----------|-------------|----------------------|----------------|
| **Domain** | Go only | `ameide primitive scaffold --kind domain` | `buf.gen.domain-{name}.local.yaml` | Yes |
| **Projection** | Go only | `ameide primitive scaffold --kind projection` | `buf.gen.projection-{name}.local.yaml` | Yes |
| **Process** | Go only | `ameide primitive scaffold --kind process` | `buf.gen.process-{name}.local.yaml` | Yes |
| **Agent** | Python (default) | `ameide primitive scaffold --kind agent` | None | No |
| **Agent** | Go (legacy) | `ameide primitive scaffold --kind agent --lang go` | `buf.gen.agent-{name}.local.yaml` | Yes |
| **Integration (MCP)** | Go only | `ameide scaffold integration mcp-adapter` | None (planned: `mcp-catalog` plugin) | No |
| **UISurface** | Node | `ameide primitive scaffold --kind uisurface` | `buf.gen.uisurface-{name}.local.yaml` | No (driver proto) |

---

## Cross-Primitive Summary

| Primitive | Proto-Driven | Buf Template | EDA (Outbox) | Temporal | MCP Surface |
|-----------|--------------|--------------|--------------|----------|-------------|
| Domain | ✅ Yes | ✅ Per-primitive | ✅ Yes | ❌ | ❌ |
| Projection | ✅ Yes | ✅ Per-primitive | ❌ | ❌ | ❌ (tools via Integration) |
| Process | ✅ Yes | ✅ Per-primitive | ❌ | ✅ Yes | ❌ |
| Agent (Python) | ❌ No | ❌ None | ❌ | ❌ | ❌ |
| Agent (Go) | ✅ Yes | ✅ Per-primitive | ❌ | ❌ | ❌ |
| Integration (MCP) | ⚠️ Manual | ❌ None (planned) | ❌ | ❌ | ✅ Yes |
| UISurface | ⚠️ Static driver | ✅ Per-primitive | ❌ | ❌ | ❌ |

---

## Generation Flow

```
                    ┌──────────────────────────────────────────────────────────────┐
                    │  Proto Source: packages/ameide_core_proto/src/               │
                    └───────────────────────────────┬──────────────────────────────┘
                                                    │
              ┌─────────────────────────────────────┼─────────────────────────────────────┐
              │                                     │                                     │
              ▼                                     ▼                                     ▼
┌─────────────────────────────┐  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐
│  buf.gen.yaml               │  │  buf.gen.sdk-go.yaml            │  │  buf.gen.{kind}-{name}.         │
│  (BSR publish)              │  │  (SDK packages)                 │  │  local.yaml (per-primitive)     │
│                             │  │                                 │  │                                 │
│  - ES (TypeScript)          │  │  - Go protobuf                  │  │  - protoc-gen-ameide-           │
│  - Go (buf.build)           │  │  - gRPC Go                      │  │    register-go                  │
│  - Python                   │  │  - ConnectRPC                   │  │                                 │
│                             │  │                                 │  │  Out: primitives/{kind}         │
│  Out: gen/ts,               │  │  Out: ameide_sdk_go/            │  │       /{name}/internal/         │
│       gen/python,           │  │       gen/go/                   │  │       gen/                      │
│       ameide_sdk_go         │  │                                 │  │                                 │
└─────────────────────────────┘  └─────────────────────────────────┘  └─────────────────────────────────┘
```

---

## Per-Primitive Details

### Domain

**CLI Outputs:** See `521a-cli-scaffolding.md § Domain`

| Path | Purpose |
|------|---------|
| `primitives/domain/{name}/README.md` | Documentation |
| `primitives/domain/{name}/go.mod` | Go module |
| `primitives/domain/{name}/Dockerfile` | Container image |
| `primitives/domain/{name}/catalog-info.yaml` | Backstage catalog |
| `primitives/domain/{name}/cmd/main.go` | gRPC server entrypoint |
| `primitives/domain/{name}/cmd/dispatcher/main.go` | Outbox dispatcher entrypoint |
| `primitives/domain/{name}/internal/handlers/handlers.go` | gRPC handlers (proto-derived) |
| `primitives/domain/{name}/internal/ports/outbox.go` | Outbox port interface |
| `primitives/domain/{name}/internal/ports/outbound.go` | Outbound ports interface |
| `primitives/domain/{name}/internal/adapters/postgres/outbox.go` | Postgres outbox adapter |
| `primitives/domain/{name}/internal/adapters/sdk/clients.go` | SDK client wiring |
| `primitives/domain/{name}/internal/dispatcher/dispatcher.go` | Dispatcher logic |
| `primitives/domain/{name}/migrations/V1__domain_outbox.sql` | Flyway migration |
| `primitives/domain/{name}/migrations/Dockerfile.*` | Migration container |
| `primitives/domain/{name}/internal/tests/{rpc}_test.go` | Per-RPC tests |

**Buf Outputs:** See `521b-buf-generation.md § Domain`

| Path | Purpose |
|------|---------|
| `primitives/domain/{name}/internal/gen/domain_services.generated.go` | Service registration glue |

---

### Projection

**CLI Outputs:** See `521a-cli-scaffolding.md § Projection`

| Path | Purpose |
|------|---------|
| `primitives/projection/{name}/README.md` | Documentation |
| `primitives/projection/{name}/go.mod` | Go module |
| `primitives/projection/{name}/Dockerfile` | Container image |
| `primitives/projection/{name}/catalog-info.yaml` | Backstage catalog |
| `primitives/projection/{name}/cmd/main.go` | gRPC server entrypoint |
| `primitives/projection/{name}/internal/handlers/handlers.go` | Query handlers (proto-derived) |
| `primitives/projection/{name}/internal/tests/{rpc}_test.go` | Per-RPC tests |

**Buf Outputs:** See `521b-buf-generation.md § Projection`

| Path | Purpose |
|------|---------|
| `primitives/projection/{name}/internal/gen/projection_services.generated.go` | Service registration glue |

---

### Process

**CLI Outputs:** See `521a-cli-scaffolding.md § Process`

| Path | Purpose |
|------|---------|
| `primitives/process/{name}/README.md` | Documentation |
| `primitives/process/{name}/go.mod` | Go module |
| `primitives/process/{name}/Dockerfile` | Container image |
| `primitives/process/{name}/catalog-info.yaml` | Backstage catalog |
| `primitives/process/{name}/cmd/worker/main.go` | Temporal worker entrypoint |
| `primitives/process/{name}/cmd/ingress/main.go` | HTTP/gRPC ingress entrypoint |
| `primitives/process/{name}/internal/handlers/handlers.go` | Proto handlers |
| `primitives/process/{name}/internal/workflows/workflow.go` | Temporal workflow stub |
| `primitives/process/{name}/internal/ingress/router.go` | Ingress router |
| `primitives/process/{name}/internal/process/state.go` | Process state struct |
| `primitives/process/{name}/internal/tests/{rpc}_test.go` | Per-RPC tests |

**Buf Outputs:** See `521b-buf-generation.md § Process`

| Path | Purpose |
|------|---------|
| `primitives/process/{name}/internal/gen/process_services.generated.go` | Service registration glue |

---

### Agent (Python/LangGraph)

**CLI Outputs:** See `521a-cli-scaffolding.md § Agent`

| Path | Purpose |
|------|---------|
| `primitives/agent/{name}/README.md` | Documentation with wiring checklist |
| `primitives/agent/{name}/pyproject.toml` | Python package config |
| `primitives/agent/{name}/src/__init__.py` | Package init |
| `primitives/agent/{name}/src/tools/__init__.py` | Tools package |
| `primitives/agent/{name}/src/prompts/{name}.md` | Prompt template |
| `primitives/agent/{name}/langgraph.dev.json` | LangGraph dev config |

**Buf Outputs:** None — Agent uses Ameide SDK clients, no proto generation needed

---

### Integration (MCP Adapter)

**CLI Outputs:** See `521a-cli-scaffolding.md § Integration`

| Path | Purpose |
|------|---------|
| `primitives/integration/{cap}-mcp-adapter/README.md` | Documentation |
| `primitives/integration/{cap}-mcp-adapter/go.mod` | Go module |
| `primitives/integration/{cap}-mcp-adapter/Dockerfile` | Container image |
| `primitives/integration/{cap}-mcp-adapter/catalog-info.yaml` | Backstage catalog |
| `primitives/integration/{cap}-mcp-adapter/cmd/main.go` | HTTP/stdio entrypoint |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/jsonrpc.go` | JSON-RPC types |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/errors.go` | Error types |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/server.go` | MCP server + tools |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/http.go` | HTTP transport |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/stdio.go` | stdio transport |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/auth.go` | Identity/scope handling |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/oauth.go` | PRM endpoint |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/origin.go` | Origin validation |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/*_test.go` | Unit tests |

**Buf Outputs:** None currently — planned: `protoc-gen-ameide-mcp-catalog` for tool catalog generation

---

### UISurface

**CLI Outputs:** See `521a-cli-scaffolding.md § UISurface`

| Path | Purpose |
|------|---------|
| `primitives/uisurface/{name}/README.md` | Documentation |
| `primitives/uisurface/{name}/package.json` | Node package |
| `primitives/uisurface/{name}/server.js` | Express server |
| `primitives/uisurface/{name}/Dockerfile` | Container image |
| `primitives/uisurface/{name}/catalog-info.yaml` | Backstage catalog |
| `primitives/uisurface/{name}/tests/server.test.js` | Tests |

**Buf Outputs:** See `521b-buf-generation.md § UISurface`

| Path | Purpose |
|------|---------|
| `primitives/uisurface/{name}/internal/gen/index.generated.html` | Static HTML (from driver proto) |

---

## Related Documents

- `520-primitives-stack-v2.md` — Architecture decisions, §2b boundary rule
- `521a-cli-scaffolding.md` — CLI scaffold details and changelog
- `521b-buf-generation.md` — Buf generation details and changelog
- `533-capability-implementation-playbook.md` — Uses 521 as context for implementation
- `534-mcp-protocol-adapter.md` — MCP adapter specification
