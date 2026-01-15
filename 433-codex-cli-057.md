# 433 – Codex CLI 0.57.0 Pin

Update (2026-01-15): Coder workspaces unpinned by default

- The Coder workspace templates moved to “best-effort latest” installs for Codex CLI + the VS Code extension (no version knobs) to avoid workspace startup failures on transient network issues.
- This document is no longer the default policy for Coder workspaces; treat it as historical context / a regression-playbook if we intentionally reintroduce pinning for a specific incident.

## Context

- The default CLI delivered with the VS Code extension was `codex-cli 0.61.1-alpha.1`, but the workspace needs to remain on **0.57.0** until we explicitly qualify newer releases.
- The process to downgrade locally was:
  1. Uninstall any existing npm shim (`npm uninstall -g @openai/codex`).
  2. Fetch the 0.57.0 aarch64 Linux artifact from [rust-v0.57.0](https://github.com/openai/codex/releases/tag/rust-v0.57.0).
  3. Expand it into `~/.local/share/codex-cli/0.57.0/codex` and point `~/.local/bin/codex` at that binary.
- `~/.codex/config.toml` now explicitly sets `model = "gpt-5-codex"` with `model_reasoning_effort = "high"` so every CLI invocation consistently targets the desired back-end model.
- Note: This pin governs the codex binary used by shell sessions and automation in the DevContainer. The VS Code Codex extension ships its own CLI; set **Settings → Extensions → Codex → Codex: CLI Path** to `~/.local/bin/codex` (or the explicit version path) if the IDE agent itself must run 0.57.0.
- This pin exists to keep **all code-execution environments** deterministic: interactive devcontainers, CI, and any platform-run execution images that invoke Codex CLI. Where a coding agent currently uses a `develop_in_container` compatibility tool, it SHOULD still inherit this pin; the canonical direction is WorkRequest-driven runner execution with a pinned toolchain (see `backlog/527-transformation-capability.md` and `backlog/505-agent-developer-v2.md`).

## Devcontainer automation

To prevent new containers from drifting back to the extension-provided CLI:

- Added `.devcontainer/lib/codex-cli.sh` with `ensure_codex_cli_version()`. It:
  - Detects the current architecture (`x86_64` or `aarch64`).
  - Downloads `<release>/codex-<triple>.tar.gz` from the GitHub release.
  - Installs each version under `~/.local/share/codex-cli/<version>/codex`.
  - Maintains the `~/.local/bin/codex` symlink so PATH resolution matches the manual workflow.
- Keeps `$HOME/.local/bin` on `PATH` via `.bashrc` so new shells (bash/zsh) automatically honor the versioned symlink.
- `ensure_codex_config()` writes `~/.codex/config.toml` with `model = "gpt-5-codex"` and `model_reasoning_effort = "high"` (overridable via `DEVCONTAINER_CODEX_MODEL` and `DEVCONTAINER_CODEX_REASONING_EFFORT`). Both the CLI and the VS Code extension read this file, so their model settings stay in lockstep.
- Updated `.devcontainer/postCreate.sh` to source that helper and run it with `DEVCONTAINER_CODEX_VERSION` (defaults to `0.57.0`). Reopening the container now guarantees `codex --version` reports 0.57.0 before any bootstrap work begins.

## Auth policy (devcontainers; v1)

Canonical devcontainer auth uses a shared Codex home:

- Authenticate once and share/mount `~/.codex/auth.json` across parallel devcontainers to avoid callback collisions on `localhost:1455` (see `backlog/581-parallel-ai-devcontainers.md`).
- Defer `OPENAI_API_KEY` / API-key login to a future automation/headless track; do not treat API keys as the default devcontainer auth mechanism.

## Operator checklist

- **Verify local CLI** – `codex --version` should always output `codex-cli 0.57.0`.
- **Override (if needed)** – set `DEVCONTAINER_CODEX_VERSION` in the container env/context to test a newer release without touching the default.
- **Config audit** – `ensure_codex_config()` writes `~/.codex/config.toml` automatically, but periodically confirm it still lists `model = "gpt-5-codex"` / `model_reasoning_effort = "high"` (or update the `DEVCONTAINER_CODEX_MODEL/DEVCONTAINER_CODEX_REASONING_EFFORT` env vars if we intentionally test alternates).
- **VS Code extension** – if the IDE agent must use the pinned binary, set the Codex extension’s **CLI Path** to `~/.local/bin/codex`; otherwise it will keep running the bundled CLI version.

## Follow-ups

- Track upstream CLI regressions and file an upgrade plan before unpinning.
- Add CI smoke coverage that ensures `codex-cli --version` is `0.57.0` when devcontainer bootstrap scripts are exercised.
