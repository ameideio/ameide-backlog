# Agentic Development Workflow

## Goal
Consistently develop on the `dev` branch, land changes via PR into `main`, and keep your local environment clean by resetting `dev` to the tracked `origin/main` after each merge.

## Working in `dev`
- Always branch from a clean `dev` that tracks `origin/main`.
  - `git checkout dev && git pull --rebase`
- Keep the working tree clean; avoid lingering changes across tasks.
- Assume `gh` CLI is authenticated, `argocd` CLI is authenticated, and Tilt is running on port `10350`; do not restart existing services without explicit authorization.

## Implement & Test
- Make changes on `dev`. Run the relevant workspace test suites (Go, TS, Py) before committing.
- Keep commits focused; avoid mixing unrelated changes.

## Commit & Push
- Commit to `dev`: `git commit -am "<message>"` (or stage explicitly).
- Push `dev`: `git push origin dev`.

## Open PR to `main`
- Create PR from `dev` → `main` **only when requested/approved** (e.g., `gh pr create --base main --head dev`).
- CI should run; address failures on `dev`, then push again.

## Merge
- Merge via PR (squash preferred) with branch protection on `main`.
- After merge, delete remote PR branch if desired.

## Reset local `dev` to continue
- Sync `main`: `git checkout main && git pull --rebase`.
- Recreate a fresh `dev` from `origin/main`:
  - `git checkout -B dev origin/main`
  - `git push origin dev` (fast-forward) if you keep `dev` published.
- Confirm clean state: `git status` should be empty.

## Hygiene & Recovery
- Never develop directly on `main` (protected).
- If a rebase is interrupted, `git rebase --abort` to reset; ensure `git status` shows no rebasing note.
- Use `git reflog` to recover lost work; create a rescue branch (`git branch rescue/<date> <sha>`) if needed.
- Avoid local `go.work`/`replace` changes in commits; reset to repository defaults before push.

## CI Expectations
- PRs must be green (unit/integration as configured) before merge.
- Docker/SDK workflows may require registry creds; keep secrets configured in repo settings.
