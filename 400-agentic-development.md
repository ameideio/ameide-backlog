# Agentic Development Workflow (PR funnel)

> Deprecated (historical): this doc assumes a `dev → main` promotion model and Telepresence-era “agent slots”.
>
> Superseded by:
> - trunk-based `main` + GitOps promotion policy (see `backlog/611-trunk-based-main-and-gitops-environment-promotion.md`)
> - internal-only, Coder-based model and CLI front doors in the 650 suite (`backlog/650-agentic-coding-overview.md`, `backlog/654-agentic-coding-cli-surface-coder.md`)

## Goal
Support predictable, auditable agentic development without direct writes to protected branches.

This workflow assumes `dev` and `main` are protected and **PR-only** (no direct pushes by humans or agents), and uses topic branches for all work.

## Branch model

- `dev` is the shared integration branch (protected; PR-only).
- `main` is the stable/release branch (protected; PR-only).
- All work happens on topic branches and lands into `dev` via PR.
- Promotion from `dev` → `main` is a separate PR (human controlled).

Recommended branch namespace patterns:
- Humans: `user/<handle>/<topic>`
- Agents (slot-based devcontainers): `agent/dev/<agent_slot>/<topic>` (see `backlog/581-parallel-ai-devcontainers.md`)

Guardrail posture (authoritative): see `backlog/527-transformation-capability.md` and `backlog/527-transformation-integration.md` for identity-scoped branch namespaces and server-side enforcement (rulesets, push rules, required checks).

## Working locally

- Start from a clean base branch:
  - `git fetch origin`
  - `git switch dev && git pull --rebase`
- Create a topic branch (never commit directly on `dev` or `main`):
  - `git switch -c agent/dev/agent-01/<topic>` (agents) or `git switch -c user/<handle>/<topic>` (humans)
- Keep the working tree clean; avoid lingering changes across tasks.
- Assume `gh` CLI is authenticated; do not restart cluster/tooling without explicit authorization.

## Implement & Test
- Make changes on your topic branch. Run the relevant workspace test suites (Go, TS, Py) before committing.
- Keep commits focused; avoid mixing unrelated changes.

## Commit & Push
- Commit to your topic branch: `git commit -am "<message>"` (or stage explicitly).
- Push your topic branch: `git push -u origin HEAD`.

## Open PR to `dev`

- Create PR from your topic branch → `dev` (e.g., `gh pr create --base dev --head <your-branch>`).
- CI should run; address failures on `dev`, then push again.

## Merge
- Merge via PR (squash preferred) with branch protection on `dev`.
- After merge, delete remote PR branch if desired.

## Promote `dev` → `main` (release/promotion)

- Only when requested/approved, open a PR `dev` → `main` (often by a release owner).
- Prefer merge queue / required checks as configured on `main`.

## Reset local state

- Sync `dev`: `git switch dev && git pull --rebase`.
- Delete stale local branches when done: `git branch -D <topic-branch>` (after merge).
- Confirm clean state: `git status` should be empty.

## Hygiene & Recovery
- Never develop directly on `main` (protected).
- Never develop directly on `dev` (protected; PR-only).
- If a rebase is interrupted, `git rebase --abort` to reset; ensure `git status` shows no rebasing note.
- Use `git reflog` to recover lost work; create a rescue branch (`git branch rescue/<date> <sha>`) if needed.
- Avoid local `go.work`/`replace` changes in commits; reset to repository defaults before push.

## CI Expectations
- PRs must be green (unit/integration as configured) before merge.
- Docker/SDK workflows may require registry creds; keep secrets configured in repo settings.
