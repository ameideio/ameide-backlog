# 521c — Internal Generation Improvements (Buf/plugins)

This document tracks **internal generation** changes across the repo: Buf templates, protoc/Buƒ plugins, generated output conventions, and the developer workflow around regeneration.

It exists to make generator evolution auditable and reduce “tribal knowledge” when people touch scaffolding.

Baseline: `backlog/521b-internal-generation-baseline.md`

Companion docs:
- External generation improvements: `backlog/521d-external-generation-improvements.md`
- Verification improvements: `backlog/521g-internal-verification-improvements.md`, `backlog/521h-external-verification-improvements.md`

See also: `backlog/533-capability-implementation-playbook.md` (uses 521 as node context for capability implementation).

## Scope

Included:
- Protobuf-driven generation (`buf generate`) for SDKs and per-primitive glue.
- Generator/plugin code under `plugins/**`.
- Generation templates under `packages/ameide_core_proto/buf.gen.*.yaml`.
- Generated output placement rules (e.g. `primitives/**/internal/gen/`) and repo policies around committing/ignoring generated artifacts.

Not included:
- Runtime implementation changes that are unrelated to generation.
- Operator/controller logic unless it directly changes what codegen emits or how it is consumed.
- CLI scaffolding/orchestration changes (tracked in `backlog/521a-external-generation-baseline.md` and `backlog/521d-external-generation-improvements.md`).

Bright line:
- Anything derived from protobuf descriptors is reproducible by running `buf generate` in a clean checkout; anything else belongs to CLI scaffolding/orchestration.

## How to use this log

When you change code generation, add a new entry under **Log** with:
- What changed and why
- Which files/paths are affected
- How it was verified (commands or tests)
- Any follow-ups (if the change intentionally leaves gaps)

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
- This reduces generator surface area by removing the need for one Go “registration plugin” per primitive.

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
- This deliberately chooses the “strict” approach: refactor legacy naming into the target state instead of keeping long-lived ignore lists.

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

## Follow-ups (ideas)

- Add `.gitattributes` rules to mark committed generated files (e.g. `zz_generated.deepcopy.go`) as generated for diffs and GitHub linguist.
- Add a single script that rebuilds plugin binaries and runs all relevant `buf generate` templates for local developer convenience.
- Add “golden output” tests for generators (descriptor-in → file-out) to lock determinism byte-for-byte.
- Decide how CRDs are sourced for new operators (generate in `operators/*/config/` and sync into `gitops/ameide-gitops`, or treat GitOps CRDs as the canonical root and enforce drift checks).
