# 521a — CLI Scaffolding (External Tooling)

Tracks CLI scaffold behavior — what repo-owned files the CLI creates per primitive type.

See `521-primitive-generation-reference.md` for the high-level mindmap.

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

## Log

### 2025-12-13 — Split: CLI orchestrates, Buf generates

Change:
- Reframed platform guidance to keep the CLI for external wiring (scaffold, GitOps, guardrails), and keep `buf generate` + plugins as the internal deterministic generator.

Key paths:
- `backlog/520-primitives-stack-v2.md`
- `backlog/510-domain-primitive-scaffolding.md`
- `backlog/511-process-primitive-scaffolding.md`
- `backlog/512-agent-primitive-scaffolding.md`
- `backlog/513-uisurface-primitive-scaffolding.md`

Verification:
- Documentation review only (no runtime impact).

### 2025-12-13 — Scaffold per-primitive Buf templates + guidance

Change:
- `ameide primitive scaffold` creates per-primitive Buf template files (config) and prints concrete `buf generate` commands in `next_steps` so new scaffolds compile with minimal manual wiring.
- The CLI remains responsible for external layout; deterministic outputs remain Buf-owned (`primitives/**/internal/gen/`).
- Agent v0 smoke hooks avoid persistent thread state by using a per-Job `threadId` instead of a fixed value.

Key paths:
- `packages/ameide_core_cli/internal/commands/primitive_scaffold.go`
- `packages/ameide_core_cli/internal/commands/primitive_scaffold_test.go`
- `gitops/ameide-gitops/sources/values/_shared/apps/agent-echo-v0-smoke.yaml`

Verification:
- `cd packages/ameide_core_cli && go test ./...`
- `./bin/ameide primitive scaffold --kind domain --name scaffoldv1 --proto-path packages/ameide_core_proto/src/... --dry-run --json`

### 2025-12-14 — Align scaffolds with 496 EDA principles

Change:
- Domain scaffolds model EDA-required metadata explicitly in the outbox port and migration (tenant isolation, idempotency key, correlation/causation, trace linkage).
- Process/Projection/Integration scaffolds include an explicit EDA/idempotency checklist in scaffold docs.
- Scaffolded handler classes/methods include clearer doc comments/docstrings.

Key paths:
- `packages/ameide_core_cli/internal/commands/templates/domain/internal/ports/outbox_port.go.tmpl`
- `packages/ameide_core_cli/internal/commands/templates/domain/internal/adapters/postgres/outbox_postgres.go.tmpl`
- `packages/ameide_core_cli/internal/commands/primitive_scaffold.go`
- `packages/ameide_core_cli/internal/commands/templates/process/readme.md.tmpl`

Verification:
- `cd packages/ameide_core_cli && go test ./...`

### 2025-12-14 — Remove "human-owned" boundary language

Change:
- Replaced "human-owned" wording with clearer repo semantics:
  - **Generated-only** outputs (Buf/plugin-owned; safe to delete/regenerate).
  - **Implementation-owned** runtime code and checked-in scaffold templates (never overwritten by generation).

Key paths:
- `backlog/522-cli-orchestration-improvements.md`
- `backlog/510-domain-primitive-scaffolding.md`
- `backlog/511-process-primitive-scaffolding.md`
- `backlog/512-agent-primitive-scaffolding.md`
- `backlog/513-uisurface-primitive-scaffolding.md`

Verification:
- Documentation review only.

### 2025-12-14 — Reduce scaffold manual steps (auto go.work + auto buf generate)

Change:
- Fixed Go workspace wiring so `go work use` happens after files are written (so `go.mod` exists), making newly scaffolded Go primitives immediately buildable in the repo workspace.
- Orchestrated `buf generate` automatically (using the per-primitive template the CLI creates) so `internal/gen/**` exists without a manual follow-up command.
- Added an explicit Go import-prefix normalization (`go_import_prefix`) for relative `go_package` values so scaffolded Go code imports SDK packages consistently.

Key paths:
- `packages/ameide_core_cli/internal/commands/primitive_scaffold.go`
- `plugins/ameide_register_go/internal/generator/params.go`

Verification:
- `go test ./packages/ameide_core_cli/...`

### 2025-12-14 — Doc + template alignment cleanups

Change:
- Updated CLI-facing docs/backlogs to use real, repo-valid proto paths (and removed the obsolete `484-ameide-cli-ORIGINAL-BACKUP.md` copy) so examples are runnable and don't teach legacy layouts.
- Updated process scaffold README language to reference W3C Trace Context (`traceparent` / `tracestate`) rather than `trace_id` as an envelope field.
- Fixed `ameide primitive impact --proto-path` help text to reflect that `--proto-path` is always required for impact analysis.

Key paths:
- `packages/ameide_core_cli/README.md`
- `packages/ameide_core_cli/internal/commands/templates/process/readme.md.tmpl`
- `packages/ameide_core_cli/internal/commands/primitive_commands.go`
- `backlog/472-ameide-information-application.md`
- `backlog/473-ameide-technology.md`

Verification:
- Documentation review only (plus `go test ./...` for CLI changes).

### 2025-12-14 — Fix agent smoke assertions (turnCount JSON)

Change:
- Updated the Agent v0 ArgoCD PostSync hook job to assert `turnCount` using a whitespace-tolerant regex instead of a space-sensitive string match.

Key paths:
- `gitops/ameide-gitops/sources/values/_shared/apps/agent-echo-v0-smoke.yaml`

Verification:
- Review only (cluster connectivity required to re-run the hook job).

### 2025-12-15 — MCP adapter scaffold command

Change:
- Added `ameide scaffold integration mcp-adapter --capability {cap}` command that generates a complete MCP protocol adapter skeleton with HTTP/stdio transports, origin validation, OAuth PRM endpoint, identity/scope handling, and test coverage.

Key paths:
- `packages/ameide_core_cli/internal/commands/scaffold.go`
- `packages/ameide_core_cli/internal/commands/templates/integration/mcp_adapter/`

Verification:
- `go test ./packages/ameide_core_cli/...`
- `ameide scaffold integration mcp-adapter --capability transformation --dry-run --json`

### 2025-12-16 — Opinionated scaffolding defaults + strict verify

Change:
- Made scaffolding idempotent by default (create missing only; safe to rerun).
- Made verification strict by default (no tests and `AMEIDE_SCAFFOLD` markers fail).

Key paths:
- `packages/ameide_core_cli/internal/commands/primitive_commands.go`
- `packages/ameide_core_cli/internal/commands/primitive_scaffold.go`
- `packages/ameide_core_cli/internal/commands/primitive.go`

Verification:
- `go test ./packages/ameide_core_cli/...`

---

## Follow-ups (ideas)

- ~~Add an official MCP adapter scaffold path...~~ ✅ Done: `ameide scaffold integration mcp-adapter`
- Add a dedicated CLI command that runs a full vertical loop (generate → build → push → GitOps sync → smoke), with clear separation between "repo actions" and "cluster actions".
- Ensure CLI-generated projects default to the same naming conventions used by the v0 samples.
- Add a proto-driven MCP schema generator plugin (e.g., `protoc-gen-ameide-mcp-catalog`) so MCP tool schemas/manifests are derived from proto annotations and cannot drift; track the actual implementation in `521b-buf-generation.md` once it lands.
