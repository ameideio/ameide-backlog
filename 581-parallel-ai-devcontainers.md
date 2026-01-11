# 581 – Parallel AI Agents in VS Code Dev Containers

**Status:** Draft  
**Owner:** Platform / Developer Experience  
**Related backlogs:** [435-remote-first-development.md](435-remote-first-development.md), [492-telepresence-verification.md](492-telepresence-verification.md), [492-telepresence-reliability.md](492-telepresence-reliability.md), [433-codex-cli-057.md](433-codex-cli-057.md)

## Decision (golden path)

Keep the existing “generic” devcontainer workflow for normal human development, and add parallel “agent slot” devcontainers for isolated work.

Add three *additional* agent slots for parallel work; they do not replace the generic workflow:

- Slots: `agent-01`, `agent-02`, `agent-03`
- One clone per slot (required; Git worktrees are deprecated for this repo’s Dev Container workflow)
- One VS Code window per slot → “Reopen in Container”
- One workspace checkout branch per slot (created by the helper scripts): `ameide-agent-01|02|03`
- One PR branch per change (recommended namespace pattern): `agent/dev/<agent_slot>/<topic>` → PRs merge into `dev` (protected; PR-only)
  - Never push directly to `dev`/`main`; always land changes via PR (see `backlog/400-agentic-development.md`).

Convenience:
- `./tools/dev/create-agent-clones.sh` creates:
  - a dedicated `ameide-dev` clone (optional “integrator seat” on `dev` for merging PRs), and
  - `ameide-agent-01..03` clones (parallel agent slots).

### Scope: developer mode vs platform mode

This backlog is the **developer mode** story: a human (or an interactive coding agent) runs inside VS Code Dev Containers and iterates locally.

The **platform mode** story (role-based ephemeral jobs from a queue, durable evidence, and a WorkRequest lifecycle) is specified in the 527 suite; devcontainers are one possible execution substrate, but not the canonical orchestration mechanism:
- `backlog/527-transformation-capability.md`
- `backlog/527-transformation-integration.md`
- `backlog/527-transformation-process.md`

## One agent identity everywhere

Devcontainer-provided identity:
- `AMEIDE_AGENT_ID=${localWorkspaceFolderBasename}` (agent devcontainer)

Recommended agent folder names (clones) so `${localWorkspaceFolderBasename}` carries the slot:
- `ameide-agent-01`, `ameide-agent-02`, `ameide-agent-03`

Tooling convention:
- When an `AMEIDE_AGENT_ID` contains `agent-01|agent-02|agent-03`, scripts should treat that substring as the canonical slot ID (e.g., `ameide-agent-01` → `agent-01`).

## Git layout recommendation (Dev Containers)

We standardize on **separate clones** for parallel agent slots.

Rationale: Git worktrees store a `.git` *file* that points at a shared “real” git directory. In Dev Containers, that path frequently becomes invalid across host/container boundaries and leads to “repo not initialized” / missing-index style failures. Clones avoid that entire class of issues.

## Telepresence + Tilt: same-workload parallelism

### Baseline: unique intercept names per agent

When `AMEIDE_AGENT_ID` is set, dev tooling should create intercepts using:

- intercept name: `${workload}-${agent_slot}` (where `agent_slot` is derived from `AMEIDE_AGENT_ID`)
- target workload: `--workload ${workload}`

This avoids **name collisions** when multiple agents are connected at once.
Important: unique names alone do **not** guarantee true parallel intercepts to the same workload; you still need request routing separation.

### Recommended (HTTP services): header-filtered intercepts

Unique names are necessary but not sufficient if both intercept “all traffic” for the same service.

For HTTP services, use Telepresence HTTP intercept filtering so each agent only receives matching traffic:

- set `AMEIDE_TELEPRESENCE_HTTP_FILTER=1`
- tooling adds: `--mechanism http --http-header "X-Ameide-Agent=${agent_slot}"`
- the caller must send that header for requests to match

Support check (practical):

- `telepresence intercept --help` should include `--http-header` (or a similar HTTP filter flag). If you are not using the repo-pinned Telepresence toolchain, verify the feature exists before relying on parallel intercept routing.

### Fallback: coordination (single-owner intercept)

If HTTP header filtering is not available, or the service is not HTTP:

- Keep unique intercept names (to prevent collisions), but treat the intercept as **single-owner per workload**.
- Coordinate so only one agent at a time intercepts a given workload; other agents should use remote-first URLs or work on different services/slices.

## VS Code port-forward collisions

Multiple VS Code windows can’t forward the same container port to the same local port.

Guidance:
- Prefer remote-first URLs (`https://*.dev.ameide.io`) during intercepts.
- If forwarding is required (application ports), rely on VS Code auto-remap or assign per-agent local ports. Do not forward Codex auth on `1455`; share `~/.codex/auth.json` instead.

## Codex CLI auth callback collision (`localhost:1455`)

Codex CLI’s “Sign in with ChatGPT” login flow runs a local callback server on `localhost:1455`.

Policy (v1; canonical):
- Authenticate once and share `~/.codex/auth.json` across all agent containers (avoid running concurrent login flows).
- Do not use `OPENAI_API_KEY` / `codex login --with-api-key` in devcontainers (defer to a future “automation/headless auth” track).

Notes:

- Treat `localhost:1455` as fixed for this login flow; avoid “per-window remap” guidance. The supported posture is “authenticate once, then share `auth.json`”.

Rationale: shared `~/.codex/auth.json` avoids local callback server contention and keeps auth handling consistent across parallel agent slots.

## Bootstrap side effects (multi-container)

Repeated bootstrap (Azure login, kube context wiring, port-forwards) across N containers creates churn and coupling.

Recommendation:
- Use `AMEIDE_BOOTSTRAP_PROFILE=primary|agent`
  - `primary`: human devcontainer; bootstrap is typically run manually because `az login --use-device-code` is interactive (set `AMEIDE_AUTO_BOOTSTRAP=1` to run it automatically during postCreate).
  - `agent`: skips bootstrap side effects by default; if `~/.config/ameide/context.env` is missing in that slot, run `tools/dev/bootstrap-contexts.sh`.

### Volume permissions

Named Docker volumes are initialized with root ownership when first created. The `.devcontainer/postCreate.sh` script fixes ownership for all mounted config volumes (including `~/.codex`, `~/.kube`, `~/.azure`, `~/.config/ameide`) using the VS Code Dev Containers recommended approach:

- Ensures mount points exist with `sudo mkdir -p`
- Fixes ownership with `sudo chown -R $(id -u):$(id -g)`
- Runs early in container startup (before pnpm install and bootstrap)

This prevents "Permission denied" errors when Codex CLI, kubectl, Azure CLI, or other tools try to write configuration files.

Reference: [VS Code Dev Containers - Persist bash history](https://code.visualstudio.com/remote/advancedcontainers/persist-bash-history)

## Implementation (repo)

- Devcontainer configs:
  - `.devcontainer/devcontainer.json`: generic config; keeps `~/.codex` shared (auth), but isolates `~/.azure`, `~/.kube`, and `~/.config/ameide` per DevContainer to avoid context/subscription cross-talk.
  - `.devcontainer/agent/devcontainer.json`: agent config; same isolation posture and sets `AMEIDE_BOOTSTRAP_PROFILE=agent` and `AMEIDE_AGENT_ID=${localWorkspaceFolderBasename}`.
- Bootstrap:
  - `.devcontainer/postCreate.sh` uses `AMEIDE_BOOTSTRAP_PROFILE` and skips remote context bootstrapping when `AMEIDE_BOOTSTRAP_PROFILE=agent` to avoid repeated side effects.
- Telepresence entrypoint:
  - **Update (2026-01):** bash wrappers were removed; use the Ameide CLI as the single entrypoint:
    - `ameide dev inner-loop verify` (Telepresence + header-filtered routing)
    - `ameide dev inner-loop up/down` (UI hot reload + intercept)

## Acceptance criteria

1. Generic devcontainer remains the default for `dev`/`main`.
2. Two+ agent devcontainers run concurrently without working-tree interference.
3. Same-workload parallelism is supported for HTTP services via header-filtered intercepts when supported; otherwise, single-owner coordination per workload is the fallback.
4. Codex CLI auth avoids `localhost:1455` collisions by default and defers API-key login in devcontainers (future automation track).
5. Bootstrap supports an explicit “agent” mode that avoids repeated side effects.
