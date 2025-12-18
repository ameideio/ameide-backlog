# 581 – Parallel AI Agents in VS Code Dev Containers

**Status:** Draft  
**Owner:** Platform / Developer Experience  
**Related backlogs:** [435-remote-first-development.md](435-remote-first-development.md), [492-telepresence-verification.md](492-telepresence-verification.md), [492-telepresence-reliability.md](492-telepresence-reliability.md), [433-codex-cli-057.md](433-codex-cli-057.md)

## Decision (golden path)

Keep the existing “generic” devcontainer workflow for normal human development on `dev`/`main`.

Add three *additional* agent slots for parallel work; they do not replace the generic workflow:

- Slots: `agent-01`, `agent-02`, `agent-03`
- One worktree per slot (or separate clones)
- One VS Code window per slot → “Reopen in Container”
- One branch per slot (e.g., `ameide-agent-01`) or per-slot topic branch (e.g., `agent-01/<topic>`) → PRs merge into `dev`

Convenience:
- `./tools/dev/create-agent-worktrees.sh` creates `ameide-agent-01..03` worktrees (and corresponding per-slot branches).

## One agent identity everywhere

Devcontainer-provided identity:
- `AMEIDE_AGENT_ID=${localWorkspaceFolderBasename}` (agent devcontainer)

Recommended worktree folder names (so `${localWorkspaceFolderBasename}` carries the slot):
- `ameide-agent-01`, `ameide-agent-02`, `ameide-agent-03`

Tooling convention:
- When an `AMEIDE_AGENT_ID` contains `agent-01|agent-02|agent-03`, scripts should treat that substring as the canonical slot ID (e.g., `ameide-agent-01` → `agent-01`).

## Telepresence + Tilt: same-workload parallelism

### Baseline: unique intercept names per agent

When `AMEIDE_AGENT_ID` is set, dev tooling should create intercepts using:

- intercept name: `${workload}-${agent_slot}` (where `agent_slot` is derived from `AMEIDE_AGENT_ID`)
- target workload: `--workload ${workload}`

This avoids name collisions when multiple agents are connected at once.

### Recommended (HTTP services): header-filtered intercepts

Unique names are necessary but not sufficient if both intercept “all traffic” for the same service.

For HTTP services, use Telepresence HTTP intercept filtering so each agent only receives matching traffic:

- set `AMEIDE_TELEPRESENCE_HTTP_FILTER=1`
- tooling adds: `--mechanism http --http-header "X-Ameide-Agent=${agent_slot}"`
- the caller must send that header for requests to match

## VS Code port-forward collisions

Multiple VS Code windows can’t forward the same container port to the same local port.

Guidance:
- Prefer remote-first URLs (`https://*.dev.ameide.io`) during intercepts.
- If forwarding is required, rely on VS Code auto-remap or assign per-agent local ports.

## Codex CLI auth callback collision (`localhost:1455`)

Codex CLI’s “Sign in with ChatGPT” login flow runs a local callback server on `localhost:1455`.

Default recommendation:
- Authenticate once and share `~/.codex/auth.json` across all agent containers (avoid running concurrent login flows).

Deprecated:
- Avoid using `OPENAI_API_KEY` / `codex login --with-api-key` for these devcontainers; prefer shared `~/.codex/auth.json` so we don’t multiply long-lived secrets across agent environments.

## Bootstrap side effects (multi-container)

Repeated bootstrap (Azure login, kube context wiring, port-forwards) across N containers creates churn and state coupling.

Recommendation:
- Use `AMEIDE_BOOTSTRAP_PROFILE=primary|agent`
  - `primary`: does the full bootstrap.
  - `agent`: skips side effects and assumes shared state is already present.

### Volume permissions

Named Docker volumes are initialized with root ownership when first created. The `.devcontainer/postCreate.sh` script fixes ownership for all shared volumes (`~/.codex`, `~/.kube`, `~/.azure`, `~/.config/ameide`) using the VS Code Dev Containers recommended approach:

- Ensures mount points exist with `sudo mkdir -p`
- Fixes ownership with `sudo chown -R $(id -u):$(id -g)`
- Runs early in container startup (before pnpm install and bootstrap)

This prevents "Permission denied" errors when Codex CLI, kubectl, Azure CLI, or other tools try to write configuration files.

Reference: [VS Code Dev Containers - Persist bash history](https://code.visualstudio.com/remote/advancedcontainers/persist-bash-history)

## Implementation (repo)

- Devcontainer configs:
  - `.devcontainer/devcontainer.json`: generic config, mounts shared volumes for `~/.codex`, `~/.azure`, `~/.kube`, `~/.config/ameide`.
  - `.devcontainer/agent/devcontainer.json`: agent config, sets `AMEIDE_BOOTSTRAP_PROFILE=agent` and sets `AMEIDE_AGENT_ID=${localWorkspaceFolderBasename}`.
- Bootstrap:
  - `.devcontainer/postCreate.sh` uses `AMEIDE_BOOTSTRAP_PROFILE` (planned: skip `tools/dev/bootstrap-contexts.sh` when `AMEIDE_BOOTSTRAP_PROFILE=agent` to avoid repeated side effects).
- Telepresence wrappers:
  - `scripts/telepresence/intercept_service.sh` uses agent-aware intercept naming and optional HTTP filtering (`AMEIDE_TELEPRESENCE_HTTP_FILTER=1`).
  - `tools/dev/telepresence.sh intercept` is the generic helper entrypoint (planned: mirror the same agent-aware naming/filtering behavior).

## Acceptance criteria

1. Generic devcontainer remains the default for `dev`/`main`.
2. Two+ agent devcontainers run concurrently without working-tree interference.
3. Same-workload parallelism is supported for HTTP services via header-filtered intercepts.
4. Codex CLI auth avoids `localhost:1455` collisions by default and deprecates API-key login in devcontainers.
5. Bootstrap supports an explicit “agent” mode that avoids repeated side effects.
