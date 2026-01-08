# 521a — External Generation Baseline (CLI scaffolding)

Tracks CLI scaffold behavior — what repo-owned files the CLI creates per primitive type.

Companion docs:
- Internal generation baseline (Buf/plugins): `backlog/521b-internal-generation-baseline.md`
- External generation improvements log: `backlog/521d-external-generation-improvements.md`
- Primitive mapping reference: `backlog/521i-primitive-generation-reference.md`
- Verification baselines: `backlog/521e-internal-verification-baseline.md`, `backlog/521f-external-verification-baseline.md`

See `backlog/521i-primitive-generation-reference.md` for the high-level mapping.

---

## Scope

The CLI creates **repo-owned files** that are never overwritten by generation:

| Output Type | Examples | Owned By |
|-------------|----------|----------|
| Project skeleton | `go.mod`, `pyproject.toml`, `package.json` | Implementation |
| Entry points | `cmd/main.go`, `cmd/worker/main.go` | Implementation |
| Handler stubs | `internal/handlers/handlers.go` | Implementation |
| Infrastructure | `Dockerfile`, `catalog-info.yaml`, `README.md` | Implementation |
| Migrations | `migrations/V1__*.sql` | Implementation |
| Buf templates | `buf.gen.{kind}-{name}.local.yaml` | CLI (config) |

The CLI does **not** create:
- `internal/gen/**` — created by `buf generate`
- SDK stubs — created by `buf generate --template buf.gen.sdk-*.yaml`

---

## Boundary

The CLI can be repo-aware and environment-aware, but it does not "grow back into codegen".

The CLI is a first-class tool for both humans and coding agents (not a manual-only tool).

**Hard rule (520 §2b):**
- Everything derived from protobuf descriptors is reproducible by running `buf generate` in a clean checkout.
- Everything else is CLI orchestration and checked-in scaffold templates (repo-authored files).

### Allowed / expected (CLI orchestrator)

- Create and maintain repo-owned project skeletons (module dirs, `go.mod`/`pyproject.toml`, Dockerfiles, `cmd/` entrypoints, READMEs/checklists).
- Drive multi-step workflows: run `buf lint`/`buf breaking`/`buf generate`, run tests, build images, wire GitOps manifests/CR templates, and open PRs.
- Provide convenience wrappers (e.g., `ameide dev check`) that run the same checks CI runs, without becoming the canonical gate.

### Not allowed (CLI orchestrator)

- Re-implement proto parsing/templating that Buf plugins already provide (no bespoke "generator pipeline" in the CLI).
- "Sync" generated code by patching files itself; regeneration is `buf generate` + regen-diff in CI.
- Write generated artifacts into repo-owned roots (keep `clean: true` safe).

### Canonical gate

- CI remains authoritative: regen-diff + compile/tests are the drift guardrails. The CLI may wrap them locally, but it does not introduce a separate "verify CLI gate".

---

## Opinionated Defaults (Strict)

- Scaffolds are **idempotent by default**: create missing files only; never overwrite implementation-owned code.
- Existing primitives can be **backfilled** with missing repo-owned wiring/tests without requiring proto (best-effort), but remain RED until implemented.
- Verification is **strict by default**: missing tests fail and `AMEIDE_SCAFFOLD` markers fail `ameide primitive verify` (see `backlog/537-primitive-testing-discipline.md`).

---

## Per-Primitive CLI Outputs

### Domain

**Command:** `ameide primitive scaffold --kind domain --name {name} --proto-path {path}`

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
| `primitives/domain/{name}/migrations/Dockerfile.dev` | Migration container (dev) |
| `primitives/domain/{name}/migrations/Dockerfile.release` | Migration container (release) |
| `primitives/domain/{name}/internal/tests/{rpc}_test.go` | Per-RPC tests |

**Templates:** `packages/ameide_core_cli/internal/commands/templates/domain/`

---

### Projection

**Command:** `ameide primitive scaffold --kind projection --name {name} --proto-path {path}`

| Path | Purpose |
|------|---------|
| `primitives/projection/{name}/README.md` | Documentation |
| `primitives/projection/{name}/go.mod` | Go module |
| `primitives/projection/{name}/Dockerfile` | Container image |
| `primitives/projection/{name}/catalog-info.yaml` | Backstage catalog |
| `primitives/projection/{name}/cmd/main.go` | gRPC server entrypoint |
| `primitives/projection/{name}/internal/handlers/handlers.go` | Query handlers (proto-derived) |
| `primitives/projection/{name}/internal/tests/{rpc}_test.go` | Per-RPC tests |

**Templates:** Shares generic Go templates with Domain

---

### Process

**Command:** `ameide primitive scaffold --kind process --name {name} --proto-path {path}`

Planned direction (see `backlog/511-process-primitive-scaffolding-refactoring.md`): Process scaffolding becomes **BPMN-first** (`primitives/process/{name}/process.bpmn`) and compilation output is generated-only under `internal/gen/**` (no process business API required).

| Path | Purpose |
|------|---------|
| `primitives/process/{name}/README.md` | Documentation |
| `primitives/process/{name}/go.mod` | Go module |
| `primitives/process/{name}/Dockerfile` | Container image |
| `primitives/process/{name}/catalog-info.yaml` | Backstage catalog |
| `primitives/process/{name}/process.bpmn` | BPMN source of truth (v1 compiler input) |
| `primitives/process/{name}/cmd/worker/main.go` | Temporal worker entrypoint |
| `primitives/process/{name}/cmd/ingress/main.go` | HTTP/gRPC ingress entrypoint |
| `primitives/process/{name}/internal/handlers/handlers.go` | Ops/control handlers (optional; not a business/query API) |
| `primitives/process/{name}/internal/workflows/workflow.go` | Temporal workflow stub |
| `primitives/process/{name}/internal/ingress/router.go` | Ingress router |
| `primitives/process/{name}/internal/process/state.go` | Process state struct |
| `primitives/process/{name}/internal/gen/**` | Generated code (compiler output; do not edit by hand) |
| `primitives/process/{name}/internal/tests/**` | Tests (workflow/router; no per-RPC requirement) |

**Templates:** `packages/ameide_core_cli/internal/commands/templates/process/`

---

### Agent (Python/LangGraph)

**Command:** `ameide primitive scaffold --kind agent --name {name}` (default: Python)

| Path | Purpose |
|------|---------|
| `primitives/agent/{name}/README.md` | Documentation with wiring checklist |
| `primitives/agent/{name}/pyproject.toml` | Python package config |
| `primitives/agent/{name}/src/__init__.py` | Package init |
| `primitives/agent/{name}/src/tools/__init__.py` | Tools package |
| `primitives/agent/{name}/src/prompts/{name}.md` | Prompt template |
| `primitives/agent/{name}/langgraph.dev.json` | LangGraph dev config |

**Templates:** `packages/ameide_core_cli/internal/commands/templates/agent/`

---

### Integration (MCP Adapter)

**Command:** `ameide scaffold integration mcp-adapter --capability {capability}`

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
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/http_test.go` | HTTP tests |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/stdio.go` | stdio transport |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/auth.go` | Identity/scope handling |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/auth_test.go` | Auth tests |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/oauth.go` | PRM endpoint |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/oauth_test.go` | OAuth tests |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/origin.go` | Origin validation |
| `primitives/integration/{cap}-mcp-adapter/internal/mcp/origin_test.go` | Origin tests |

**Notes (534/535/536 alignment):**
- Streamable HTTP endpoint is `/mcp`; PRM discovery is `/.well-known/oauth-protected-resource`.
- Scope gating is enforced (`mcp:tools`, `mcp:resources`); stdio is treated as trusted local dev by default.
- Scaffold includes `ping` and `validateWrite` tool surfaces; capability-specific tools must be registered (proto-first) before the adapter is considered complete.

**Templates:** `packages/ameide_core_cli/internal/commands/templates/integration/mcp_adapter/`

---

### UISurface

**Command:** `ameide primitive scaffold --kind uisurface --name {name}`

| Path | Purpose |
|------|---------|
| `primitives/uisurface/{name}/README.md` | Documentation |
| `primitives/uisurface/{name}/package.json` | Node package |
| `primitives/uisurface/{name}/server.js` | Express server |
| `primitives/uisurface/{name}/Dockerfile` | Container image |
| `primitives/uisurface/{name}/catalog-info.yaml` | Backstage catalog |
| `primitives/uisurface/{name}/tests/server.test.js` | Tests |

**Templates:** Uses inline generation (not separate template files)

---

## Log (moved)

This file is the **baseline**. The chronological log moved to:
- `backlog/521d-external-generation-improvements.md`

Rationale: keep baseline stable; track changes in one place.
