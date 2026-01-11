---
title: "629 – AmeideDevContainerService E2E + smoke test specification (Coder workspace)"
status: draft
owners:
  - platform-devx
  - qa
  - gitops
created: 2026-01-11
---

# 629 – AmeideDevContainerService E2E + smoke test specification (Coder workspace)

## 0) Purpose

Define an automated validation flow for the Coder-based human workspace (626/628) that proves:

- workspace provisioning works
- VS Code (Web) is reachable
- git workflow works (branch → push → PR)
- (optional/extended) Codex can run inside the workspace to produce a deterministic change

This is a **platform smoke/E2E**, not `ameide dev inner-loop-test` (which is Telepresence-centric per `backlog/621-ameide-cli-inner-loop-test.md`).

## 0.0 Decision: Coder Community Edition (CE) only

Tests must not rely on paid Coder features (e.g., browser-only enforcement, workspace proxies).

## 0.1 Scope clarifications

- This test validates the **Coder workspace platform** and the **devcontainer parity** for `github.com/ameideio/ameide`.
- This test does not validate Telepresence inside a Kubernetes-hosted workspace because Telepresence is **not supported** in this environment. Telepresence remains validated by existing CLI workflows/runbooks.
- Any “branch → push → PR” step must run against a **bot-writable repo**; it must not assume write access to `ameideio/ameide`.

## 1) Test types

### 1.1 Smoke (fast, minimal secrets)

Goal: catch “Coder is broken” quickly.

Checks:

1. Create workspace from template.
2. Verify the app proxy path end-to-end (outside the workspace):
   - fetch the code-server app URL via Coder API/CLI
   - perform an authenticated HTTP request to the app URL and expect non-5xx
3. `coder ssh` into the workspace and verify:
   - repo checkout works
   - toolchain presence (git + gh; and any mandatory Ameide tooling)
4. Verify code-server health on localhost (via `curl http://127.0.0.1:<port>/healthz`).
5. Delete workspace.

Auth expectations:

- Humans authenticate via Keycloak OIDC in the browser.
- Automation uses a Coder API token / session token (not interactive browser auth).

The smoke must also validate that OIDC is configured by checking:

- Coder is configured to use the Keycloak issuer (configuration-level assertion; do not attempt browser automation).
- The deployment is past first-run setup (bootstrap complete):
  - `GET /api/v2/users/first` returns “initial user has already been created”.

### 1.2 Full E2E (includes PR + optional Codex)

Goal: prove the full developer workflow is viable end-to-end.

Steps (high-level):

1. Create workspace.
2. Clone a bot-writable repo (or the target repo if policy allows).
3. Create branch.
4. Make a deterministic change:
   - minimal path: append a line to `README.md` via shell
   - extended path: use Codex non-interactive mode to produce the same change
5. Commit, push branch.
6. Create PR (GitHub CLI).
7. Delete workspace (always).

Implementation reference (current):

- GitHub Actions runner workflow: `.github/workflows/coder-devcontainer-e2e.yaml`
- The workflow validates both templates:
  - `ameide-dev` (smoke; no PR step)
  - `ameide-gitops` (full E2E, including Codex + PR to this repo)

## 2) Secrets and identities

### 2.1 GitHub

Use a dedicated bot token with:

- permission to push branches
- permission to create PRs

### 2.2 Codex (optional/extended)

Preferred posture for automation inside Kubernetes is file-based auth, not interactive login:

- `Secret/codex-auth` providing `auth.json`
- `CODEX_HOME` set to the mount location

Related: `backlog/613-codex-auth-json-secret.md`, `backlog/527-transformation-e2e-sequence-execution-implementation.md`.

If we intentionally choose API-key auth for CI, document it as a separate “automation auth mode” and reconcile with the devcontainer auth policy in `backlog/433-codex-cli-057.md`.

## 3) Environment scope

- Runs only against **AKS dev**.
- Must not require staging/prod access.
- Coder access URL should follow: `coder.dev.ameide.io` (future envs: `coder.<env>.ameide.io`).

## 4) Pass/fail criteria

Smoke pass:

- workspace can be created and deleted
- code-server health endpoint responds
- repo operations (clone) succeed

E2E pass:

- branch pushed from inside workspace
- PR created successfully
- (if enabled) Codex runs and produces the expected change

## 5) Operational requirements

- Test must be self-cleaning (workspace deletion on failure).
- Test must not print secrets.
- Test must run under restrictive NetworkPolicy posture once egress allowlists are in place (GitHub + coderd + DNS + optional OpenAI).
- Workspace storage assumptions for tests: ephemeral (node-backed) is acceptable; tests must not assume persistence across workspace recreation.
- Workspace auto-stop must be enabled by default (do not rely on tests to stop idle workspaces).

## 6) Networking prerequisites for automation

- `CODER_ACCESS_URL` must be reachable from the test runner, and ingress must support WebSockets.
- If the chosen Coder CLI connection mode requires direct/UDP connectivity, the test runner environment must allow it; otherwise prefer running tests from an in-cluster runner network where the supported connection mode is known to work.
