# 484d – Ameide CLI: Migration & Phased Implementation

> **Deprecation notice (520):** This migration plan assumes a CLI scaffolder as the primary generator; that approach is deprecated. Canonical v2 uses **`buf generate`** (pinned plugins, deterministic outputs, generated-only roots, regen-diff CI gate). See `backlog/520-primitives-stack-v2.md`.

> **Update (2026-01, 670):** GitOps wiring is authored via a CI-owned workflow in `ameide-gitops` (workflow → PR → merge).  
> Any “CLI writes GitOps files” phase is superseded; the CLI should trigger the GitOps workflow instead. See `backlog/670-gitops-authoritative-write-path-for-scaffolding.md`.

**Status:** Active
**Audience:** Platform engineers, CLI implementers
**Scope:** Legacy command migration, phased implementation plan

> **Parent document**: [484 – Ameide CLI Overview](484-ameide-cli.md)

> **Update (2026-01): testing contract is 430v2**
>
> Treat `backlog/430-unified-test-infrastructure-v2-target.md` as the normative repo contract for test phases and JUnit evidence. Legacy “integration packs / `INTEGRATION_MODE`” guidance should be considered historical only.

---

## 1. Current State & Migration

The existing `packages/ameide_core_cli` has legacy commands:

| Command | Status | Action |
|---------|--------|--------|
| `ameide dev` | Legacy | Remove (use Tilt directly) |
| `ameide generate` | Legacy | Remove (use `buf generate`) |
| `ameide model` | Legacy | Remove (stubbed, incomplete) |
| `ameide command` | Legacy | Remove (stubbed) |
| `ameide query` | Legacy | Remove (stubbed) |
| `ameide events` | Migrate | → `ameide events *` |
| `ameide workflows` | Migrate | → `ameide workflows *` |
| `ameide config` | Keep | Still useful |
| `ameide primitive *` | **New** | Implement |

---

## 2. Phased Implementation

Rather than building the full CLI at once, we implement in phases that deliver value incrementally.

### Phase 1: Describe & Verify Only (MVP)

**Goal:** Give agents visibility into repo state without any generation.

**Commands:**
```bash
ameide primitive describe --json    # What exists, what's expected, what's the delta
ameide primitive drift --json       # SDK staleness, test coverage gaps, convention issues
ameide primitive impact --json      # What consumers would be affected by a proto change
ameide primitive verify --json      # Run tests, lint, breaking checks
```

**What's NOT included:**
- `scaffold` command (agents create files manually)
- GitOps generation
- Any file mutations

**Value:** Agents can reason about the codebase with structured data. They write code themselves but have guardrails to validate correctness.

**Verification checks included:**
- Test execution (pass/fail)
- Lint results (buf-lint, golangci-lint, eslint)
- Security scans (secret scan, dependency vulns)
- Command/event discipline (RPC naming, forbidden prefixes)
- EDA reliability (outbox wiring, idempotency guards, tenant validation)

### Phase 2: Conservative Scaffolding

**Goal:** Generate safe, mechanical structures that agents would otherwise write identically – with **TDD-ready failing tests**.

**Commands:**
```bash
ameide primitive scaffold --kind domain --name orders --lang go --json
# Generates:
#   - Directory structure
#   - Handler files returning `codes.Unimplemented`
#   - Test files that CALL handlers and FAIL (proper RED state)
#   - Dockerfile, go.mod, README stub
```

**Flags:**
- GitOps wiring is not generated directly by the CLI as the canonical path; it is authored via the GitOps repo workflow (670).
- `--dry-run` – Show what would be created without writing

**TDD alignment:**
- Handlers return `codes.Unimplemented` by default
- Tests call the handlers and assert on expected behavior (not empty)
- **Freshly scaffolded code FAILS all tests** → agent starts in RED state
- Agent's job: make tests GREEN by implementing handlers

**Rules:**
- **One-shot only**: If folder exists, refuse to scaffold
- **No business logic**: Handlers return unimplemented, not real logic
- **Tests must fail**: Scaffolded tests exercise the API and fail until implemented

### Phase 3: GitOps & Test Harness (Optional)

**Goal:** For teams that want more automation, add GitOps and test infrastructure wiring via PR-based workflows.

**Commands:**
```bash
ameide primitive gitops scaffold --kind <kind> --name <name> --version v0 --json
# Opens a PR in `ameide-gitops` that adds:
#   - `environments/_shared/components/**/component.yaml`
#   - `sources/values/**.yaml`
# per repo conventions (see 670).

ameide primitive scaffold --include-test-harness --json
# Adds:
#   - __mocks__/ directory with client stubs
#   - run_integration_tests.sh (430-compliant)
#   - Mode-aware helper files
```

**Both remain optional flags**, not defaults.

### Phase 4: Transformation Integration (Future)

**Goal:** CLI can query Transformation Domain for context.

**Commands:**
```bash
ameide transformation context --json  # What should exist per design artifacts
ameide transformation sync --json     # Compare design artifacts to repo state
```

**Value:** Agents get "what should be built" from Transformation, not just "what exists" from repo.

---

## 3. Implementation Timeline

| Phase | Scope | Complexity | Prerequisite |
|-------|-------|------------|--------------|
| 1 | describe, drift, impact, verify | Medium | Buf integration |
| 2 | scaffold (code only) | Low | Phase 1 |
| 3 | scaffold (gitops, test harness) | Low | Phase 2 |
| 4 | transformation context | High | Transformation APIs |

**Start with Phase 1.** It delivers the most value (visibility, guardrails) with the least risk (no mutations).

---

## 4. Implementation Notes

### 4.1 Technology Stack

- CLI is in Go, using Cobra for command structure
- JSON output uses proto-generated structs (or compatible shapes)
- `--json` flag switches output mode
- Config stored in `~/.ameide/config.yaml`
- Service commands use gRPC with tenant metadata
- Primitive commands are local filesystem operations

### 4.2 Proto Message Alignment

All JSON output matches `ameide_core_proto.primitive.v1` message shapes. This ensures:
- Consistent parsing for AI agents
- Future service mode compatibility
- Type-safe SDK generation

See [484b – Proto Contract](484b-ameide-cli-proto-contract.md) for full message definitions.

### 4.3 CI Integration

`verify` should be callable from CI:

```yaml
# .github/workflows/verify.yml
- name: Verify all primitives
  run: ameide primitive verify --all --json > verify-result.json

- name: Check for failures
  run: |
    if jq -e '.summary == "fail"' verify-result.json; then
      exit 1
    fi
```

### 4.4 Exit Codes

| Exit Code | Meaning |
|-----------|---------|
| 0 | All checks pass |
| 1 | One or more checks failed |
| 2 | CLI error (invalid flags, missing files) |

---

## 5. Migration Path for Agents

### Before CLI (Current State)

Agents must:
1. Parse repo manually to understand structure
2. Guess at conventions
3. Write code without structured validation
4. Hope tests pass

### After Phase 1 (describe/verify)

Agents can:
1. Run `describe` to understand current state
2. Run `drift` to identify gaps
3. Run `impact` to understand change scope
4. Write code manually
5. Run `verify` to validate correctness

### After Phase 2 (scaffold)

Agents can:
1. Run `scaffold --dry-run` to preview structure
2. Run `scaffold` to generate skeleton
3. Focus on implementing business logic
4. Run `verify` to validate

### After Phase 4 (transformation)

Agents can:
1. Query Transformation for design intent
2. Compare design to implementation
3. Generate code aligned with business requirements
4. Close the loop with UAT

---

## 6. Backwards Compatibility

### Existing Commands

| Command | Behavior During Migration |
|---------|---------------------------|
| `ameide dev` | Deprecated warning → removed in v2 |
| `ameide generate` | Deprecated warning → use `buf generate` |
| `ameide events` | Kept, migrated to new structure |
| `ameide workflows` | Kept, migrated to new structure |
| `ameide config` | Kept unchanged |

### Config File

Existing `~/.ameide/config.yaml` remains compatible. New fields added:

```yaml
# Existing
api:
  base_url: https://api.ameide.io

# New (optional)
primitive:
  repo_root: ~/ameide-core
  gitops_root: ~/ameide-gitops
  default_lang: go
```

---

## 7. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [365-buf-sdks-v2](365-buf-sdks-v2.md) | Buf integration, SDK regeneration |
| [430v2 unified test infrastructure](430-unified-test-infrastructure-v2.md) | Test phases and JUnit evidence |
| [471-ameide-business-architecture](471-ameide-business-architecture.md) | Transformation Domain as source of truth |
| [477-primitive-stack](477-primitive-stack.md) | Target repo layout |
