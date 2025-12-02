# 392 - Branch Flow (main ↔ feature ↔ PR)

## Goal
Provide a generic branch → PR → merge cycle that any contributor can follow to keep `main` stable while iterating safely.

## Workflow
1. **Sync baseline**
   - `git checkout main && git pull origin main`
   - Starting from the latest `main` reduces conflicts and ensures CI parity.
2. **Create feature branch**
   - `git checkout -b <feature-branch>`
   - Work in small, focused commits; keep the branch lifespan short.
3. **Push & open PR early**
   - `git push origin <feature-branch>`
   - `gh pr create --base main --head <feature-branch> --title "..." --body "..."`
   - Early PRs surface CI failures sooner and keep reviewers in the loop.
4. **Iterate with feedback**
   - Address CI or review comments with additional commits.
   - Push updates frequently to avoid large rebases; fetch + rebase onto `main` if needed.
5. **Merge cleanly**
   - Follow repo policy (e.g., squash merge) using `gh pr merge <id> --squash --delete-branch`.
   - Ensure the feature branch is removed remotely after merge.
6. **Re-sync**
   - `git checkout main && git pull origin main`
   - Start the next task from a fresh branch created off the updated `main`.

## Tips
- Communicate when touching high-churn files (workflows, configs) to avoid overlapping edits.
- Keep backlog notes up to date alongside code so context travels with the change.
- Use `gh run view <id> --log` / `gh run list` to observe CI output without waiting for reviewers.
