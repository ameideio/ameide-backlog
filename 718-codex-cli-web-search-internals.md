---
title: "718 — Codex CLI Web Search (internals + enablement; pinned 0.87.0)"
status: draft
owners:
  - platform-devx
  - agents
  - platform
created: 2026-01-21
updated: 2026-01-21
related:
  - 433-codex-cli-057.md
  - 613-codex-auth-json-secret.md
  - 656-codex-cli-repo-navigation-archaeology-v6.md
  - 717-ameide-agents-v6.md
  - 650-agentic-coding-overview-v6.md
  - 654-agentic-coding-cli-surface-coder.md
---

# 718 — Codex CLI Web Search (internals + enablement)

This backlog documents how **Codex CLI** enables the OpenAI Responses native **`web_search`** tool, and how we should configure it in Ameide workspaces and agent runs.

Scope:
- enablement knobs (flag + config),
- the internal wiring points (config → tool registration → API headers),
- compatibility/deprecation notes for older config keys.

Out of scope:
- transformation memory (Git-backed memory is projection-owned; see `backlog/656-agentic-memory-v6.md`),
- whether the platform should enable web search by default (policy decision).

## 0) Version grounding (Ameide pin)

This doc is grounded in **`codex-cli 0.87.0`** (see `backlog/433-codex-cli-057.md`).

For upstream source references, use tag **`openai/codex` `rust-v0.87.0`**.

## 1) User-facing enablement

Codex CLI defaults to *not* exposing web search to the model unless enabled.

### 1.1 Per-session (CLI flag)

- `codex --search`
  - enables **live** web search for that run.

This is implemented as a config override equivalent to `-c web_search="live"`.

### 1.2 Permanent (config.toml)

In `~/.codex/config.toml`, set one of:

- `web_search = "live"`
- `web_search = "cached"`
- `web_search = "disabled"`

Notes:
- `disabled` is an explicit opt-out.
- `cached` is “web_search tool present, but external web access disabled”.

### 1.3 Per-profile (config.toml)

Codex supports profiles; `web_search` can be set per profile (profile wins over global).

## 2) Internal wiring (how web search becomes a tool)

Codex CLI’s wiring is **config-driven**:

1) **Resolve `web_search_mode`**
- In core config load, Codex resolves a `web_search_mode` from explicit config (preferred) and/or feature flags (legacy compatibility).

Key function:
- `codex-rs/core/src/config/mod.rs`: `resolve_web_search_mode(...)`

2) **Register the Responses tool**
- Codex registers the OpenAI tool spec only when `web_search_mode` is set to `cached` or `live`.

Key function:
- `codex-rs/core/src/tools/spec.rs`: tool spec builder (`match config.web_search_mode { ... }`)
  - `Live` → `ToolSpec::WebSearch { external_web_access: Some(true) }`
  - `Cached` → `ToolSpec::WebSearch { external_web_access: Some(false) }`
  - `Disabled|None` → no web search tool

3) **Emit a Responses request header**
- Codex sets the `x-oai-web-search-eligible` request header based on the resolved mode.

Key function:
- `codex-rs/core/src/client.rs`: `build_responses_headers(...)`
  - `web_search_mode == Disabled` → header `"false"`
  - otherwise (including `None`) → header `"true"`

Important: “eligible header true” does not, by itself, mean the tool is exposed; the tool is exposed only when the tool spec is registered (step 2).

## 3) Legacy / deprecated config paths (compatibility)

Codex still supports older knobs, but we should not standardize on them for new work.

### 3.1 Feature flag (legacy)

If `features.web_search_request=true`, Codex will resolve web search mode to `live` when no explicit `web_search` mode is set.

Enable via:
- `codex --enable web_search_request`
- or `codex -c features.web_search_request=true`

### 3.2 Tools table (legacy alias)

Codex’s `[tools]` table still contains a legacy boolean that maps to web search request:

- `[tools] web_search = true`
- `[tools] web_search_request = true` (alias)

This is kept for backwards compatibility; prefer `web_search = "live"` instead.

## 4) Ameide guidance (how we use this)

### 4.1 Architect roles (research + verification)

Per `backlog/717-ameide-agents-v6.md`, architect agents may run Codex CLI for research/verification.

If web search is needed for an architecture task:
- prefer `codex --search` (per-session),
- avoid permanently enabling `web_search="live"` for every run unless the environment threat model explicitly allows it.

### 4.2 Developer role (implementation)

Developers use Codex CLI as a development tool, but all repo verification remains via Ameide front doors:
- `./ameide test` (default)
- see `backlog/654-agentic-coding-cli-surface-coder.md`

## 5) Limitations (model/tool constraints)

- Web search is tool-mediated; the model decides when to call it and is subject to model/tool limits (e.g., per-call query limits).
- Cached mode may not satisfy tasks that require fetching fresh vendor documentation.

## 6) Accuracy review (feedback incorporated)

This section embeds a review pass against upstream Codex sources (tag `rust-v0.87.0`) and clarifies minor wording issues.

### ✅ Confirmed correct

| Section | Claim | Evidence (upstream) |
|---|---|---|
| 1.1 | `codex --search` enables web search | `codex-rs/cli/src/main.rs`: `--search` maps to `web_search="live"` |
| 1.2 | Config values: `"live"`, `"cached"`, `"disabled"` | `codex-rs/core/config.schema.json`: `WebSearchMode` enum |
| 1.3 | Profile overrides global | `codex-rs/core/src/config/mod.rs`: `config_profile.web_search.or(config_toml.web_search)` |
| 2.1 | `resolve_web_search_mode()` exists in config | `codex-rs/core/src/config/mod.rs`: `resolve_web_search_mode(...)` |
| 2.2 | Tool spec registration logic | `codex-rs/core/src/tools/spec.rs`: `match config.web_search_mode { Live/Cached/... }` |
| 2.3 | Header `x-oai-web-search-eligible` | `codex-rs/core/src/client.rs`: `build_responses_headers(...)` |
| 3.1 | `features.web_search_request=true` resolves to Live mode | `codex-rs/core/src/config/mod.rs`: `Feature::WebSearchRequest => WebSearchMode::Live` |
| 3.2 | `[tools] web_search` / `web_search_request` aliases | `codex-rs/core/src/config/mod.rs`: `ToolsToml` alias mapping |

### ⚠️ Clarifications

| Section | Issue | Correction |
|---|---|---|
| 2.2 | “Tool spec builder” naming | The code uses a “builder + `push_spec(...)` pattern” in `codex-rs/core/src/tools/spec.rs` (no special named builder function for web search). |
| 2.3 | Header meaning | The header is `"true"` even when `web_search_mode` is `None`; this header indicates eligibility, not that the `web_search` tool was registered. |
