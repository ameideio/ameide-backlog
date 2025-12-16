# 521b — Buf Generation (Internal Tooling)

Tracks Buf/plugin behavior — what generated-only files Buf creates from proto sources.

See `521-primitive-generation-reference.md` for the high-level mindmap.

---

## Scope

Generated-only outputs (safe to delete/regenerate):

| Output | Template | Location |
|--------|----------|----------|
| SDK Go stubs | `buf.gen.sdk-go.yaml` | `packages/ameide_sdk_go/gen/go/` |
| SDK TS stubs | `buf.gen.ts.yaml` | `packages/ameide_core_proto/gen/ts/` |
| SDK Python stubs | `buf.gen.yaml` | `packages/ameide_core_proto/gen/python/` |
| Per-primitive registration | `buf.gen.{kind}-{name}.local.yaml` | `primitives/{kind}/{name}/internal/gen/` |

**Not included:**
- CLI scaffold templates and GitOps wiring automation (tracked in `521a-cli-scaffolding.md`)
- Runtime implementation changes unrelated to generation
- Operator/controller logic unless it directly changes what codegen emits

**Bright line:**
- Anything derived from protobuf descriptors is reproducible by running `buf generate` in a clean checkout; anything else belongs to the CLI orchestrator and/or implementation-owned templates.

---

## Verification Coupling (CLI)

Buf is the **source of truth** for generated artifacts, and verification treats generation drift as a hard error:

- `ameide primitive verify --check codegen` expects the relevant generated roots to exist and be fresh.
- Missing generated roots are treated as **FAIL** (the fix is to run the appropriate `buf generate` template(s)).
- TypeScript drift is verified via a temp-tree generate+diff; Go/Python drift is verified via proto-vs-generated staleness heuristics.

This keeps “proto-first” real in practice: scaffolding creates repo-owned structure, Buf creates generated-only code, and verify fails if they drift.

---

## Buf Generation Layers

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
│  Plugins:                   │  │  Plugins:                       │  │  Plugin:                        │
│  - buf.build/bufbuild/es    │  │  - protocolbuffers/go           │  │  - protoc-gen-ameide-           │
│  - buf.build/proto/go       │  │  - grpc/go                      │  │    register-go                  │
│  - buf.build/grpc/go        │  │  - connectrpc/go                │  │                                 │
│  - buf.build/connectrpc/go  │  │                                 │  │  Or for UISurface:              │
│  - buf.build/proto/python   │  │  Out: ameide_sdk_go/gen/go/     │  │  - protoc-gen-ameide-           │
│  - buf.build/grpc/python    │  │                                 │  │    uisurface-static             │
│                             │  │                                 │  │                                 │
│  Out: gen/ts, gen/python,   │  │                                 │  │  Out: primitives/{kind}         │
│       ameide_sdk_go         │  │                                 │  │       /{name}/internal/gen/     │
└─────────────────────────────┘  └─────────────────────────────────┘  └─────────────────────────────────┘
```

---

## Templates

### Global Templates

| Template | Purpose | Output Location |
|----------|---------|-----------------|
| `buf.gen.yaml` | BSR publish (ES, Go, Python) | `gen/ts`, `gen/python`, `ameide_sdk_go` |
| `buf.gen.sdk-go.yaml` | SDK Go stubs | `packages/ameide_sdk_go/gen/go/` |
| `buf.gen.sdk-ts.local.yaml` | SDK TS stubs (local) | `packages/ameide_sdk_ts/src/proto/` |
| `buf.gen.sdk-go.local.yaml` | SDK Go stubs (local) | `packages/ameide_sdk_go/gen/go/` |
| `buf.gen.ts.yaml` | TypeScript stubs | `packages/ameide_core_proto/gen/ts/` |
| `buf.gen.ts-only.yaml` | TypeScript only (no Go/Python) | `gen/ts/` |

### Per-Primitive Templates

| Template Pattern | Primitive | Plugin |
|------------------|-----------|--------|
| `buf.gen.domain-{name}.local.yaml` | Domain | `protoc-gen-ameide-register-go` |
| `buf.gen.projection-{name}.local.yaml` | Projection | `protoc-gen-ameide-register-go` |
| `buf.gen.process-{name}.local.yaml` | Process | `protoc-gen-ameide-register-go` |
| `buf.gen.agent-{name}.local.yaml` | Agent (Go) | `protoc-gen-ameide-register-go` |
| `buf.gen.uisurface-{name}.local.yaml` | UISurface | `protoc-gen-ameide-uisurface-static` |

---

## Plugins

| Plugin | Purpose | Output | Location |
|--------|---------|--------|----------|
| `protoc-gen-ameide-register-go` | Service registration glue | `*_services.generated.go` | `plugins/ameide_register_go/` |
| `protoc-gen-ameide-uisurface-static` | Static HTML for UISurface | `index.generated.html` | `plugins/ameide_uisurface_static/` |
| `protoc-gen-ameide-mcp-catalog` | MCP tool catalog (planned) | `mcp-tool-catalog.json` | N/A |

### protoc-gen-ameide-register-go

Generates typed service registration glue (interface + register function) without per-primitive hand wiring.

**Options:**
- `package=gen` — Output package name
- `output_file=domain_services.generated.go` — Output filename
- `register_func=RegisterDomainServices` — Register function name
- `interface_name=DomainServices` — Interface name
- `go_import_prefix=github.com/ameideio/ameide-sdk-go/gen/go` — Import prefix for relative `go_package`

**Example template:**
```yaml
version: v2
clean: true
inputs:
  - directory: src
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/ameideio/ameide-sdk-go/gen/go
plugins:
  - local: protoc-gen-ameide-register-go
    out: ../../primitives/domain/transformation/internal/gen
    opt:
      - package=gen
      - output_file=domain_services.generated.go
      - register_func=RegisterDomainServices
      - interface_name=DomainServices
```

### protoc-gen-ameide-uisurface-static

Generates a deterministic `index.generated.html` from UISurface proto shape for v0 hello demos.

**Options:**
- `title=Hello UISurface v0` — Page title
- `heading=Hello UISurface v0` — Page heading

---

## Per-Primitive Buf Outputs

### Domain

**Template:** `buf.gen.domain-{name}.local.yaml`

| Generated File | Purpose |
|----------------|---------|
| `primitives/domain/{name}/internal/gen/domain_services.generated.go` | Service registration glue |

**Generation command:**
```bash
cd packages/ameide_core_proto && \
  PATH="$PWD/../../bin:$PATH" buf generate \
  --template buf.gen.domain-{name}.local.yaml \
  --path src/ameide_core_proto/{capability}/domain/v1/{service}.proto
```

### Projection

**Template:** `buf.gen.projection-{name}.local.yaml`

| Generated File | Purpose |
|----------------|---------|
| `primitives/projection/{name}/internal/gen/projection_services.generated.go` | Service registration glue |

**Generation command:**
```bash
cd packages/ameide_core_proto && \
  PATH="$PWD/../../bin:$PATH" buf generate \
  --template buf.gen.projection-{name}.local.yaml \
  --path src/ameide_core_proto/{capability}/query/v1/{service}.proto
```

### Process

**Template:** `buf.gen.process-{name}.local.yaml`

| Generated File | Purpose |
|----------------|---------|
| `primitives/process/{name}/internal/gen/process_services.generated.go` | Service registration glue |

**Generation command:**
```bash
cd packages/ameide_core_proto && \
  PATH="$PWD/../../bin:$PATH" buf generate \
  --template buf.gen.process-{name}.local.yaml \
  --path src/ameide_core_proto/{capability}/process/v1/{service}.proto
```

### Agent (Go)

**Template:** `buf.gen.agent-{name}.local.yaml`

| Generated File | Purpose |
|----------------|---------|
| `primitives/agent/{name}/internal/gen/agent_services.generated.go` | Service registration glue |

### Agent (Python)

**No Buf generation** — Python agents use Ameide SDK clients directly.

### Integration (MCP)

**No Buf generation currently** — MCP adapter templates are CLI-embedded.

**Planned:** `protoc-gen-ameide-mcp-catalog` to generate tool catalog from `(ameide.mcp.expose)` annotations.

### UISurface

**Template:** `buf.gen.uisurface-{name}.local.yaml`

| Generated File | Purpose |
|----------------|---------|
| `primitives/uisurface/{name}/internal/gen/index.generated.html` | Static HTML from proto |

**Generation command:**
```bash
cd packages/ameide_core_proto && \
  PATH="$PWD/../../bin:$PATH" buf generate \
  --template buf.gen.uisurface-{name}.local.yaml \
  --path src/ameide_core_proto/primitive/v1/primitive_service.proto
```

---

## Log

### 2025-12-13 — Unify Go service registration glue

Change:
- Added a generic Go protoc plugin that generates typed service registration glue (interface + register function) without per-primitive hand wiring.
- The generator reads `source_file_descriptors` when available so SOURCE-retained file options (e.g. `go_package`) are respected.

Key paths:
- `plugins/ameide_register_go/main.go`
- `plugins/ameide_register_go/internal/generator/generator.go`
- `plugins/ameide_register_go/internal/generator/generator_test.go`
- `packages/ameide_core_proto/buf.gen.domain-transformation.local.yaml`
- `packages/ameide_core_proto/buf.gen.process-ping.local.yaml`
- `packages/ameide_core_proto/buf.gen.agent-echo.local.yaml`
- `packages/ameide_core_proto/buf.gen.projection-foo.local.yaml`
- `packages/ameide_core_proto/buf.gen.integration-echo.local.yaml`

Verification:
- `go test ./plugins/ameide_register_go/...`
- `buf generate` using one of the `packages/ameide_core_proto/buf.gen.*.local.yaml` templates

Notes:
- This reduces generator surface area by removing the need for one Go "registration plugin" per primitive.

### 2025-12-13 — UISurface static generator for v0 hello

Change:
- Added a small generator that emits a deterministic `index.generated.html` from UISurface proto shape to prove the full GitOps stack end-to-end.

Key paths:
- `plugins/ameide_uisurface_static/main.go`
- `plugins/ameide_uisurface_static/internal/generator/generator.go`
- `packages/ameide_core_proto/buf.gen.uisurface-hello.local.yaml`

Verification:
- `go test ./plugins/ameide_uisurface_static/...`
- `buf generate --template packages/ameide_core_proto/buf.gen.uisurface-hello.local.yaml --path src/ameide_core_proto/uisurface/v1/hello_uisurface.proto`

### 2025-12-13 — Generated output location and gitignore policy

Change:
- Standardized per-primitive generated output under `primitives/**/internal/gen/`.
- Kept generated artifacts out of git by ignoring that directory.

Key paths:
- `.gitignore`
- `primitives/**/internal/gen/` (generated-only roots; not committed)

Verification:
- `git status` stays clean after regenerating (no generated files staged)

### 2025-12-14 — Buf/BSR-native refactor (snake_case + CSR subject options)

Change:
- Migrated all tracked `.proto` filenames to `lower_snake_case.proto` and removed Buf lint compatibility shims so `STANDARD` is enforceable without exceptions.
- Added Buf/BSR CSR subject associations to topic-aggregator messages using `buf.confluent.v1.subject` (TopicNameStrategy-style `<topic>-value`).

Key paths:
- `packages/ameide_core_proto/buf.yaml`
- `packages/ameide_core_proto/src/ameide_core_proto/**` (renamed + imports updated)

Verification:
- `cd packages/ameide_core_proto && buf format -w && buf lint && buf build`

Notes:
- This deliberately chooses the "strict" approach: refactor legacy naming into the target state instead of keeping long-lived ignore lists.

### 2025-12-14 — SDK regeneration templates (local TS + export drift fix)

Change:
- Added a local TS SDK generation template so TypeScript stubs can be regenerated from the workspace proto sources (not only synced from BSR).
- Fixed SDK TS barrel exports so `createClientSet()` remains stable under unit tests (kept `extensions` runtime service and added missing Commerce modules).

Key paths:
- `packages/ameide_core_proto/buf.gen.sdk-ts.local.yaml`
- `packages/ameide_sdk_ts/src/proto/index.ts`
- `packages/ameide_core_proto/src/index.ts`

Verification:
- `cd packages/ameide_core_proto && buf generate --template buf.gen.sdk-ts.local.yaml`
- `pnpm -C packages/ameide_sdk_ts test`

### 2025-12-14 — Register-go import prefix support

Change:
- Extended `protoc-gen-ameide-register-go` to accept `go_import_prefix=...` and apply it when protos use relative `go_package` paths (Buf managed mode pattern).
- Updated generated per-primitive Buf templates to pass the SDK Go import prefix so generated `internal/gen/*_services.generated.go` compiles without manual edits.

Key paths:
- `plugins/ameide_register_go/internal/generator/generator.go`
- `plugins/ameide_register_go/internal/generator/params.go`
- `packages/ameide_core_cli/internal/commands/primitive_scaffold.go`

Verification:
- `go test ./plugins/ameide_register_go/...`
- `go test ./packages/ameide_core_cli/...`

### 2025-12-14 — Trace Context fields in core message envelopes

Change:
- Added optional W3C Trace Context fields (`traceparent`, `tracestate`) to the Scrum domain and process message envelope types so async propagation is first-class and consistent with 496/509 guidance.

Key paths:
- `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_common.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/process/scrum/v1/process_scrum_facts.proto`

Verification:
- `cd packages/ameide_core_proto && buf lint && buf build`

### 2025-12-15 — Domain-owned Flyway migrations + migration image contract

Change:
- Switched Domain scaffolding from an ad-hoc `migrations/0001_create_outbox.sql` file to a Flyway-compatible, domain-owned migrations set (starting at `migrations/V1__domain_outbox.sql`).
- Added per-domain migration image Dockerfiles under `migrations/` so the Domain operator can run migrations automatically via `spec.db.migrationJobImage`.
- Updated Domain verify/guardrails so domains must ship migrations + migration image (no central migration bundle for domain DB schemas).

Key paths:
- `packages/ameide_core_cli/internal/commands/primitive_scaffold.go`
- `packages/ameide_core_cli/internal/commands/templates/domain/readme.md.tmpl`
- `packages/ameide_core_cli/internal/commands/primitive.go`
- `backlog/498-domain-operator.md`
- `backlog/510-domain-primitive-scaffolding.md`

Verification:
- `go test ./packages/ameide_core_cli/...`
- `go test ./operators/domain-operator/...`

---

## Follow-ups (ideas)

- Add `.gitattributes` rules to mark committed generated files (e.g. `zz_generated.deepcopy.go`) as generated for diffs and GitHub linguist.
- Add a single script that rebuilds plugin binaries and runs all relevant `buf generate` templates for local developer convenience.
- Add "golden output" tests for generators (descriptor-in → file-out) to lock determinism byte-for-byte.
- Decide how CRDs are sourced for new operators (generate in `operators/*/config/` and sync into `gitops/ameide-gitops`, or treat GitOps CRDs as the canonical root and enforce drift checks).
- Add `protoc-gen-ameide-mcp-catalog` plugin to generate MCP tool catalog from `(ameide.mcp.expose)` annotations (see 534 §14).
