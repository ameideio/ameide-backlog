---
title: "692 – Implement 691 CLI surface (test/e2e + dev) and cluster enablement"
status: draft
owners:
  - platform-devx
  - test-infra
created: 2026-01-17
suite: "650-agentic-coding"
source_of_truth: false
parents:
  - 691-cli-surface-test-dev-e2e-smoke-dev.md
---

# 692 – Implement 691 CLI surface (test/e2e + dev) and cluster enablement

This backlog is the **implementation plan** for the normative CLI surface in `backlog/691-cli-surface-test-dev-e2e-smoke-dev.md`.

Scope:
- CLI changes live in `ameideio/ameide` (Go CLI + `www_ameide_platform`).
- Cluster enablement lives in `ameideio/ameide-gitops` (RBAC, NetworkPolicy, admission policy, janitors).

Non-scope:
- Re-designing preview environments (owned elsewhere).

---

## Milestone A — CLI surface alignment (no functional change beyond command wiring)

**Goal:** align the CLI entrypoints with 691/430v2/621 without changing test semantics.

Tasks (repo: `ameideio/ameide`)
- Add `ameide test e2e` subcommand:
  - Runs Playwright E2E only (deployed-system truth).
  - Resolves base URL by reading `AUTH_URL` from `ConfigMap/www-ameide-platform-config` in `AMEIDE_ENV_NAMESPACE`.
  - Reads personas from `Secret/playwright-int-tests-secrets` in `AMEIDE_ENV_NAMESPACE`.
  - Emits mandatory JUnit evidence and redacts secrets from artifacts.
- Change `ameide test` to run Phase 0/1/2 only.
- Ensure `ameide test ci` remains Phase 0/1/2 only.
- Update CLI help text and remove any legacy docs implying Phase 3 is part of `ameide test`.

Acceptance criteria
- `ameide test` runs with no Kubernetes access.
- `ameide test e2e` fails fast with a clear error if `AMEIDE_ENV_NAMESPACE` is missing or required objects are not readable.

---

## Milestone B — Make Phase 3/Gateway attachment robust (parentRef resolution)

**Goal:** remove brittle hardcoding so `ameide dev` / Phase 3 can attach correctly in AKS environments.

Tasks (repo: `ameideio/ameide`)
- Replace hardcoded Gateway parentRef with dynamic resolution:
  - Inspect the environment namespace for the baseline platform HTTPRoute and derive the `(gateway name, namespace, sectionName)`.
- Improve diagnostics:
  - When attachment fails, print a short “likely causes” list (allowedRoutes selector, missing RBAC, Gateway not found).

Acceptance criteria
- In dev/AKS, the created HTTPRoute reaches `Accepted=True` and `ResolvedRefs=True` without manual edits.

---

## Milestone C — `ameide dev` (human UI inner loop) with canonical-host auth bounce

**Goal:** provide fast hotreload UI access with a stable UX and without relying on IdP wildcard redirect URIs.

Tasks (repo: `ameideio/ameide`)
- Add `ameide dev` command:
  - Runs inside a workspace pod; starts `pnpm -C services/www_ameide_platform dev` on port 3001.
  - Creates per-workspace Service + HTTPRoute in the workspace namespace.
  - Uses hostname `platform-dev-<workspace-env-id>.dev.ameide.io` (workspace-derived, stable).
  - Adds TTL annotations/labels per 691.
  - Cleans up Service/HTTPRoute and stops the dev server on exit.
- Add (or confirm) canonical-host auth bounce support in `www_ameide_platform`:
  - Keep IdP redirect URI static on the canonical host (e.g. `platform.dev.ameide.io`).
  - Implement safe `return_to`/`state` allowlist for redirects back to `platform-dev-*.dev.ameide.io`.
  - Ensure cookie strategy supports `.dev.ameide.io` (no `__Host-` cookies if `Domain=` is required).
- Add fail-fast validation:
  - Verify bounce config is enabled before printing the dev URL.

Acceptance criteria
- A developer can run `ameide dev` in Coder and immediately browse the dev URL and complete login.

---

## Milestone D — Cluster enablement (GitOps) for safe innerloop routing

**Goal:** make innerloop routing safe-by-policy and reliable-by-default (no manual cluster mutation).

Tasks (repo: `ameideio/ameide-gitops`)
- Admission policy for workspace-created `HTTPRoute`:
  - Enforce hostname allowlist `platform-dev-*.dev.ameide.io`.
  - Enforce parentRef allowlist (resolved platform Gateway only).
  - Enforce same-namespace `Service` backendRefs only.
  - Require TTL labels/annotations.
- Janitor for leaked innerloop resources:
  - Periodically delete expired `HTTPRoute`/`Service` labeled `ameide.devx/owner=ameide-cli` in workspace namespaces.
- NetworkPolicy:
  - Ensure Envoy dataplane pods can reach workspace devserver `:3001` (and only those sources).
- RBAC:
  - Ensure workspace SAs can create/delete Services and HTTPRoutes in their namespace and read required env ConfigMaps/Secrets.

Acceptance criteria
- Workspace cannot create arbitrary HTTPRoutes (policy blocks unsafe shapes).
- Leaked resources are cleaned up automatically after TTL.

---

## Milestone E — Che/vCluster bridging decision and implementation

**Goal:** define and implement the supported path(s) for Che workspaces when the workspace kube API is not the host cluster.

Decision (choose one, document explicitly in 691 + implement here)
- Option 1: Mirror/sync required env ConfigMaps/Secrets and selected Gateway API resources into the vCluster.
- Option 2: Provide host-cluster kubeconfig/token to the workspace and force CLI to use it for cluster-only commands.

Tasks (repo: `ameideio/ameide-gitops` and/or `ameideio/ameide`)
- Implement the chosen bridging mechanism.
- Add explicit CLI detection and fail-fast messaging when pointed at a cluster API that cannot satisfy the contracts.

Acceptance criteria
- `ameide test e2e` and `ameide dev` are either fully supported in Che/vCluster, or fail-fast with actionable guidance.

---

## Milestone F — E2E verification path and documentation

**Goal:** prove the end-to-end workflow is real and keep docs consistent.

Tasks
- Add a “developer quickstart” doc (location TBD) that shows:
  - `ameide test`
  - `ameide test e2e`
  - `ameide dev`
- Add/update backlog references where older surfaces are described.
- Run one end-to-end validation:
  - In Coder: `ameide dev` login + hotreload.
  - In Coder/Che: `ameide test e2e` against dev/preview.

Acceptance criteria
- One documented happy path exists for “fast UI loop” and “deployed truth E2E”, with the commands in 691.

