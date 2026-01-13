# 484 – Ameide CLI Overview & Index

> **Deprecation notice (520):** Any guidance that treats the CLI as the **canonical generator/scaffolder** is deprecated. Canonical v2 uses **`buf generate`** (pinned plugins, deterministic outputs, generated-only roots, regen-diff CI gate); a CLI may exist only as orchestration/reporting around standard gates. See `backlog/520-primitives-stack-v2.md`.

**Status:** Active
**Audience:** AI agents (primary), developers (secondary)
**Scope:** Proto-aligned CLI for agentic development on Ameide primitives

> This is the **landing page** for the Ameide CLI documentation. For detailed specifications, see the linked sub-documents.

---

## 1. Purpose

The **Ameide CLI** is a proto-aligned command-line interface that serves as **guardrails for AI agents** doing autonomous development. It is designed to be consumed by:

- **AI Agents** (primary) – automating development work via structured JSON output
- **Developers** (secondary) – running commands interactively when needed

---

## 2. Design Philosophy: Shape, Not Meaning

The CLI knows **shape** (repo layout, proto structure, test conventions) but not **meaning** (business logic, validation rules, domain invariants). This separation is critical:

| CLI Owns (Shape) | Agent Owns (Meaning) |
|------------------|----------------------|
| Directory layout per primitive kind | Business logic & invariants |
| Empty handler/test skeletons | Test assertions & fixtures |
| Proto/SDK freshness checks | What the proto should contain |
| Convention enforcement | How to implement requirements |
| GitOps manifest structure | When/what to deploy |

---

## 3. Three Responsibilities

1. **Describe** – Introspect repo state, detect drift, report what exists vs what's expected
2. **Verify** – Run checks (tests, lint, breaking changes, EDA principles) and report structured pass/fail
3. **Scaffold** – Generate mechanical, proto-driven skeletons (optional, conservative)

The CLI does **no fuzzy reasoning**. All non-determinism lives in agents.

---

## 4. TDD as the Core Development Model

The CLI is built around **Test-Driven Development** principles:

```
RED → GREEN → REFACTOR
```

| Phase | What Happens | CLI Role |
|-------|--------------|----------|
| **RED** | Write failing test first | `plan` outputs test specs; `scaffold` creates failing test stubs |
| **GREEN** | Implement minimal code to pass | `verify` checks if test passes |
| **REFACTOR** | Clean up without breaking | `verify` ensures no regression |

**Key invariants:**
- Tests are **never optional** – every RPC must have tests
- Tests must **fail before implementation** – scaffolded tests call real handlers and fail
- Implementation is **test-driven** – agent writes code to make tests pass
- All tests must pass **before commit** – `verify --all` gates the workflow

> **Details**: See [484a – Primitive Workflows & TDD](484a-ameide-cli-primitive-workflows.md) for the full agentic development sequence.

---

## 5. Core Principles

### 5.1 Proto-Aligned, Like SDKs

The CLI follows the same proto package structure as the SDKs:

```bash
ameide <proto-package> <action> [flags]
```

Examples:
```bash
ameide workflows list              # → ameide_core_proto.workflows_runtime.v1
ameide graph query                 # → ameide_core_proto.graph.v1
ameide primitive plan              # → ameide_core_proto.primitive.v1
ameide primitive verify            # → ameide_core_proto.primitive.v1
```

### 5.2 Dual Output: Agent-First

Every command emits:

1. **Structured JSON** (with `--json` flag) – Primary for AI agent consumption
2. **Human-readable** (default stdout) – Secondary for developer terminal use

AI agents should always use `--json`.

### 5.3 Local-First, Service-Optional

| Mode | Description |
|------|-------------|
| **Local** | Works on filesystem, runs tests, parses protos directly |
| **Service** | Connects to Ameide services via gRPC/Connect (future) |

> **Details**: See [484b – Proto Contract](484b-ameide-cli-proto-contract.md) for message shapes and service alignment.

---

## 6. Command Taxonomy

### 6.1 Service Commands (gRPC clients)

Connect to Ameide services:

```bash
ameide workflows list [--status pending|running|completed]
ameide graph query <query>
ameide events subscribe [--types <types>]
```

### 6.2 Primitive Commands (local, for agentic development)

Operate on local filesystem for AI-assisted development:

| Command | Purpose |
|---------|---------|
| `describe` | What exists, what's expected, what's the delta |
| `drift` | SDK staleness, missing tests, convention issues |
| `plan` | What work is needed for a primitive |
| `impact` | What consumers would be affected by a proto change |
| `verify` | Run tests/checks, report structured pass/fail |
| `scaffold` | Generate empty skeletons (optional, conservative) |

> **Details**: See [484a – Primitive Workflows & TDD](484a-ameide-cli-primitive-workflows.md) for command details and examples.

### 6.3 Config Commands

```bash
ameide config init       # Create ~/.ameide/config.yaml
ameide config show       # Display current configuration
ameide config set <k> <v> # Update config value
```

---

## 7. Agentic Development Flow (Summary)

```
┌─────────────────────────────────────────────────────────────────┐
│  1. OBSERVE                                                     │
│     └─ describe, drift → understand current state               │
│                                                                 │
│  2. REASON                                                      │
│     └─ plan, impact → research & form hypothesis                │
│                                                                 │
│  3. HUMAN GATE (if significant)                                 │
│     └─ Present findings, get approval                           │
│                                                                 │
│  4. ACT                                                         │
│     └─ scaffold, implement, verify (TDD loop)                   │
│                                                                 │
│  5. HUMAN UAT                                                   │
│     └─ Final acceptance before merge                            │
└─────────────────────────────────────────────────────────────────┘
```

> **Details**: See [484a – Primitive Workflows & TDD §5](484a-ameide-cli-primitive-workflows.md#5-intelligent-agent-sequence) for the complete sequence.

---

## 8. Document Index

| Document | Scope |
|----------|-------|
| **[484a – Primitive Workflows & TDD](484a-ameide-cli-primitive-workflows.md)** | Primitive commands, TDD loop, agentic development sequence, verify checks, cascade verification |
| **[484b – Proto Contract](484b-ameide-cli-proto-contract.md)** | `ameide_core_proto.primitive.v1` message definitions, local vs service mode |
| **[484c – Repo & GitOps Alignment](484c-ameide-cli-repo-gitops.md)** | Core vs GitOps repo split, directory structure, test infrastructure (430), GitOps alignment (434) |
| **[484d – Migration & Phases](484d-ameide-cli-migration.md)** | Legacy command migration, phased implementation plan |
| **[484e – Industry Patterns](484e-ameide-cli-industry-patterns.md)** | Backstage, Buf, Kubebuilder alignment; what we don't do (JHipster) |
| **[484f – Scaffold Implementation](484f-ameide-cli-scaffold-implementation.md)** | Per-primitive scaffold outputs, templates, worked examples, cross-reference index |

---

## 9. Implementation Status (2024-05)

- **Repo-first workflows delivered** – `primitive describe` now emits `repo_path`, `proto_path`, `drift`, and `expected_but_missing` so agents understand local shape and GitOps gaps without Kubernetes access.
- **Guardrail-rich verify** – `verify --mode repo` executes the documented security/EDA checks (gitleaks, govulncheck, Semgrep, event coverage) alongside naming/tests before touching the cluster, surfaces Buf lint/breaking status directly in the JSON, and `--cascade --proto-path` fans out into Go, npm, and pytest suites for every detected consumer.
- **Cross-SDK drift + impact** – Proto drift and impact analysis now cover Go, TypeScript, and Python artifacts/consumers, ensuring downstream awareness beyond Go services.
- **Scaffolding coverage expanded** – All primitive kinds can be scaffolded across Go/TypeScript/Python; outputs include Dockerfiles, Backstage `catalog-info.yaml`, GitOps manifests respecting `--gitops-root`, language-aware integration harnesses, and intentionally failing tests per RPC so agents follow RED→GREEN→REFACTOR regardless of language.
- **Next focus** – Wire Buf/cascade summaries into cluster-mode verification, tighten npm/pytest dependency bootstrapping inside the integration harness, and extend cascade signals into future non-code consumers (docs, SDK publish workflows).

### 9.1 Cascade Roadmap

- **Non-code consumers** – `verify --cascade` now emits `language=workflow` entries for proto/SDK workflows (e.g., `scripts/ci/check_proto_sdk_regen.sh`, `scripts/check_ameide_client_manifest.sh`, `release/verify_{ts,python,go}_sdk.sh`, `release/verify_docs.sh`) so agents run the same checks SDK/doc pipelines expect.
- **Dependency awareness** – Repo + cluster modes now share Buf summaries, per-consumer language hints, and dependency bootstrap guidance (npm install / uv sync) so agents know whether to install tooling or re-run cascade.
- **Pipeline hand-off** – Workflow cascade entries identify the exact script + directory the automation runs, prepping the future GitHub workflow hand-off once we wire docs/publish consumers into the same map.

### 9.2 Known Gaps

- **Docs beyond SUMMARY** – `release/verify_docs.sh` only enforces the SUMMARY.md snapshots. The technical writer workflows (MkDocs/Next.js/Storybook) are still outside of `verify --cascade`, so doc builds can drift until the frontdoor scripts expose a deterministic entry point.
- **Workflow status provenance** – Workflow cascades currently run local shell scripts; we still need to emit the associated GitHub workflow IDs/targets so CI/CD can selectively retry or parallelize them without re-running the full verify command.

## 10. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [365-buf-sdks-v2](365-buf-sdks-v2.md) | SDK regeneration, Buf breaking rules |
| [430v2 unified test infrastructure](430-unified-test-infrastructure-v2.md) | Test phases and evidence contract (JUnit) |
| [434-unified-environment-naming](434-unified-environment-naming.md) | GitOps structure, namespace labels |
| [470-ameide-vision](470-ameide-vision.md) | Core invariants including EDA (§8-13) |
| [472-ameide-information-application](472-ameide-information-application.md) | CQRS, command/event patterns |
| [477-primitive-stack](477-primitive-stack.md) | Repo layout, operator structure |
| [496-eda-principles-v2](496-eda-principles-v2.md) | Full EDA reference |
