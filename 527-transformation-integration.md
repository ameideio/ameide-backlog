# 527 Transformation — Integration Primitive Specification

**Status:** Draft (MCP adapter scaffold implemented; connectors pending)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Integration primitives** — adapters for external systems (git providers, CI/status surfaces, registries, interoperability bindings) used to realize and govern transformation initiatives.

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Integration), Application Services/Interfaces (external ports), Data Objects (integration payloads).
- **Out-of-scope layers:** Canonical domain state and portal UX.

## 1) Integration responsibilities

Scope identity (required everywhere): `{tenant_id, organization_id, repository_id}` (integration never invents alternate repository identifiers; no separate `graph_id`).

- External connectors:
    - GitHub/GitLab and container registries,
    - CI/status surfaces (required checks, artifacts) for merge-time quality gates.
- Strict idempotency for inbound webhooks and retries.
- Emit intents/commands into domains (never facts); domains remain the only fact emitters.

Work items are **Transformation-domain owned** (initiative/work/work request). External ticketing systems are out of scope for v1.

### 1.0 CLI posture (tool, not orchestrator)

The CLI is not the delivery process. In an IT4IT-aligned posture:

- **Process primitives** are the orchestrators (execute promoted ProcessDefinitions; emit process facts).
- **Integrations** provide “runner” capabilities for external side effects, including invoking deterministic tools (CLI, `buf`, test runners) in a controlled environment.
- The CLI remains a **tool** used by workflow steps (and by humans locally), not a long-lived service boundary.

Implications:

- Any CLI invocation that matters for promotion must produce structured outputs that can be recorded as evidence (attachments) and linked to the governing baseline/definition promotion.
- The Process primitive is responsible for sequencing (preflight → scaffold → generate → verify → promote → deploy), retries, and emitting process facts; the Integration primitive is responsible for actually running external commands and capturing outputs.
  - CI can still exist as a repo-level quality gate for PR merges, but it is not the canonical substrate for WorkRequest execution in v1 (WorkRequests run as KEDA-scheduled Jobs).

### 1.0.1 “Runner” responsibilities (scaffolding/codegen/verify)

Integration runners exist to bridge “definition-driven process steps” to deterministic tool execution:

- **Scaffold:** run deterministic scaffolding from `packages/ameide_coding_helpers/scaffold` (invoked via the repo’s `ameide` CLI) to create repo-owned primitive skeletons (Domain/Process/Projection/Integration/UISurface/Agent) driven by promoted decomposition/scaffolding plans.
- **Generate:** run `buf generate` (SDKs/stubs) deterministically from protos/templates.
- **Verify:** run `ameide primitive verify` (and tests/build/lint as required) and return machine-readable results for process gates.

The runner does not decide policy; it executes tools, captures outputs, and emits intents/commands when external systems must be updated (Git provider/registry/CI), while Domain/Process facts remain the authoritative evidence streams.

### 1.0.2 Runner interface contract (normative; v1)

Integration runners exist so deterministic tool steps can run without turning the CLI (or a scaling layer) into the orchestrator.

**Inputs (minimum; required)**

- Scope: `{tenant_id, organization_id, repository_id}`
- Traceability: `process_instance_id`, `activity_id` (or step name), `correlation_id` (+ `causation_id` when available)
- Work identity:
  - `work_request_id` (stable)
  - `client_request_id` / `idempotency_key` (stable; used for end-to-end dedupe across retries)
- Repo checkout:
  - `repo_url` (or equivalent fetch reference)
  - `commit_sha` (exact revision)
  - `workdir` (path inside checkout)
- Plan reference:
  - `scaffolding_plan_ref` → a promoted `ScaffoldingPlanDefinition` version (Definition Registry id + version), or a fetched plan artifact path provided by the Process primitive
- Requested action:
  - `action_kind` ∈ `{preflight, scaffold, generate, verify, build, publish, deploy, smoke}` (v1 set; expand later)
  - `execution_scope` ∈ `{slice, repo}` (see §1.0.4)
- Idempotency:
  - `idempotency_key` (stable per WorkRequest) so replays do not duplicate side effects (e.g., “open PR”)

**Outputs (minimum; required)**

- `tool_run_evidence` descriptor (machine-readable; see §1.0.5) that the Process primitive can:
  - attach as evidence,
  - cite in `ToolRunRecorded`,
  - and show in projection-backed audit timelines.

**Non-goals**

- The runner MUST NOT emit domain facts.
- The runner MUST NOT become a policy engine (approval logic, promotion decisions, contract rules).
- The runner MUST NOT infer scope identifiers (it is always called with explicit `{tenant_id, organization_id, repository_id}`).

### 1.0.3 Runner execution backend (normative): KEDA ScaledJob → devcontainer-derived Jobs

In v1, long-running tool runs execute via **queue-driven ephemeral execution** in Kubernetes:

- A KEDA `ScaledJob` consumes `WorkRequested` facts and schedules a **Kubernetes Job per message** (one WorkRequest per Job).
- The Job executes the requested tool run, produces evidence artifacts, and records outcomes back into Domain via a Domain intent.

Hard rule (v1): queue-triggered Jobs MUST consume **WorkRequested** facts (explicitly requested by Process/Domain), not raw external webhooks/events.

#### Runtime image (normative): `.devcontainer` parity

KEDA-scheduled Jobs MUST run a runtime image derived from the repo’s `.devcontainer` toolchain so local (developer-mode) and platform (job-mode) executions use the same pinned environment:

- Base image is built from `.devcontainer/Dockerfile` (or an equivalent published image that is provably derived from it).
- The runtime image MUST include the repo-native `ameide` CLI (which embeds `packages/ameide_coding_helpers/**` for `doctor`/`verify`/`generate`/`scaffold`; see `backlog/558-ameide-coding-helpers.md`).
- The Job entrypoint executes the runner/agent work loop (checkout repo at `commit_sha` → run tool(s) → upload artifacts → record evidence/outcome in Domain).
- Toolchain pins that matter for evidence (e.g., Codex CLI) are inherited from the devcontainer pin (see `backlog/433-codex-cli-057.md`).

This does not imply “VS Code devcontainer” in-cluster; it means the **same containerized toolchain** is used for deterministic runs.

Operational constraints (normative defaults; tune per capability):

- `parallelism = 1`, `completions = 1` (one WorkRequest per Job)
- `activeDeadlineSeconds` set (hard timeout)
- `backoffLimit` set (bounded retries)
- history limits for completed/failed Jobs
- optional `ttlSecondsAfterFinished` for cleanup (not correctness)

Idempotency + ack discipline (required):

- Treat the queue as **at-least-once** delivery: duplicates are normal.
- A Job MUST NOT ack/delete its message until it has durably recorded the outcome in Domain (idempotently) and persisted/linked evidence artifacts.
- Domain MUST treat `client_request_id` as an idempotency key for result recording so duplicate Job runs converge to one canonical outcome record.

Recommendation: use “one queue per role/class of work” (e.g., `toolrun.verify`, `toolrun.generate`, `agentwork.coder`) so Kubernetes ServiceAccounts and external credentials can be scoped tightly per workload.

### 1.0.4 Execution scopes (verify posture; v1)

Runners must support two explicit execution scopes so “work on one slice” does not silently depend on unrelated repo health:

- `execution_scope = slice`:
  - run checks only for the target primitive/cooperation implied by the plan (or the working directory)
  - report repo-wide health issues as **blocked-by** (informational) unless the workflow step is explicitly a repo gate
- `execution_scope = repo`:
  - run repo-wide gates (buf lint/breaking/codegen drift across the repo) and block on failures

The ProcessDefinition decides which scope applies to each step; the runner only executes what it is instructed to run.

### 1.0.5 Tool-run evidence descriptor (maps to `ToolRunRecorded`)

Runners must return a stable evidence descriptor that can be attached and referenced by process facts (`ToolRunRecorded`) without scraping logs.

Minimum required fields (conceptual; keep stable and diff-friendly):

- Identity and scope: `{tenant_id, organization_id, repository_id}`, `process_instance_id`, `activity_id`, `attempt`
- Work identity: `work_request_id`, `idempotency_key`
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

### 1.0.6 Verification output posture (agent-grade; “no band aids”)

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

### 1.0.7 Tool identity and “doctor” posture

Runners must record tool identity deterministically:

- prefer a single canonical repo tool entrypoint (e.g., `bin/ameide`)
- capture `tool_version`/`tool_git_sha` for every run
- run a `preflight/doctor` step early in workflows to avoid “wrong binary” / “missing tool” cascades (the process decides; the runner executes)

### 1.0.7 Git provider enforcement posture (agentic coding)

For coding guardrails, the git provider (e.g., GitHub) is the **hard enforcement plane**. Transformation policies must be expressible as server-side controls, not “local rules”.

- **Protected branches are PR-only:** merges require checks/reviews; agents cannot push directly.
- **Branch namespaces are identity-scoped:** agent credentials can only push to their own `agent/<role>/<agent_id>/*` (or equivalent) namespace; switching branches locally must not grant new write authority.
- **Sensitive paths are server-blocked:** push rulesets block `.github/workflows/**`, `CODEOWNERS`, and infra/release directories except for explicit human bypass actors.
- **Evidence is captured:** runner outputs SHOULD include the relevant git provider policy snapshot (required checks, ruleset ids, branch namespace) so approvals and audits can prove “what was possible”.

### 1.0.8 Transport bindings: CloudEvents (optional; boundary-only)

Canonical semantics for Ameide EDA remain proto-defined envelopes + topic families (per `backlog/496-eda-principles.md`). CloudEvents is supported only as an optional **transport binding** at Integration boundaries (webhooks, external audit/event buses, partner integrations).

Normative mapping (canonical → CloudEvents v1.0):

- `message_id` → `id`
- producer identity → `source` (URI-reference; recommend controlled URNs like `urn:ameide:<kind>:<name>`)
- stable message type (`event_fact.type` or equivalent) → `type`
- subject identity → `subject`
- occurred timestamp → `time`
- payload → `data` with `datacontenttype` (`application/protobuf` or `application/json`)
- schema reference → `dataschema` (optional)

Ameide-required invariants that do not fit core CloudEvents attributes MUST be carried as extension attributes (lowercase alphanumeric; no underscores). Minimum recommended set:

- `tenantid`, `orgid`, `repositoryid`
- `correlationid`, `causationid`
- `traceparent` (and optional `tracestate`)

**Execution substrate (coding/tool runs):**

- Prefer a single “executor toolchain” image/profile for all runner executions (so verification results are comparable across CI/in-cluster/local).
- If the runner uses an LLM coding backend (e.g., Codex CLI), pin and record the version as part of `tool_run_evidence` (see Codex pin rationale in `backlog/433-codex-cli-057.md`).

**Metamodel interoperability:** import/export and downstream tool integration is profile-driven:

- Standards-compliant profiles (e.g., ArchiMate 3.2, BPMN 2.0) export in standard formats and must validate that the source element graph uses only standard `type_key` namespaces.
- Extended profiles may export custom bundles, but integrations must label the profile used so consumers can opt in explicitly.

## 1.1) Implementation progress (repo snapshot)

Delivered (scaffold):

- [x] MCP adapter scaffold exists at `primitives/integration/transformation-mcp-adapter` (transport skeleton + tests).
- [x] Scaffold tests run: `cd primitives/integration/transformation-mcp-adapter && go test ./...`.
- [x] Repo guardrails run for Integration primitives (shape + checks): `bin/ameide primitive verify --kind integration --name transformation-mcp-adapter --mode repo` (warnings may remain until transport/security conformance is complete).

Not yet delivered (integration meaning):

- [ ] Git/CI/tool runners (scaffold/generate/verify/build/publish) that return structured evidence for `ToolRunRecorded`.
- [ ] MCP tool/resource exposure derived from proto annotations (default deny; generated schemas).
- [ ] Evidence bundle production + REFERENCE linking into promotions/baselines.

## 1.2) Clarification requests (next steps)

Confirm/decide:

- Which external connectors are in the v1 acceptance slice (Git provider, registry, CI status) and what “evidence bundle” minimally contains (logs, links, elements, attestations).
- Whether integration-side facts (`transformation.integration.facts.v1`) are needed vs relying solely on domain/process facts for outcomes.
- The required MCP adapter posture for v1 (which tools/resources are exposed, what auth scopes are required, and what is query-vs-command split).

## 2) Acceptance criteria

1. Integrations are idempotent and do not emit domain facts.
2. External side effects are bounded and audit-friendly (evidence bundles).
