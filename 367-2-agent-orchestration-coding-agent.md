> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/367-2-agent-orchestration-coding-agent – Software Delivery Automation

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)

## Purpose
Document the specialization of automation for coding agents (e.g., Ameide's internal "Codex CLI" automation runtime—not related to OpenAI Codex) acting as Developers within any methodology profile. Focuses on repo edits, tests, previews, PRs, and feeding Increment evidence back to governance workflows.

- Branch/ workspace automation, dependency install, test execution, preview deployment + aliasing, PR creation, and failure handoff flows.
- **Auto-planning trigger:** when a WorkItem is promoted to backlog and meets DoR (including architect context links), the system auto-creates a default Coding Agent automation plan (based on repo adapter/profile defaults) so users don’t have to click “Create Plan.”
- **Auto-run:** once the plan is ready (and DoR satisfied), the executor automatically launches the Coding Agent run. Manual reruns remain available for retries/handoffs.
- Methodology-aware evidence packaging (e.g., Scrum Increment evidence vs. SAFe Iteration objective proof). Each run emits context (Sprint Goal, PI Objective, ADM Deliverable) so Transformation can attach outputs automatically and registers the outputs as `governance.v1.Attestation`(s) with checksums + provenance.
- Guardrails (allowed directories, command budgets, secret scopes), policy enforcement (DoR/DoD checks), and telemetry (command logs, success/failure reasons).
- Hooks for human takeovers (patch bundles, `.codex/followup.md`) with backlog linkage and impediment creation when automation can’t proceed.
- **Notifications:** requesters/subscribers receive notifications when coding/testing completes and when the preview deployment/alias is live; messages include the preview URL and evidence summary.

## Deliverables
1. **Agent runtime packaging** – Codex CLI container/image, repo adapters, overlay support, preview promotion automation.
2. **Plan/run schema extensions** – Capture methodology goal references, Increment evidence metadata, quality gates, automatic registration of governance `Attestation`/method impediment calls, and metadata indicating whether the run was auto-triggered vs manual.
3. **UI integration** – Run status streaming, diff previews, evidence checklist tied to DoD, quick handoff actions, and notification center entries (with preview URLs and run summaries) for subscribed users.
4. **Observability** – Success metrics, lead time from READY to DONE, policy violations, exception dashboards, and notification delivery stats.

## Non-goals
- EA or solution-level artifact generation (covered in other Stage 2 docs).

## Dependencies
- Generic scheduler infrastructure.
- Repo adapters + overlay catalogs.
- Methodology profiles for mapping evidence requirements.
- Graph metamodel from [backlog/300-ameide-metamodel.md](./300-ameide-metamodel.md) so automation outputs attach directly to backlog elements and relationships.

## Exit Criteria
- Coding agents can execute backlog work end-to-end, automatically attaching evidence to the relevant methodology artifacts (Increment, PI Objective, ADM deliverable) and respecting DoR/DoD policies.
- Failures raise impediments or reassign work with preserved context.
- LangChain v1 migration complete (runtime uses `create_agent`, OTEL traces show middleware).

## Implementation guide
### Runtime + workspace packaging
- Reuse the LangGraph runtime that already powers AI agents to package the Coding Agent image instead of building a bespoke runner. `services/agents_runtime/src/agents_service/runtime.py` composes Buf-generated catalog clients with `langchain.agents.create_agent` (per backlog/313) and drives executions via `LangGraphRuntime`; the corresponding worker bootstrapper in `services/agents_runtime/src/agents_service/main.py` wires cache, tool runtime, and telemetry. Add repo-adapter binaries, workspace mounts, and Codex CLI entrypoints to this image (and to `services/agents_runtime/Dockerfile*`) so coding runs, LangGraph executions, and plan orchestration ship together.
- Complete the LangChain v1 migration as part of this work: no legacy plan builders in `services/agents_runtime/src/agents_service/runtime.py`; OTEL traces must show middleware from `langchain.agents.create_agent`.
- Keep artifact packaging aligned with the Agents service. `services/agents/README.md` and the MinIO bootstrapper at `services/agents_runtime/scripts/bootstrap_artifacts.py` already define how compiled graphs and node metadata are published. Extend that pipeline so repo adapters and preview tooling are versioned artifacts referenced from agent instances, which lets the executor lock to specific tooling per methodology profile.
- Guardrails need to stay in lockstep with backlog/362. Use `scripts/vault/ensure-local-secrets.py` to register every new CLI/preview secret and `infra/kubernetes/scripts/validate-hardened-charts.sh` to prove the Helm overlays for `agents`, `agents-runtime`, `workflows`, and shared db-migrations fail fast when secrets/ExternalSecrets drift. This keeps Layer 15 as the single source of truth for bootstrap credentials even when repo adapters add new env vars.
- Distribute repo adapter helpers through the CLI packages so human and automated runs invoke the same tooling. `packages/ameide_core_cli` (see `internal/commands/workflow.go` and `internal/client`) already wraps Buf SDKs behind a single `ameide` binary. Ship git/test/preview helpers there and expose them via new subcommands so Codex CLI sessions and human reruns stay aligned with the Buf SDK policies described in backlog/365.

### Workflow plan/run orchestration
- The Workflow service (`services/workflows/src/grpc-handlers.ts`, `repositories/definitions-graph.ts`, `repositories/rules-graph.ts`) already persists definitions, versions, and rule bindings on top of the platform Postgres schema described in backlog/305. Extend those models so every Coding Agent plan/run stores methodology context (`timebox_ids`, `goal_ids`, `work_item_ids`, graph element ids) and repo-adapter defaults. Persist the richer metadata in the JSONB `spec`/`annotations` columns rather than inventing new tables so it remains queryable through the existing APIs.
- Temporal orchestration is concentrated inside `services/workflows/src/temporal/facade.ts` and the worker at `services/workflows_runtime/src/workflows_runtime/workflows.py`. Update `WorkflowRunPayload`, `GuardContext`, and `ActionBundle` (in `services/workflows_runtime/src/workflows_runtime/models.py`) to carry the methodology references, repo adapter settings, allowed commands, preview requirements, and DoR status. Surface the same fields as Temporal memo + search attributes so runs can be correlated back to elements/timeboxes per backlog/300 and backlog/303.
- Keep SDKs in sync whenever plan/run metadata changes. `packages/ameide_sdk_ts/src/client.ts`, `packages/ameide_sdk_go/client.go`, and `packages/ameide_sdk_python/src/ameide_sdk/client.py` already expose the WorkflowService stubs with shared interceptors (auth, retries, tracing). Regenerate them via `ameide generate sdk` whenever proto contracts gain new fields so Transformation, UI, CLI, and automation runners read/write identical plan payloads (per backlog/365).

### Repo/test automation & guardrails
- Wire the executor activities to real repo operations. `services/workflows_runtime/src/workflows_runtime/activities.py` validates definitions, enforces guard approvals, and executes repository actions, but `execute_graph_actions` currently just simulates failures. Replace the placeholder with adapters that call the Codex CLI / repo-specific scripts, run the install/test/preview phases, and capture diffs + logs per DoD. Feed allowable commands/secrets into each action via the metadata object so per-repo guardrails (paths, secret scopes, command budgets) are enforced uniformly.
- Capture patch bundles and workspace state on failure. `ActionBundle` already carries metadata; extend it so the executor zips the working tree and CLI transcripts into MinIO (same bucket strategy used for agent artifacts) and attaches object URIs to status updates for handoffs.
- Honor platform guardrails during automation. Ensure each worker pod applies the `namespace_guard.py` check, uses the same temporal task queues declared in `infra/kubernetes/charts/platform/workflows_runtime`, and sources secrets through the Layer 15 ExternalSecrets referenced earlier so repo adapters cannot bypass backlog/362 enforcement.

### UI + CLI integration
- The platform UI already surfaces workflow configuration and executions (`services/www_ameide_platform/features/workflows/components/WorkflowCatalog.tsx`, `ExecutionRuns.tsx`, `ElementWorkflowPanel.tsx`). Extend these components to show Coding Agent–specific metadata: repo adapter, DoR/DoD checklist, preview URLs, evidence counts, and impediment links. Data access goes through the shared React Query hooks in `services/www_ameide_platform/lib/api/hooks.ts`, which wrap the Workflow client retrieved via `hooks/useWorkflowClient.ts`; add new fields there once the SDK exposes them so every view stays tenant-aware (per backlog/329).
- Provide equivalent surfaces in automation tooling. `packages/ameide_core_cli/internal/commands/workflow.go` currently lists definitions and runs; add commands for “plan diff”, “trigger automation”, “download evidence”, and “handoff” so engineers can interact with Coding Agent runs from a terminal. Because the CLI uses the same Buf stubs as the UI, CLI additions automatically benefit downstream SDK consumers.
- Connect repo adapters to builders/operators. `services/workflows/src/repositories/rules-graph.ts` already maps workflow rules to `graph_id`/`transformation_id`, allowing repository and transformation settings pages (`services/www_ameide_platform/app/(app)/org/.../settings/workflows`) to show contextual rules. Coding Agent plans should register the default repo adapter + guardrails in those annotations so UI affordances (drop-downs, warnings) stay in sync with the executor.

### Evidence, notifications & observability
- Status telemetry already flows from the worker to the service (`services/workflows_runtime/src/workflows_runtime/callbacks.py` and `record_status_update`, plus `services/workflows/src/status-updates.ts` and `repositories/status-updates-graph.ts`). Extend the payload so every status update contains preview URLs, evidence blobs (URI + checksum per backlog/300), and impediment identifiers. This gives the UI/CLI enough context to show evidence checklists and failure diagnostics without scraping logs.
- Hook evidence storage into the graph/metamodel. Once runs emit `Evidence` descriptors, persist them through the element model described in backlog/303 by creating or linking `Element` rows for run outputs (documents, binaries) and relationships pointing back to the originating work items, timeboxes, and governance goals.
- Notifications should hang off the same pipeline. When `record_status_update` writes a final RUNNING→COMPLETED/FAILED transition, fan out to the notification service (or enqueue events) so subscribers receive preview URLs and evidence summaries automatically. Unit + integration tests should exercise the full lifecycle—definition → run → evidence attach → notification—using the existing suites in `services/workflows/__tests__/integration` and the UI Playwright specs.

## Open topics / criticalities
- **Repo/test execution still stubbed** – `execute_graph_actions` in `services/workflows_runtime/src/workflows_runtime/activities.py` only simulates failures by checking `simulate_failure` flags; it never invokes git, installs dependencies, runs tests, or calls preview CLIs, so Coding Agent runs cannot yet produce actual diffs or test evidence.
- **Methodology metadata missing from runs** – `WorkflowRunPayload` and `ActionBundle` (`services/workflows_runtime/src/workflows_runtime/models.py`) carry only workflow/execution ids, so `TemporalFacade` (`services/workflows/src/temporal/facade.ts`) can index runs by tenant/definition/element but has no notion of `timebox_ids`, `goal_ids`, or `work_item_ids` required by backlog/300/303. Evidence can’t be correlated to increments until those fields exist end-to-end.
- **No Attestation/Impediment persistence** – The Workflow service currently writes status rows to `workflows_execution_status_updates` via `services/workflows/src/status-updates.ts` and `repositories/status-updates-graph.ts`. There is no code that registers `governance.v1.Attestation`, `Dependency`, or method-level impediment records, so governance profiles (backlog/367-2-generic) cannot consume Coding Agent outputs yet.
- **Auto-plan triggers + CLI coverage incomplete** – `packages/ameide_core_cli/internal/commands/workflow.go` only lists definitions/runs and `startWorkflowRun` in `services/workflows/src/grpc-handlers.ts` requires explicit API calls. There is no automation that watches WorkItems/DoR (from backlog/367-5) and instantiates plans automatically, nor CLI commands to approve or rerun Coding Agent plans, so Stage 2 auto-planning is still manual.
- **Preview + notification gaps** – UI components such as `services/www_ameide_platform/features/workflows/components/ExecutionRuns.tsx` only display status/duration, and the worker callback in `services/workflows_runtime/src/workflows_runtime/callbacks.py` never emits preview URLs or subscriber notifications. Without these hooks, requesters cannot learn when previews are ready or which evidence files to review.
