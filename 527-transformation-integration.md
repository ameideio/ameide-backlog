# 527 Transformation — Integration Primitive Specification

**Status:** Draft (MCP adapter scaffold implemented; connectors pending)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Integration primitives** — adapters for external systems (git/CI/scaffolding/ticketing) used to realize and govern transformation initiatives.

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Integration), Application Services/Interfaces (external ports), Data Objects (integration payloads).
- **Out-of-scope layers:** Canonical domain state and portal UX.

## 1) Integration responsibilities

Scope identity (required everywhere): `{tenant_id, organization_id, repository_id}` (integration never invents alternate repository identifiers; no separate `graph_id`).

- External connectors:
  - Backstage scaffolder / scaffolding runners,
  - GitHub/GitLab, CI systems, container registries,
  - ticketing/work item systems where required.
- Strict idempotency for inbound webhooks and retries.
- Emit intents/commands into domains (never facts); domains remain the only fact emitters.

### 1.0 CLI posture (tool, not orchestrator)

The CLI is not the delivery process. In an IT4IT-aligned posture:

- **Process primitives** are the orchestrators (execute promoted ProcessDefinitions; emit process facts).
- **Integrations** provide “runner” capabilities for external side effects, including invoking deterministic tools (CLI, `buf`, test runners) in a controlled environment (CI-like or in-cluster job).
- The CLI remains a **tool** used by workflow steps (and by humans locally), not a long-lived service boundary.

Implications:

- Any CLI invocation that matters for promotion must produce structured outputs that can be recorded as evidence (attachments) and linked to the governing baseline/definition promotion.
- The Process primitive is responsible for sequencing (preflight → scaffold → generate → verify → promote → deploy), retries, and emitting process facts; the Integration primitive is responsible for actually running external commands and capturing outputs.

### 1.0.1 “Runner” responsibilities (scaffolding/codegen/verify)

Integration runners exist to bridge “definition-driven process steps” to deterministic tool execution:

- **Scaffold:** run existing CLI scaffolding to create repo-owned primitive skeletons (Domain/Process/Projection/Integration/UISurface/Agent) driven by promoted decomposition/scaffolding plans.
- **Generate:** run `buf generate` (SDKs/stubs) deterministically from protos/templates.
- **Verify:** run `ameide primitive verify` (and tests/build/lint as required) and return machine-readable results for process gates.

The runner does not decide policy; it executes tools, captures outputs, and emits intents/commands when external systems must be updated (GitOps/CI/ticketing), while Domain/Process facts remain the authoritative evidence streams.

### 1.0.2 Runner interface contract (normative; v1)

Integration runners exist so the Process primitive can invoke deterministic tool steps without turning the CLI into an orchestrator.

**Inputs (minimum; required)**

- Scope: `{tenant_id, organization_id, repository_id}`
- Traceability: `process_instance_id`, `activity_id` (or step name), `correlation_id` (+ `causation_id` when available)
- Repo checkout:
  - `repo_url` (or equivalent fetch reference)
  - `commit_sha` (exact revision)
  - `workdir` (path inside checkout)
- Plan reference:
  - `scaffolding_plan_ref` → a promoted `ScaffoldingPlanDefinition` version (Definition Registry id + version), or a fetched plan artifact path provided by the Process primitive
- Requested action:
  - `action_kind` ∈ `{preflight, scaffold, generate, verify, build, publish, deploy, smoke}` (v1 set; expand later)
  - `execution_scope` ∈ `{slice, repo}` (see §1.0.3)
- Idempotency:
  - `idempotency_key` (stable per {process_instance_id, activity_id, attempt}) so replays do not duplicate side effects (e.g., “open PR”)

**Outputs (minimum; required)**

- `tool_run_evidence` descriptor (machine-readable; see §1.0.4) that the Process primitive can:
  - attach as evidence,
  - cite in `ToolRunRecorded`,
  - and show in projection-backed audit timelines.

**Non-goals**

- The runner MUST NOT emit domain facts.
- The runner MUST NOT become a policy engine (approval logic, promotion decisions, contract rules).
- The runner MUST NOT infer scope identifiers (it is always called with explicit `{tenant_id, organization_id, repository_id}`).

### 1.0.3 Execution scopes (verify posture; v1)

Runners must support two explicit execution scopes so “work on one slice” does not silently depend on unrelated repo health:

- `execution_scope = slice`:
  - run checks only for the target primitive/cooperation implied by the plan (or the working directory)
  - report repo-wide health issues as **blocked-by** (informational) unless the workflow step is explicitly a repo gate
- `execution_scope = repo`:
  - run repo-wide gates (buf lint/breaking/codegen drift across the repo) and block on failures

The ProcessDefinition decides which scope applies to each step; the runner only executes what it is instructed to run.

### 1.0.4 Tool-run evidence descriptor (maps to `ToolRunRecorded`)

Runners must return a stable evidence descriptor that can be attached and referenced by process facts (`ToolRunRecorded`) without scraping logs.

Minimum required fields (conceptual; keep stable and diff-friendly):

- Identity and scope: `{tenant_id, organization_id, repository_id}`, `process_instance_id`, `activity_id`, `attempt`
- Tool identity:
  - `tool_name` (e.g., `bin/ameide`, `buf`, `go test`, `pnpm`)
  - `tool_version` (and `tool_git_sha` if applicable)
- Invocation:
  - `argv` (redact secrets)
  - `workdir`
  - `started_at`, `finished_at`
- Result:
  - `exit_code`
  - `status` ∈ `{success, failed, skipped}`
  - `summary` (short; human-readable)
- Outputs:
  - `artifacts[]` (paths/URIs to logs, reports, generated bundles)
  - `check_results[]` (optional; structured failures with “what/why/fix”, used to drive agent next steps)

The Process primitive uses this descriptor to emit `ToolRunRecorded` and to link evidence attachments to the relevant baseline/definition promotion (via REFERENCE relationships).

### 1.0.5 Verification output posture (agent-grade; “no band aids”)

When the runner executes verification (`action_kind = verify`), the returned `tool_run_evidence` SHOULD include structured, actionable failures (not just raw logs) so agents and humans can repair the correct layer.

Recommended failure classification (v1):

- `policy_violation`: repo/primitive violates an invariant (fix implementation/contracts)
- `blocked_by_repo_health`: failure is outside the current slice (fix repo health or rerun with `execution_scope = slice`)
- `guardrail_bug_suspected`: checker appears inconsistent with the policy intent (fix the checker/guardrail, not the repo to satisfy brittle conventions)

Each failure SHOULD include:

- `what_failed` (precise)
- `why_it_matters` (policy intent)
- `where_to_fix` ∈ `{design, proto, implementation, ops, guardrail}`
- `fix_steps[]` (concrete commands/edits)

Cross-reference: external verification baseline for test expectations lives in `backlog/521f-external-verification-baseline.md`.

### 1.0.6 Tool identity and “doctor” posture

Runners must record tool identity deterministically:

- prefer a single canonical repo tool entrypoint (e.g., `bin/ameide`)
- capture `tool_version`/`tool_git_sha` for every run
- run a `preflight/doctor` step early in workflows to avoid “wrong binary” / “missing tool” cascades (the process decides; the runner executes)

**Metamodel interoperability:** import/export and downstream tool integration is profile-driven:

- Standards-compliant profiles (e.g., ArchiMate 3.2, BPMN 2.0) export in standard formats and must validate that the source element graph uses only standard `type_key` namespaces.
- Extended profiles may export custom bundles, but integrations must label the profile used so consumers can opt in explicitly.

## 1.1) Implementation progress (repo snapshot)

Delivered (scaffold):

- [x] MCP adapter scaffold exists at `primitives/integration/transformation-mcp-adapter` (transport skeleton + tests).
- [x] Scaffold tests run: `cd primitives/integration/transformation-mcp-adapter && go test ./...`.
- [ ] Gate parity: `bin/ameide primitive verify` does not yet support `--kind integration` (CLI guardrails for Integration are pending).

Not yet delivered (integration meaning):

- [ ] Git/CI/tool runners (scaffold/generate/verify/build/publish) that return structured evidence for `ToolRunRecorded`.
- [ ] MCP tool/resource exposure derived from proto annotations (default deny; generated schemas).
- [ ] Evidence bundle production + REFERENCE linking into promotions/baselines.

## 1.2) Clarification requests (next steps)

Confirm/decide:

- Which external connectors are in the v1 acceptance slice (Git provider, CI, ticketing) and what “evidence bundle” minimally contains (logs, links, elements, attestations).
- Whether integration-side facts (`transformation.integration.facts.v1`) are needed vs relying solely on domain/process facts for outcomes.
- The required MCP adapter posture for v1 (which tools/resources are exposed, what auth scopes are required, and what is query-vs-command split).

## 2) Acceptance criteria

1. Integrations are idempotent and do not emit domain facts.
2. External side effects are bounded and audit-friendly (evidence bundles).
