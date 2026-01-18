---
title: "656 — Codex CLI Repo Navigation Archaeology (v6: projection inputs)"
status: draft
owners:
  - platform
created: 2026-01-18
related:
  - 656-agentic-memory-v6.md
  - 656-agentic-memory-implementation-v6.md
  - 509-proto-naming-conventions-v6.md
  - 496-eda-principles-v6.md
---

# Codex CLI repo-navigation archaeology report (source-backed)

A. Executive summary (1 page)

Repo + version
- Repository inspected: `codex` (cloned under `/tmp`)
- Commit: `0a568a47fdd9dc9290e50d8050f4685e2511b74b`

Default repo-navigation loop (one paragraph)
Codex CLI’s default “repo navigation” is primarily an agentic execution loop that delegates localization to model-chosen shell commands, then records and truncates outputs into a rolling conversation history with automatic compaction when token usage approaches a model-derived threshold. Concretely, each user turn builds an initial context bundle (permissions developer-instructions + project instructions from `AGENTS.md` + environment context), sends it to the model, executes any returned tool calls (most importantly `shell_command`), appends tool outputs back into history (with truncation policies), and repeats until the model emits a final assistant message; when history grows, Codex can “compact” it by summarizing user messages into a smaller replacement history.

Maturity rating table (Codex vs GitLab vs Research)
Scale: 1 (low) … 5 (high). “Codex (default)” means what is enabled in the common shipped model presets + observed tool surface.

| Capability | Codex (default) | GitLab approach (per your framing) | Research landscape (typical) |
|---|---:|---:|---:|
| Retrieval/indexing maturity | 1–2 | 4–5 | 3–5 |
| Structure awareness (AST/symbol/graph) | 1–2 | 3–5 | 3–5 |
| Planning/exploration policy | 2 | 2–4 | 3–5 |
| Context packaging & token efficiency | 4 | 3–4 | 3–4 |
| Verification/execution loop integration | 4 | 2–4 | 3–5 |
| Governance/safety controls | 4–5 | 4–5 | 2–4 |

Key differentiator (executive)
- Codex CLI is a mature tool-execution + context-management engine; its default “navigation substrate” is the filesystem via `shell_command` rather than a built-in semantic index, symbol graph, or retrieval pipeline.
- Codex core does contain structured repo tools (`grep_files`, `read_file`, `list_dir`) and rich context management (truncation + compaction), but those repo tools are gated behind `experimental_supported_tools` (often empty by default in shipped models).

B. “Mechanism map” aligned to the research taxonomy

1) Repository representation

What it does
- Maintains a conversational “thread history” of model inputs, tool calls, and tool outputs; does not build a persistent repository read-model (no symbol index DB, no embedding store, no call graph) in core.
- Injects environment + instruction scaffolding into each session/turn.

Where it lives (paths + symbols)
- Turn/session context assembly: `codex-rs/core/src/codex.rs` (`Session::build_initial_context`, `Codex::spawn`)
- Environment context: `codex-rs/core/src/environment_context.rs` (`EnvironmentContext`)
- Project doc discovery (`AGENTS.md`): `codex-rs/core/src/project_doc.rs` (`get_user_instructions`, `discover_project_doc_paths`, `read_project_docs`)
- History store: `codex-rs/core/src/context_manager/history.rs` (`ContextManager`)

How it’s used (call flow)
- On session spawn, Codex reads project docs/skills into `user_instructions`, stores in `SessionConfiguration` (`codex-rs/core/src/codex.rs` at `Codex::spawn`).
- For each turn, `Session::build_initial_context` injects:
  - developer permissions instructions
  - optional developer/user instructions
  - environment context message (cwd, shell)

Evidence (excerpt: initial context components)
`codex-rs/core/src/codex.rs` (`Session::build_initial_context`):
```rust
pub(crate) async fn build_initial_context(&self, turn_context: &TurnContext) -> Vec<ResponseItem> {
    let mut items = Vec::<ResponseItem>::with_capacity(4);
    let shell = self.user_shell();
    items.push(
        DeveloperInstructions::from_policy(
            &turn_context.sandbox_policy,
            turn_context.approval_policy,
            &turn_context.cwd,
        ).into(),
    );
    if let Some(user_instructions) = turn_context.user_instructions.as_deref() {
        items.push(UserInstructions { text: user_instructions.to_string(),
            directory: turn_context.cwd.to_string_lossy().into_owned(), }.into());
    }
    items.push(ResponseItem::from(EnvironmentContext::new(
        Some(turn_context.cwd.clone()), shell.as_ref().clone(),
    )));
    items
}
```

Evidence (excerpt: project doc scan model)
`codex-rs/core/src/project_doc.rs` (module header + `discover_project_doc_paths`):
```rust
//! Project-level documentation is primarily stored in files named `AGENTS.md`.
//! ... determine the Git repository root ... do **not** walk past the Git root.

pub fn discover_project_doc_paths(config: &Config) -> std::io::Result<Vec<PathBuf>> {
    // Build chain from cwd upwards and detect git root.
    // ...
    if git_exists { git_root = Some(cursor.clone()); break; }
    // ...
    // Collect every `AGENTS.md` found from the repository root down to cwd (inclusive).
}
```

Limits/edge cases
- Environment context is minimal (cwd + shell only); no structured repo snapshot is injected automatically.
  - `codex-rs/core/src/environment_context.rs` (`EnvironmentContext` holds only `cwd`, `shell`).
- “Repo representation” is implicit: the model sees only what it (or the agent) fetches via tools.

How it compares to research + GitLab
- Research systems often precompute a repo representation (files→chunks→embeddings/BM25; sometimes AST/symbol graphs).
- GitLab (per your framing) externalizes representation as a continuously maintained index; Codex core does not maintain such an index.

2) Retrieval & localization

What it does
- Default localization relies on model-driven tool use, dominated by `shell_command` (e.g., `rg`, `git grep`, `ls`, `sed`, `nl`, etc.).
- Structured navigation tools exist in core (`grep_files`, `read_file`, `list_dir`) but are gated behind `experimental_supported_tools`.

Where it lives (paths + symbols)
- Tool surface assembly/gating: `codex-rs/core/src/tools/spec.rs` (`ToolsConfig`, `build_specs`)
- Structured tools:
  - `codex-rs/core/src/tools/handlers/grep_files.rs` (`GrepFilesHandler`)
  - `codex-rs/core/src/tools/handlers/read_file.rs` (`ReadFileHandler`)
  - `codex-rs/core/src/tools/handlers/list_dir.rs` (`ListDirHandler`)
- Shell tool:
  - `codex-rs/core/src/tools/handlers/shell.rs` (`ShellHandler`, `ShellCommandHandler`)

How it’s used (call flow)
- Tools are registered in `build_specs` based on `ToolsConfig.experimental_supported_tools`.
- The model requests a tool call; Codex executes and returns outputs as `FunctionCallOutput` items.

Evidence (excerpt: gating of repo tools)
`codex-rs/core/src/tools/spec.rs` (`build_specs`):
```rust
if config.experimental_supported_tools.contains(&"grep_files".to_string()) {
    builder.push_spec_with_parallel_support(create_grep_files_tool(), true);
    builder.register_handler("grep_files", Arc::new(GrepFilesHandler));
}
if config.experimental_supported_tools.contains(&"read_file".to_string()) {
    builder.push_spec_with_parallel_support(create_read_file_tool(), true);
    builder.register_handler("read_file", Arc::new(ReadFileHandler));
}
if config.experimental_supported_tools.iter().any(|tool| tool == "list_dir") {
    builder.push_spec_with_parallel_support(create_list_dir_tool(), true);
    builder.register_handler("list_dir", Arc::new(ListDirHandler));
}
```

Evidence (excerpt: `grep_files` is ripgrep wrapper; returns filenames only)
`codex-rs/core/src/tools/handlers/grep_files.rs` (`run_rg_search`):
```rust
let mut command = Command::new("rg");
command.current_dir(cwd)
    .arg("--files-with-matches")
    .arg("--sortr=modified")
    .arg("--regexp").arg(pattern)
    .arg("--no-messages");
if let Some(glob) = include { command.arg("--glob").arg(glob); }
command.arg("--").arg(search_path);
```

Evidence (excerpt: `read_file` supports slice + indentation “semantic zoom”)
`codex-rs/core/src/tools/handlers/read_file.rs` (`ReadMode`, absolute path requirement):
```rust
#[serde(rename_all = "snake_case")]
enum ReadMode { #[default] Slice, Indentation }

let path = PathBuf::from(&file_path);
if !path.is_absolute() {
    return Err(FunctionCallError::RespondToModel("file_path must be an absolute path".to_string()));
}
let collected = match mode {
    ReadMode::Slice => slice::read(&path, offset, limit).await?,
    ReadMode::Indentation => indentation::read_block(&path, offset, limit, indentation).await?,
};
```

Evidence (structured tools appear “available but not default”)
- Integration tests explicitly mark `read_file` and `list_dir` as disabled until enabled:
  - `codex-rs/core/tests/suite/read_file.rs` (`#[ignore = "disabled until we enable read_file tool"]`)
  - `codex-rs/core/tests/suite/list_dir.rs` (`#[ignore = "disabled until we enable list_dir tool"]`)
- A test-only model slug enables these tools via model metadata:
  - `codex-rs/core/src/models_manager/model_info.rs` (`find_model_info_for_slug`, `slug.starts_with("test-gpt-5")` sets `experimental_supported_tools` including `grep_files`, `list_dir`, `read_file`).

Excerpt (test model enables tools)
`codex-rs/core/src/models_manager/model_info.rs` (`find_model_info_for_slug`):
```rust
} else if slug.starts_with("test-gpt-5") {
    model_info!(
        slug,
        experimental_supported_tools: vec![
            "grep_files".to_string(),
            "list_dir".to_string(),
            "read_file".to_string(),
            "test_sync_tool".to_string(),
        ],
        shell_type: ConfigShellToolType::ShellCommand,
    )
}
```

Default posture evidence (shipped models)
- In shipped `models.json`, models have `experimental_supported_tools: []` (observed via `jq`), and the default tool list observed in experiments is 7 tools: `shell_command`, MCP resource tools, `update_plan`, `apply_patch`, `view_image`.
- Tool gating means “repo navigation” is mostly `shell_command` unless a model explicitly advertises repo tools.

“Not found” (semantic retrieval/indexing in core)
Commands run against `codex-rs/core/src`:
- `rg -n "faiss|hnsw|bm25|tf-idf|embedding" codex-rs/core/src -S` → no matches
- `rg -n "\\bLSP\\b|language server" codex-rs/core/src -S` → no matches
- `rg -n --fixed-strings "rust-analyzer" codex-rs/core/src -S` → no matches
- `rg -n --fixed-strings "ctags" codex-rs/core/src -S` → no matches
- `rg -n --fixed-strings "callgraph" codex-rs/core/src -S` → no matches
- `rg -n --fixed-strings "xref" codex-rs/core/src -S` → no matches

Tree-sitter is present but for shell safety parsing, not code AST indexing:
- `codex-rs/core/src/bash.rs` (`try_parse_shell`, uses `tree_sitter_bash`).

3) Exploration & planning policy

What it does
- Implements an explicit controller loop for “model ↔ tools ↔ history” within a turn; tool selection and exploration sequencing are model-driven.
- Provides a plan tool (`update_plan`) but it is not an internal planner; it’s a tool exposed to the model/UI.

Where it lives (paths + symbols)
- Turn loop: `codex-rs/core/src/codex.rs` (`run_turn`)
- Tool dispatch: `codex-rs/core/src/tools/router.rs` (`ToolRouter`)
- Registry dispatch + gating: `codex-rs/core/src/tools/registry.rs` (`ToolRegistry::dispatch`)
- Plan tool spec is always included: `codex-rs/core/src/tools/spec.rs` (`builder.push_spec(PLAN_TOOL.clone())`)

Evidence (excerpt: run_turn describes tool-call loop)
`codex-rs/core/src/codex.rs` (`run_turn` doc comment):
```rust
/// Takes a user message as input and runs a loop where ... the model replies with either:
/// - requested function calls
/// - an assistant message
/// If the model requests a function call, we execute it and send the output back ...
/// If the model sends only an assistant message ... consider the turn complete.
pub(crate) async fn run_turn( ... ) -> Option<String> { ... }
```

Evidence (no explicit search policy / MCTS / RL in core)
Command run:
- `rg -n "mcts|monte carlo|beam search|value function|reinforcement learning" codex-rs/core/src -S` → no matches

Limits/edge cases
- No built-in multi-candidate retrieval pipeline (e.g., file shortlist → rerank → slice) is enforced by the engine; you get what the model chooses to do with available tools.

How it compares to research + GitLab
- SWE-agent-style systems often implement an “ACI” tool protocol with a more opinionated browse/search/open workflow; Agentless/graph-based systems encode localization pipelines and rerankers; Codex CLI’s default is closer to “general tool-use loop + guardrails.”

4) Context packaging

What it does
- Maintains a history of `ResponseItem`s, truncating tool outputs under a configurable policy.
- Estimates token usage heuristically, triggers compaction to keep within context.
- Packages developer instructions (permission/sandbox/approval policy), user instructions (AGENTS/skills), and environment context.

Where it lives (paths + symbols)
- History + truncation of tool outputs: `codex-rs/core/src/context_manager/history.rs` (`ContextManager::record_items`, `process_item`)
- Truncation policy helpers: `codex-rs/core/src/truncate.rs` (`TruncationPolicy`, `truncate_text`, `truncate_function_output_items_with_policy`)
- Auto-compaction threshold: `codex-rs/protocol/src/openai_models.rs` (`ModelInfo::auto_compact_token_limit`)
- Compaction implementation: `codex-rs/core/src/compact.rs` (`run_inline_auto_compact_task`, `build_compacted_history_with_limit`)

Evidence (excerpt: tool output truncation on insert)
`codex-rs/core/src/context_manager/history.rs` (`process_item`):
```rust
match item {
  ResponseItem::FunctionCallOutput { call_id, output } => {
    let truncated = truncate_text(output.content.as_str(), policy_with_serialization_budget);
    let truncated_items = output.content_items.as_ref().map(|items| {
      truncate_function_output_items_with_policy(items, policy_with_serialization_budget)
    });
    ResponseItem::FunctionCallOutput { call_id: call_id.clone(),
      output: FunctionCallOutputPayload { content: truncated, content_items: truncated_items,
        success: output.success, }, }
  }
  // ...
}
```

Evidence (excerpt: default auto-compaction threshold = 90% context window if unset)
`codex-rs/protocol/src/openai_models.rs` (`ModelInfo::auto_compact_token_limit`):
```rust
pub fn auto_compact_token_limit(&self) -> Option<i64> {
    self.auto_compact_token_limit.or_else(|| {
        self.context_window.map(|context_window| (context_window * 9) / 10)
    })
}
```

Evidence (excerpt: trigger point in run_turn)
`codex-rs/core/src/codex.rs` (`run_turn`):
```rust
let model_info = turn_context.client.get_model_info();
let auto_compact_limit = model_info.auto_compact_token_limit().unwrap_or(i64::MAX);
let total_usage_tokens = sess.get_total_token_usage().await;
if total_usage_tokens >= auto_compact_limit {
    run_auto_compact(&sess, &turn_context).await;
}
```

Evidence (excerpt: compaction replaces history with selected recent user messages + summary)
`codex-rs/core/src/compact.rs` (`build_compacted_history_with_limit`):
```rust
for message in user_messages.iter().rev() {
    let tokens = approx_token_count(message);
    if tokens <= remaining { selected_messages.push(message.clone()); remaining -= tokens; }
    else { selected_messages.push(truncate_text(message, TruncationPolicy::Tokens(remaining))); break; }
}
for message in &selected_messages {
    history.push(ResponseItem::Message { role: "user".to_string(),
        content: vec![ContentItem::InputText { text: message.clone() }], id: None });
}
history.push(ResponseItem::Message { role: "user".to_string(),
    content: vec![ContentItem::InputText { text: summary_text.to_string() }], id: None });
```

Limits/edge cases
- Token counts are heuristic (byte-based) in `ContextManager::estimate_token_count` (`codex-rs/core/src/context_manager/history.rs`).
- Compaction summarizes user messages and appends a summary as a user message; repeated compactions can reduce fidelity (Codex emits a warning in `compact.rs` after compaction).

How it compares to research + GitLab
- Codex’s context management (truncation + compaction + instruction injection) is comparatively mature and explicit; many research systems handwave this or implement simpler “top-K chunks” packing.

5) Verification & execution loop

What it does
- Executes model-requested commands via shell tools; records stdout/stderr (bounded) and returns to model.
- Approval/sandbox policies gate potentially mutating actions; safe command detection influences gating.

Where it lives (paths + symbols)
- Shell execution tool handlers: `codex-rs/core/src/tools/handlers/shell.rs` (`ShellHandler`, `ShellCommandHandler`)
- Tool registry gating for mutating actions: `codex-rs/core/src/tools/registry.rs` (`ToolHandler::is_mutating`, `ToolRegistry::dispatch`)
- Approval request plumbing: `codex-rs/core/src/codex.rs` (`Session::request_command_approval`)
- Output bounding for unified exec: `codex-rs/core/src/unified_exec/head_tail_buffer.rs` (`HeadTailBuffer`)

Evidence (excerpt: approval policy guard against “escalated permissions” in non-OnRequest)
`codex-rs/core/src/tools/handlers/shell.rs` (`ShellHandler::run_exec_like`):
```rust
if exec_params.sandbox_permissions.requires_escalated_permissions()
    && !matches!(turn.approval_policy, AskForApproval::OnRequest)
{
    return Err(FunctionCallError::RespondToModel(format!(
        "approval policy is {policy:?}; reject command — ...", policy = turn.approval_policy
    )));
}
```

Evidence (excerpt: registry waits on tool gate for mutating tools)
`codex-rs/core/src/tools/registry.rs` (`ToolRegistry::dispatch`):
```rust
if handler.is_mutating(&invocation).await {
    invocation.turn.tool_call_gate.wait_ready().await;
}
match handler.handle(invocation).await { /* ... */ }
```

Evidence (excerpt: bounded output buffer keeps head+tail)
`codex-rs/core/src/unified_exec/head_tail_buffer.rs` (`HeadTailBuffer::new`):
```rust
/// preserves a stable prefix ("head") and suffix ("tail"), dropping the middle once it exceeds ...
pub(crate) fn new(max_bytes: usize) -> Self {
    let head_budget = max_bytes / 2;
    let tail_budget = max_bytes.saturating_sub(head_budget);
    Self { max_bytes, head_budget, tail_budget, /* ... */ }
}
```

Limits/edge cases
- No automatic “test selection” logic is imposed by core; tests are just commands the model decides to run.
- Verification strategy is therefore model/prompt-dependent.

How it compares to research + GitLab
- Codex CLI has strong execution-loop integration (tool orchestration, gating, output bounding). Many research systems emphasize retrieval/localization more than execution hardening.

6) Extensibility: MCP and tool interfaces

What it does
- Supports MCP servers configured in `config.toml`; merges MCP tools into the tool surface offered to the model.
- Exposes built-in MCP “resource” tools (`list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource`) and an MCP tool-call path (via `ToolPayload::Mcp`).

Where it lives (paths + symbols)
- MCP server config types: `codex-rs/core/src/config/types.rs` (`McpServerConfig`, `RawMcpServerConfig`)
- Config field: `codex-rs/core/src/config/mod.rs` (`Config::mcp_servers`)
- MCP tool discovery and calls: `codex-rs/core/src/mcp_connection_manager.rs` (`list_all_tools`, `call_tool`, `parse_tool_name`)
- Tool routing for MCP tool names: `codex-rs/core/src/tools/router.rs` (`ToolRouter::build_tool_call`)
- Tool list assembly includes MCP resource tools by default: `codex-rs/core/src/tools/spec.rs` (`build_specs`)
- Turn builds router with MCP tools: `codex-rs/core/src/codex.rs` (`run_sampling_request`)

Evidence (excerpt: MCP server config includes allow/deny tool filters)
`codex-rs/core/src/config/types.rs` (`McpServerConfig`):
```rust
pub struct McpServerConfig {
    #[serde(flatten)]
    pub transport: McpServerTransportConfig,
    #[serde(default = "default_enabled")]
    pub enabled: bool,
    pub startup_timeout_sec: Option<Duration>,
    pub tool_timeout_sec: Option<Duration>,
    pub enabled_tools: Option<Vec<String>>,
    pub disabled_tools: Option<Vec<String>>,
}
```

Evidence (excerpt: router recognizes MCP tool names and builds ToolPayload::Mcp)
`codex-rs/core/src/tools/router.rs` (`ToolRouter::build_tool_call`):
```rust
if let Some((server, tool)) = session.parse_mcp_tool_name(&name).await {
    Ok(Some(ToolCall { tool_name: name, call_id,
        payload: ToolPayload::Mcp { server, tool, raw_arguments: arguments }, }))
} else {
    Ok(Some(ToolCall { tool_name: name, call_id, payload: ToolPayload::Function { arguments }, }))
}
```

Evidence (excerpt: run_sampling_request merges MCP tools into ToolRouter)
`codex-rs/core/src/codex.rs` (`run_sampling_request`):
```rust
let mcp_tools = sess.services.mcp_connection_manager.read().await.list_all_tools().await?;
let router = Arc::new(ToolRouter::from_config(
    &turn_context.tools_config,
    Some(mcp_tools.into_iter().map(|(name, tool)| (name, tool.tool)).collect()),
));
let prompt = Prompt { input, tools: router.specs(), /* ... */ };
```

Evidence (excerpt: tool list always includes MCP resource tools)
`codex-rs/core/src/tools/spec.rs` (`build_specs`):
```rust
builder.push_spec_with_parallel_support(create_list_mcp_resources_tool(), true);
builder.push_spec_with_parallel_support(create_list_mcp_resource_templates_tool(), true);
builder.push_spec_with_parallel_support(create_read_mcp_resource_tool(), true);
builder.register_handler("list_mcp_resources", mcp_resource_handler.clone());
builder.register_handler("list_mcp_resource_templates", mcp_resource_handler.clone());
builder.register_handler("read_mcp_resource", mcp_resource_handler);
```

Security/permission model
- Permissions guidance is injected as developer instructions and varies by sandbox + approval policy:
  - `codex-rs/protocol/src/models.rs` (`DeveloperInstructions::from_policy`, `sandbox_text`, `impl From<AskForApproval>`)
  - Templates: `codex-rs/protocol/src/prompts/permissions/approval_policy/*`, `codex-rs/protocol/src/prompts/permissions/sandbox_mode/*`

Excerpt (developer instructions derived from policy)
`codex-rs/protocol/src/models.rs` (`DeveloperInstructions::from_policy`):
```rust
pub fn from_policy(sandbox_policy: &SandboxPolicy, approval_policy: AskForApproval, cwd: &Path) -> Self {
    let network_access = if sandbox_policy.has_full_network_access() {
        NetworkAccess::Enabled
    } else { NetworkAccess::Restricted };
    let (sandbox_mode, writable_roots) = match sandbox_policy { /* ... */ };
    DeveloperInstructions::from_permissions_with_network(
        sandbox_mode, network_access, approval_policy, writable_roots,
    )
}
```

C. Architecture diagrams

Sequence diagram (typical “fix bug” session)
1. User → `codex exec` CLI (`codex-rs/exec/src/cli.rs`) supplies prompt + options (`--model`, `--sandbox`, `--json`).
2. CLI → core session spawn (`codex-rs/core/src/codex.rs` `Codex::spawn`)
   - loads skills and project docs (`get_user_instructions`)
   - initializes MCP servers (if configured)
3. Turn begins (`codex-rs/core/src/codex.rs` `run_turn`)
   - checks auto-compaction threshold; maybe compacts
   - records user prompt into history
4. Core → model request (`run_sampling_request`)
   - builds `ToolRouter` with built-ins + MCP tools
   - sends `Prompt { input history, tools }` to model client
5. Model → tool call(s)
6. Core executes tool calls (`ToolRouter::dispatch_tool_call` → `ToolRegistry::dispatch` → handler)
   - shell commands via `ShellCommandHandler` / `ShellHandler`
   - patch via `ApplyPatchHandler`
   - MCP via MCP handlers / connection manager
7. Core appends tool outputs to `ContextManager` (truncating per policy)
8. Repeat 4–7 until model returns a final assistant message; turn completes.

Component diagram (core “repo navigation” relevant pieces)
- CLI layer: `codex-rs/exec` (parses args, emits JSONL/human output)
- Session/turn orchestration: `codex-rs/core/src/codex.rs` (`Codex`, `Session`, `TurnContext`, `run_turn`)
- Context management: `codex-rs/core/src/context_manager/*` + `codex-rs/core/src/truncate.rs` + `codex-rs/core/src/compact.rs`
- Tool system:
  - Spec registry builder: `codex-rs/core/src/tools/spec.rs`
  - Router: `codex-rs/core/src/tools/router.rs`
  - Registry + gating: `codex-rs/core/src/tools/registry.rs`
  - Handlers: `codex-rs/core/src/tools/handlers/*` (shell, apply_patch, grep_files, read_file, list_dir, MCP)
- MCP subsystem: `codex-rs/core/src/mcp_connection_manager.rs` + config types in `codex-rs/core/src/config/types.rs`
- Safety/permissions substrate:
  - developer instruction templates: `codex-rs/protocol/src/prompts/permissions/*`
  - shell safety + approvals: `codex-rs/core/src/tools/handlers/shell.rs`, `codex-rs/core/src/codex.rs` (`request_command_approval`)

D. Empirical validation (small)

Setup notes (controlled)
- Repo under test: Codex repo itself (same commit above).
- Interface: `codex exec --json` (JSONL event stream) as defined in `codex-rs/exec/src/cli.rs` (`Cli::json` flag).
- Model: `gpt-5.2-codex` (default tool surface observed: 7 tools; see below).

Experiment 1: “Locate a symbol” task
Prompt: “Locate the ToolRouter struct”

Observed tool calls (transcript excerpt)
```json
{"type":"item.completed","item":{"type":"command_execution","command":"/bin/bash -lc 'rg -n \"pub struct ToolRouter\" codex-rs/core/src -S'","exit_code":0}}
{"type":"item.completed","item":{"type":"command_execution","command":"/bin/bash -lc 'nl -ba codex-rs/core/src/tools/router.rs | head -n 140'","exit_code":0}}
```

Localization cost metrics
- Tool calls until first correct file identified: 1 (the `rg` call returns `codex-rs/core/src/tools/router.rs:28`)
- Unique files explicitly opened/printed: 1 (`codex-rs/core/src/tools/router.rs` via `nl | head`)
- Total command executions: 2

Context efficiency metrics (request body sizes)
From request logs (each model request is a `/responses` body with `body_bytes`):
- Request bytes: 25725 → 26094 → 32169
- Interpretation: file content inclusion materially grows prompt size (third request is larger after adding outputs).

Relating behavior to code
- This matches the `run_turn` loop semantics (`codex-rs/core/src/codex.rs` `run_turn`) and the default reliance on `shell_command` (tools registered in `codex-rs/core/src/tools/spec.rs`).

Experiment 2: “Bugfix-like” task
Prompt: “Fix the failing script at tmp/bug.sh”

Observed iterations (transcript excerpt)
```json
{"type":"item.completed","item":{"type":"command_execution","command":"/bin/bash -lc 'bash ./tmp/bug.sh'","aggregated_output":"fail\n","exit_code":1,"status":"failed"}}
{"type":"item.completed","item":{"type":"file_change","changes":[{"path":".../tmp/bug.sh","kind":"update"}]}}
{"type":"item.completed","item":{"type":"command_execution","command":"/bin/bash -lc 'bash ./tmp/bug.sh'","aggregated_output":"ok\n","exit_code":0}}
```

Localization/verification metrics
- Command executions: 2 (run script, rerun script)
- Patch iterations: 1 file update
- Outcome: script returns exit 0 (current file content: prints “ok”, exits 0)

Context efficiency metrics
- Request bytes: 25731 → 26005 → 26361 → 26633 (steady growth, smaller than Exp1’s jump).

Relating behavior to code
- Execution + patching reflects the tool loop (`run_turn`) and tool gating/dispatch (`ToolRouter`/`ToolRegistry`) plus bounded-history insert (`ContextManager::process_item`).

Observed default tool surface (runtime)
Across both experiments, the tool list size in requests is 7, and the names are:
- `shell_command`, `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource`, `update_plan`, `apply_patch`, `view_image`
This aligns with `build_specs` always registering shell + MCP resource tools + plan + apply_patch (when enabled) + view_image (`codex-rs/core/src/tools/spec.rs`).

E. Comparison section (Codex vs GitLab vs Research)

Mapping Codex mechanisms to research taxonomy
- SWE-agent (ACI-style): Typically exposes structured browse/search/open/edit tools as first-class primitives; Codex has analogous primitives (`grep_files`, `read_file`, `list_dir`) but they are gated and not enabled in common shipped models, making default behavior “shell-driven ACI.”
- Agentless-style localization pipelines: Often implement explicit localization stages (file candidate generation via retrieval, ranking, then targeted reads); Codex core does not implement a pipeline—localization is implicit in model tool choice.
- AutoCodeRover / graph-based retrieval: Often relies on repository graphs (dependency/call graphs) and/or chunk graphs; Codex core has MCP to plug this in, but no built-in graph index.
- MCTS/RL retrieval sub-agents: Not found in Codex core (see “not found” search for MCTS/beam/value-function terms).

Comparison to “GitLab semantic code search indexing + chunking + MCP tools” (as a substrate)
- Codex CLI substrate (evidence-backed): filesystem + shell + optional structured tools + optional MCP integration; no continuously maintained repo read-model in core.
- GitLab substrate (per your framing): platform-managed semantic index with chunking; exposed via code search APIs/tools; plus MCP-style tool exposure.
- Net: GitLab/research treat retrieval/indexing as a first-class system component; Codex treats retrieval as a tool-use behavior (and delegates most navigation to the model + shell unless you enable richer tools or add MCP servers).

Biggest missing primitives in Codex core (default)
- Semantic index (vector/BM25) and chunk store (not found in `codex-rs/core/src`; see searches).
- AST/symbol extraction for code navigation (tree-sitter is present only for shell parsing safety in `codex-rs/core/src/bash.rs`).
- Graph traversal primitives (call graph, import graph, dependency graph): not found in core.

Codex strengths relative to research/GitLab
- Context management: explicit truncation policy on tool outputs + compaction with defined thresholds and mechanics.
- Execution governance: sandbox + approval policies influence both tool execution and injected developer instructions (policy templates + runtime guards).

F. Recommendations (actionable)

Top 5 improvements (ranked by impact vs effort)

1) Enable structured repo tools by default for Codex-optimized models
- Impact: High (replaces ad-hoc `rg` + `cat` patterns with bounded, schema’d tools; reduces prompt bloat and improves determinism).
- Effort: Low–Medium.
- Where: `codex-rs/core/models.json` (populate `experimental_supported_tools` for the default codex models) and/or model metadata pipeline; tool registration already exists in `codex-rs/core/src/tools/spec.rs`.
- How to test: un-ignore and stabilize `codex-rs/core/tests/suite/read_file.rs` and `codex-rs/core/tests/suite/list_dir.rs` by enabling tools in default model config.

2) Upgrade `grep_files` to return “jump points” (match lines with line numbers), not only filenames
- Impact: High (materially improves localization speed vs shell-driven `rg` loops).
- Effort: Medium.
- Where: `codex-rs/core/src/tools/handlers/grep_files.rs` (switch from `--files-with-matches` to an output format that includes line numbers; cap results).
- How to test: extend `codex-rs/core/tests/suite/grep_files.rs` to assert on line-numbered matches.

3) Add a first-party MCP “semantic repo search” server (MCP add-on, not core)
- Impact: High (closes the “no semantic index” gap without bloating core).
- Effort: Medium–High (new MCP server + index build).
- Interface: MCP tools like `search_symbols`, `search_semantic`, `find_references`, returning file+line ranges compatible with `read_file` slices.
- How to evaluate: reuse the localization-cost metrics above (tool calls to first correct file; bytes added to context).

4) Make tool-output shaping more “navigation-aware” (context efficiency)
- Impact: Medium.
- Effort: Medium.
- Where: `codex-rs/core/src/context_manager/history.rs` and `codex-rs/core/src/truncate.rs` to add structured summaries for large command outputs (e.g., keep only headers + errors, or detect `rg` output patterns).
- How to test: add targeted unit tests around `ContextManager::process_item` truncation behavior.

5) Add an opt-in “repo navigation mode” profile that turns on repo tools + safer defaults
- Impact: Medium (easy activation path; avoids model-metadata dependence).
- Effort: Medium (config surface + documentation).
- Where: config/profile plumbing in `codex-rs/core/src/config/mod.rs` and selection in `codex-rs/exec/src/cli.rs`.
- Note: a direct config override for `experimental_supported_tools` was not found in `codex-rs/core/config.schema.json`; today this appears model-driven via `ModelInfo.experimental_supported_tools` (`codex-rs/core/src/tools/spec.rs`).

Source evidence index
- `codex-rs/core/src/codex.rs`: `Codex::spawn`, `run_turn`, `run_auto_compact`, `run_sampling_request`, `TurnContext`, `Session::build_initial_context`, `Session::request_command_approval`, `Session::parse_mcp_tool_name`
- `codex-rs/core/src/tools/spec.rs`: `ToolsConfig`, `ToolsConfig::new`, `build_specs`, `create_grep_files_tool`, `create_read_file_tool`, `create_list_dir_tool`
- `codex-rs/core/src/tools/router.rs`: `ToolRouter::from_config`, `ToolRouter::build_tool_call`, `ToolRouter::dispatch_tool_call`
- `codex-rs/core/src/tools/registry.rs`: `ToolHandler`, `ToolRegistry::dispatch`, `ToolKind`
- `codex-rs/core/src/tools/handlers/grep_files.rs`: `GrepFilesHandler`, `run_rg_search`
- `codex-rs/core/src/tools/handlers/read_file.rs`: `ReadFileHandler`, `ReadMode`
- `codex-rs/core/src/tools/handlers/list_dir.rs`: `ListDirHandler`
- `codex-rs/core/src/tools/handlers/shell.rs`: `ShellHandler`, `ShellCommandHandler`, `ShellHandler::run_exec_like`
- `codex-rs/core/src/context_manager/history.rs`: `ContextManager`, `ContextManager::record_items`, `ContextManager::process_item`, `ContextManager::estimate_token_count`
- `codex-rs/core/src/truncate.rs`: `TruncationPolicy`, `truncate_text`, `truncate_function_output_items_with_policy`
- `codex-rs/core/src/compact.rs`: `run_inline_auto_compact_task`, `run_compact_task_inner`, `build_compacted_history_with_limit`
- `codex-rs/protocol/src/openai_models.rs`: `ModelInfo`, `ModelInfo::auto_compact_token_limit`
- `codex-rs/core/src/models_manager/model_info.rs`: `find_model_info_for_slug`, `with_config_overrides`
- `codex-rs/protocol/src/models.rs`: `DeveloperInstructions`, `DeveloperInstructions::from_policy`, `DeveloperInstructions::sandbox_text`, `impl From<AskForApproval> for DeveloperInstructions`
- `codex-rs/protocol/src/prompts/permissions/approval_policy/unless_trusted.md`
- `codex-rs/protocol/src/prompts/permissions/approval_policy/on_request.md`
- `codex-rs/protocol/src/prompts/permissions/sandbox_mode/danger_full_access.md`
- `codex-rs/protocol/src/prompts/permissions/sandbox_mode/workspace_write.md`
- `codex-rs/protocol/src/prompts/permissions/sandbox_mode/read_only.md`
- `codex-rs/core/src/project_doc.rs`: `get_user_instructions`, `read_project_docs`, `discover_project_doc_paths`
- `codex-rs/core/src/environment_context.rs`: `EnvironmentContext`
- `codex-rs/core/src/bash.rs`: `try_parse_shell` (tree-sitter-bash usage for shell parsing)
- `codex-rs/core/src/config/types.rs`: `McpServerConfig`, `RawMcpServerConfig`
- `codex-rs/core/src/config/mod.rs`: `Config::mcp_servers` (field), `Config::tool_output_token_limit` (field)
- `codex-rs/core/src/mcp_connection_manager.rs`: `list_all_tools`, `call_tool`, `parse_tool_name`, `ToolFilter`
- `codex-rs/core/tests/suite/read_file.rs`: `read_file_tool_returns_requested_lines` (ignored until tool enabled)
- `codex-rs/core/tests/suite/list_dir.rs`: `list_dir_tool_returns_entries` (ignored until tool enabled)
- `codex-rs/core/tests/suite/grep_files.rs`: `MODEL_WITH_TOOL`, `grep_files_tool_collects_matches`
- `codex-rs/exec/src/cli.rs`: `Cli` (`--json`, `--model`, `--sandbox`)
- `codex-rs/exec/src/main.rs`: arg0 dispatch + CLI entry
