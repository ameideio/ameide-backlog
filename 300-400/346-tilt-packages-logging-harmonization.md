# Backlog 346: Tilt Package Logging Harmonization

## Context
- Tilt pipelines for `ameide_core_proto`, language SDKs, and package registries now run reliably, but their console output remains inconsistent.
- Package-oriented Tilt resources (`packages:health`, `packages:status`, `packages:*-(publish|smoke)`) and infra helpers in `infra/kubernetes/scripts/` each define bespoke log formats, emojis, and success markers.
- The divergence complicates scanning Tilt panes, hides failures in the noise of downstream commands, and makes it harder to extend workflows without re-learning each script’s conventions.

## Problem Statement
- There is no shared logging utility across shell/Python helpers; `infra/kubernetes/scripts/common.sh` isn’t sourced and includes side effects (global `set -euo pipefail`, ANSI color codes) that discourage reuse.
- Tilt resources emit unprefixed output (e.g., raw `pnpm` logs, kubectl banners) before any human-friendly summary, reducing visibility in the Tilt UI.
- Publish/smoke pairs maintain temporary state in ad-hoc files and echo success/failure with different wording, making cross-language comparisons harder.

## Goals
- Provide a lightweight logging shim usable from shell, Node, and Python helpers that enforces consistent prefixes and start/finish markers.
- Normalize publish/smoke scripts so they announce the snapshot version/tag, record state in a shared JSON schema, and emit identical success/failure summaries.
- Update infra Tilt targets (`packages:health`, `packages:status`) to adopt the shared logger and finish with clear, prefixed status lines.
- Document the logging contract so future helpers follow the same structure.

## Non-Goals
- Rewriting business logic inside publish, smoke, or health scripts beyond logging/state-hand-off improvements.
- Introducing new external dependencies or colorized output (ANSI codes stay optional and off by default).

## Plan
- Design shared logger API
  - [x] Author `scripts/lib/logging.sh` exposing `log_step`, `log_info`, `log_ok`, `log_warn`, `log_fail` with an optional `LOG_PREFIX`.
  - [x] Provide Node and Python adapters that mirror the shell contract (e.g., `scripts/lib/logging.mjs`, `scripts/lib/logging.py`).
- Migrate package Tilt helpers
  - [x] Refactor `infra/kubernetes/scripts/tilt-packages-health-summary.sh` to source the logger and replace bracketed prints.
  - [x] Wrap `scripts/infra/packages-status.sh` so both the kubectl wrapper and the in-cluster Python script emit `[packages:status] …`.
  - [x] Update `scripts/tilt-packages-*-publish.sh` to call release helpers via a `run_step` wrapper, emit start/finish logs, and write `tmp/tilt-packages/<name>.json`.
  - [x] Update `scripts/tilt-packages-*-smoke.sh` to read the JSON state, surface the snapshot version, and use the shared logger for success/failure.
- Harmonize SDK build targets
  - [ ] Add logger usage around `pnpm build` / `uv run` / `node scripts/sync-proto.mjs` so Tilt panes begin with `[packages:*] start`, end with `[packages:*] ok`, and hide intermediary noise unless `VERBOSE=1`.
  - [ ] Review `infra/kubernetes/scripts/common.sh`; move reusable pieces into the shared logger (without implicit `set -euo pipefail`), or delete if superseded.
- Documentation & rollout
  - [ ] Document the logging/playbook in `docs/tilt-packages.md` (or update an existing README) with examples of canonical output.
  - [ ] Coordinate with CI to ensure any workflows invoking these scripts continue to succeed (env vars, new JSON state files).
  - [ ] Track adoption by verifying Tilt UI panes for `ameide_core_proto`, `ameide_sdk_*`, and `packages:*` now share the new prefix and concise summaries.

## Verification
- Tilt UI shows consistent `[packages:*]` log headers and end-state lines across health, status, publish, and smoke resources.
- Smoke scripts fail fast with prefixed `log_fail` output and include the offending version/tag in the message.
- CI workflows consuming the scripts run without modification beyond sourcing the logger and respecting the JSON state hand-off.
