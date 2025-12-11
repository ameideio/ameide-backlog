# 041 – core-platform-coder package review & improvement plan

Date: 2025-07-25

Author: internal-agents-review

> **Superseded – 2025-02:** The remediation actions and runtime direction in this notebook have been replaced by the platform-wide LangGraph coder plan: [505-agent-developer.md](../505-agent-developer.md) (agent-developer), [504-agent-vertical-slice.md](../504-agent-vertical-slice.md) for CLI/operator guardrails, and [500-agent-operator.md](../500-agent-operator.md) for CRD reconciliation. Keep this document only as an audit trail of what was wrong with the original `core-platform-coder` package; all new fixes should be tracked against agent-developer epics and Transformation AgentDefinitions instead of patching the legacy package in isolation.

### Where to look now

* **Runtime fixes + tooling:** follow agent-developer for the devcontainer service, LangGraph DAG, and `develop_in_container` tool instead of the bespoke CLI wrappers discussed below.
  * Reference implementation: `services/inference/src/inference_service/agents/coding_agent/` + `services/inference/src/inference_service/tools/devcontainer/develop_in_container/`.
* **Risk + policy:** apply the risk-tier enforcement and tool grants from backlog 500; the ad-hoc security suggestions in this file are superseded by those policies.
* **CLI alignment:** align with the 484 CLI guardrail suite referenced in backlog 504; the “cheat-sheet” near the end of this file is informational only and is no longer authoritative for Ameide automation.

## Context

The `packages/ameide_core-platform-coder` component implements an LLM-based agent that analyses, plans and rewrites Python test files for better quality.  An in-depth review (see previous discussion) uncovered both architectural strengths and several robustness / correctness issues that we should address before the tool can be used reliably by developers or integrated into CI.

## Problem statement

1. **External-tool fragility** – hard dependence on `codex`, `claude` binaries and the OpenAI python SDK. Lack of graceful fallback causes the whole graph to fail when any tool is unavailable.
2. **Routing bug** – the conditional edge mapping in `graph.py` is missing the `"retry"` key, causing a runtime exception when the gatekeeper requests a retry.
3. **Security & stability** – long prompts are passed as raw CLI arguments which can hit the ARG_MAX limit and expose shell-escaping issues.
4. **Scoring accuracy** – the `compute_quality_score` algorithm can exceed its stated 0-10 range because weights are not normalised.
5. **State typing** – attributes such as `analysis_details` are added dynamically and are missing from the `State` dataclass, hampering static analysis.
6. **Gatekeeper coverage** – only lint + syntax are checked; pytest execution is not attempted, reducing confidence in rewritten tests.

## Proposal

### M1 – quick-win fixes

* Add graceful degradation helpers in `tools.py` that skip analysis / rewrite nodes when the external binary is not found (`shutil.which`).
* Patch `graph.py` to include the missing `"retry"` mapping.
* Pipe prompts through `stdin` for `CodexCLI` / `ClaudeCodeCLI` instead of positional arguments.

### M2 – scoring & state clean-up

* Re-balance `compute_quality_score` so weighted sum is capped at 10; expose per-metric weights in constants.
* Extend `State` dataclass with `analysis_details: Dict[str, Any]`.

### M3 – deeper validation

* Inside `gate_node`, spin up a temporary venv / workspace, write all candidate files and run `pytest -q` to catch behavioural regressions.

### M4 – developer experience

* Ship a `pyproject.toml` with `ruff`, `mypy`, `pytest` versions and a `make lint test` target.
* Document environment requirements & sample usage in `README.md`.

## Acceptance criteria

1. Running the CLI without external binaries prints warnings but still completes without traceback.
2. The retry path in the graph no longer raises KeyError and loops up to 3 times.
3. Lint (`ruff`) and unit tests in `__tests__` pass.
4. A sample refactor run on `sample_test.py` completes and, when external tools are available, produces a parametrised test file that passes `pytest`.

## Stakeholders

* Core-platform-coder maintainers
* QA automation team
* Dev-experience group

## Links

* In-depth review message (2025-07-25)









Below is a vendor‑documented, step‑by‑step cheat‑sheet that shows **how to drive each CLI from scripts or other automation**.  Every bullet is backed by an excerpt from the official docs or the project’s canonical README.

---

## 1 · OpenAI **Codex CLI**

| What you need                            | How to do it                                                                      | Doc proof                                      |
| ---------------------------------------- | --------------------------------------------------------------------------------- | ---------------------------------------------- |
| **Install once**                         | `npm install -g @openai/codex` <br>*(or `brew install codex`)*                    | ([OpenAI Help Center][1])                      |
| **Authenticate non‑interactively**       | Export the key **before** calling the CLI: <br>`export OPENAI_API_KEY=<your_key>` | ([GitHub][2])                                  |
| **Run head‑less (CI / cron / Makefile)** | `codex exec --full-auto "update CHANGELOG for next release"`                      | ([GitHub][2])                                  |
| **Select a model & mute prompts**        | `codex exec -m o3 --ask-for-approval never "lint src/"`                           | model & approval flags in README ([GitHub][2]) |
| **Capture machine‑readable logs**        | Prefix run with a log level: <br>`RUST_LOG=codex_core=debug codex exec "...""`    | ([GitHub][2])                                  |

> **Automation pattern**

```bash
#!/usr/bin/env bash
set -euo pipefail
export OPENAI_API_KEY="$OPENAI_KEY"

codex exec --full-auto \
           --model o3 \
           --ask-for-approval never \
           "refactor tests/ to pytest"
```

Codex exits **0** on success and prints the unified diff, so you can pipe it to `git apply` or email it.

---

## 2 · Anthropic **Claude CLI**

| What you need                  | How to do it                                         | Doc proof                  |
| ------------------------------ | ---------------------------------------------------- | -------------------------- |
| **Fire‑and‑forget invocation** | `claude -p "Explain this function"`                  | ([Anthropic][3])           |
| **Pipe code straight in**      | `cat file.py \| claude -p "Document this"`           | ([Anthropic][3])           |
| **Structured output**          | `claude -p "Return JSON" --output-format json`       | ([Anthropic][3])           |
| **Stream JSON for long jobs**  | `claude -p --output-format stream-json "large task"` | flag list ([Anthropic][3]) |
| **Resume previous session**    | `claude -c -r <session‑id> -p "continue"`            | ([Anthropic][3])           |

Because `--output-format=json` dumps a single JSON blob to **stdout**, you can:

```bash
title=$(claude -p "summarize" --output-format json | jq -r .content)
echo "Title: $title"
```

All without opening an interactive REPL.

---

## 3 · Google **CLI** options

### 3·A  Google Cloud **gcloud CLI**

| What you need                                | How to do it                                                                          | Doc proof                            |                                             |
| -------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------ | ------------------------------------------- |
| **Turn off prompts (safe for cron/Jenkins)** | `gcloud --quiet compute instances list`                                               | ([Google Cloud][4])                  |                                             |
| **Global disable via config**                | `gcloud config set disable_prompts true`                                              | ([Google Cloud][5])                  |                                             |
| **Get JSON you can pipe to `jq`**            | `gcloud projects list --format=json`                                                  | formatting flags ([Google Cloud][6]) |                                             |
| **Auth for unattended jobs**                 | Use a **service‑account key** and `gcloud auth activate-service-account --key-file=…` | scripting guide ([Google Cloud][7])  |                                             |
| **Quiet install in Docker or CI**            | \`curl [https://sdk.cloud.google.com](https://sdk.cloud.google.com)                   |  bash -- --quiet\`                   | non‑interactive install ([Google Cloud][8]) |

> **Automation snippet**

```bash
gcloud --quiet auth activate-service-account \
       --key-file="$SA_KEY" \
       --project="$PROJECT_ID"

gcloud --quiet functions deploy my-func \
       --runtime python311 --trigger-http \
       --format=json | jq .
```

### 3·B  Google **Gemini CLI** (AI agent)

| What you need              | How to do it                                      | Doc proof      |
| -------------------------- | ------------------------------------------------- | -------------- |
| **One‑shot run via `npx`** | `npx https://github.com/google-gemini/gemini-cli` | ([GitHub][9])  |
| **Global install**         | `npm install -g @google/gemini-cli` → `gemini`    | ([GitHub][9])  |
| **Authenticate once**      | `export GEMINI_API_KEY=<your_key>`                | ([GitHub][10]) |

After that you can script it exactly like Codex/Claude:

```bash
echo "Write a Rust iterator" \
  | gemini --output-format=json > idea.json
```

---

### Take‑away cheat‑sheet

| CLI        | Non‑interactive flag        | JSON/Structured output               | Auth env‑var              |
| ---------- | --------------------------- | ------------------------------------ | ------------------------- |
| **Codex**  | `codex exec`                | prints unified diff & logs           | `OPENAI_API_KEY`          |
| **Claude** | `claude -p`                 | `--output-format json / stream-json` | `ANTHROPIC_API_KEY` (SDK) |
| **gcloud** | `--quiet` + `--format=json` | JSON via `--format`                  | ADC or `SA` key           |
| **Gemini** | stdin → `gemini`            | `--output-format=json` (CLI flag)    | `GEMINI_API_KEY`          |

With these documented switches you can drop any of the CLIs into **shell scripts, Makefiles, CI jobs, or Python `subprocess` calls** and get clean, machine‑parsable output without ever touching an interactive TUI.

[1]: https://help.openai.com/en/articles/11096431 "OpenAI Codex CLI – Getting Started | OpenAI Help Center"
[2]: https://github.com/openai/codex "GitHub - openai/codex: Lightweight coding agent that runs in your terminal"
[3]: https://docs.anthropic.com/en/docs/claude-code/cli-reference "CLI reference - Anthropic"
[4]: https://cloud.google.com/sdk/gcloud?utm_source=threadsgpt.com "gcloud CLI overview | Google Cloud SDK Documentation"
[5]: https://cloud.google.com/sdk/docs/configurations?utm_source=threadsgpt.com "Managing gcloud CLI configurations | Google Cloud SDK ..."
[6]: https://cloud.google.com/sdk/docs/enabling-accessibility-features?utm_source=threadsgpt.com "Enabling accessibility features | Google Cloud SDK Documentation"
[7]: https://cloud.google.com/sdk/docs/scripting-gcloud "Scripting gcloud CLI commands  |  Google Cloud SDK Documentation"
[8]: https://cloud.google.com/sdk/docs/downloads-interactive?utm_source=threadsgpt.com "Using the Google Cloud CLI installer"
[9]: https://github.com/google-gemini/gemini-cli "GitHub - google-gemini/gemini-cli: An open-source AI agent that brings the power of Gemini directly into your terminal."
[10]: https://github.com/reugn/gemini-cli "GitHub - reugn/gemini-cli: A command-line interface (CLI) for Google Gemini"
