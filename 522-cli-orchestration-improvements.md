# 522 — CLI Orchestrator (boundary + improvements log)

This document defines the **boundary** for the Ameide CLI when it acts as an orchestrator (what it does and does not do), and tracks changes to that orchestration behavior over time.

This is intentionally separate from codegen plugin changes (tracked in `backlog/521-code-generation-improvements.md`).

## Boundary

The CLI can be repo-aware and environment-aware, but it does not “grow back into codegen”.

The CLI is a first-class tool for both humans and coding agents (not a manual-only tool).

Hard rule (see `backlog/520-primitives-stack-v2.md` §2b):

- Everything derived from protobuf descriptors is reproducible by running `buf generate` in a clean checkout.
- Everything else is CLI orchestration and checked-in scaffold templates (repo-authored files).

### Allowed / expected (CLI orchestrator)

- Create and maintain repo-owned project skeletons (module dirs, `go.mod`/`pyproject.toml`, Dockerfiles, `cmd/` entrypoints, READMEs/checklists).
- Drive multi-step workflows: run `buf lint`/`buf breaking`/`buf generate`, run tests, build images, wire GitOps manifests/CR templates, and open PRs.
- Provide convenience wrappers (e.g., `ameide dev check`) that run the same checks CI runs, without becoming the canonical gate.

### Not allowed (CLI orchestrator)

- Re-implement proto parsing/templating that Buf plugins already provide (no bespoke “generator pipeline” in the CLI).
- “Sync” generated code by patching files itself; regeneration is `buf generate` + regen-diff in CI.
- Write generated artifacts into repo-owned roots (keep `clean: true` safe).

### Canonical gate

- CI remains authoritative: regen-diff + compile/tests are the drift guardrails. The CLI may wrap them locally, but it does not introduce a separate “verify CLI gate”.

## Scope

Included:
- `packages/ameide_core_cli/**` command behavior and templates
- Repo layout conventions created/updated by the CLI (checked-in files)
- GitOps wiring automation (writing to `gitops/ameide-gitops/**` and bumping submodule pointers)
- “One-command” developer loops that call `buf generate`, tests, builds, and probes

Not included:
- Protoc/Buƒ plugin implementation changes (see 521)
- Operator reconcile behavior (except where CLI behavior depends on it)

## How to use this log

When you change CLI orchestration behavior, add a new entry under **Log** with:
- What changed and why
- Which files/paths are affected
- How it was verified (commands)
- Follow-ups (if intentionally incomplete)

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

### 2025-12-14 — Remove “human-owned” boundary language

Change:
- Replaced “human-owned” wording with clearer repo semantics:
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
- Updated CLI-facing docs/backlogs to use real, repo-valid proto paths (and removed the obsolete `484-ameide-cli-ORIGINAL-BACKUP.md` copy) so examples are runnable and don’t teach legacy layouts.
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

## Follow-ups (ideas)

- Add a dedicated CLI command that runs a full vertical loop (generate → build → push → GitOps sync → smoke), with clear separation between “repo actions” and “cluster actions”.
- Ensure CLI-generated projects default to the same naming conventions used by the v0 samples.
