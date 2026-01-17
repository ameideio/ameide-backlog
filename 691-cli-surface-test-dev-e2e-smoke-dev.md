---
title: "691 – CLI surface for test/e2e + human dev loop (no flags)"
status: draft
owners:
  - platform-devx
  - test-infra
created: 2026-01-17
suite: "650-agentic-coding"
source_of_truth: false
parents:
  - 430-unified-test-infrastructure-v2-target.md
  - 621-ameide-cli-inner-loop-test-v2.md
---

# 691 – CLI surface for test/e2e + human dev loop (no flags)

This backlog defines the **normative CLI front doors** for:

- Local-only verification (Phases 0/1/2).
- Deployed-system truth (Playwright E2E).
- A human-friendly UI dev loop in an in-cluster workspace (Coder/Che) without Telepresence.

This document is a **CLI surface spec** (user-facing contract + invariants). Implementation is tracked separately.

## Current implementation mismatches / risks (must be reconciled)

This section records known mismatches between the normative surface described below and the current implementation
in `ameideio/ameide` (and cluster enablement in `ameideio/ameide-gitops`).

1) **`ameide test` vs `ameide test e2e` surface mismatch**

- The normative contract (430v2 + 621 + this doc) states:
  - `ameide test` runs Phase 0/1/2 only (local-only).
  - `ameide test e2e` is the explicit Phase 3 front door (Playwright, deployed target truth).
- Current CLI wiring in the `ameide` repo has drifted such that `ameide test` runs Phase 0/1/2/3 by default and
  the `ameide test e2e` subcommand may not exist yet.

Reconciliation requirement:
- Bring the CLI surface back into alignment with this spec (either update the CLI or update the contract docs).

2) **Gateway parentRef must not be hardcoded**

Inner-loop routing (both Phase 3 and `ameide dev`) must attach to the environment platform Gateway listener.
Hardcoding `(gateway name, namespace, sectionName)` is fragile across clusters/namespaces and is known to break when
the Gateway is deployed into the environment namespace (e.g. `ameide-dev`).

Reconciliation requirement:
- Always resolve the platform gateway parentRef by inspecting the baseline platform HTTPRoute in the target
  environment namespace (there is already a resolver helper for this).

3) **Hostname routing increases exposure (security posture change)**

`ameide dev` intentionally swaps header-gated routing (tool-friendly) for per-run hostname routing (human-friendly).
This is a deliberate trade:

- Pros: browsers “just work” (no header injection), human UX is excellent.
- Cons: the devserver becomes reachable by anyone who can guess/obtain the per-run hostname. Since the devserver is
  started with real environment secrets (Auth.js/DB creds), exposure risk must be mitigated at multiple layers.

Mitigations (non-optional):
- Non-guessable run id in hostname (sufficient entropy), short TTL, and cleanup on exit + janitor.
- Admission policy restricting workspace-created HTTPRoutes to safe shapes (host allowlist + parentRef allowlist +
  same-namespace Service backends only).
- NetworkPolicy constraining inbound to `:3001` to Envoy dataplane pods only.

4) **Auth/OIDC must support per-run subdomains**

`ameide dev` depends on Auth.js + OIDC redirect/callback behavior working on per-run subdomains.
If this is not enabled, the correct behavior is fail-fast with explicit diagnostics (not “best effort”).

Design decision:
- This spec chooses **dev-only wildcard redirect URIs** for `*.dev.ameide.io` as the normative requirement.
- If the team prefers “static redirect on canonical host and bounce back”, that should be specified as an explicit
  alternative in this doc and implemented intentionally (not as an accidental workaround).

5) **Che-in-vCluster: host-cluster Gateway + Secrets boundary**

`ameide dev` and (by default) `ameide test e2e` assume the workspace can interact with the **same Kubernetes API**
that owns:

- the environment namespace (for reading `ConfigMap/www-ameide-platform-config` and persona secrets), and
- the Gateway API control plane (for creating `HTTPRoute` that attaches to the platform Gateway).

In a Che-in-vCluster architecture, the workspace commonly sees the **vCluster API** by default. In that case:

- `ameide test` (Phase 0/1/2) remains compatible (no Kubernetes required).
- `ameide test e2e` is compatible only if either:
  - required env ConfigMaps/Secrets are mirrored into the vCluster, OR
  - the CLI is explicitly configured to talk to the host-cluster API (provided kubeconfig/token/context).
- `ameide dev` is compatible only if there is an explicit host-cluster write path for `Service` + `HTTPRoute`
  that the host Gateway reconciles (mirroring Gateway API resources to host, or using host-cluster credentials).

Reconciliation requirement:
- Pick and document the supported Che/vCluster bridging mechanism(s), and make the CLI fail-fast when it is
  pointed at a cluster API that cannot satisfy the required contracts.

## Goals

1. **One thing per command.** No “mode” flags or multi-purpose commands.
2. **No flags.** Commands are stable and agent-friendly. Configuration is cluster-derived (ConfigMaps/Secrets) and/or target selection.
3. **Fail-fast, no fallbacks.** If a command requires cluster wiring (RBAC, NetworkPolicy, Gateway attachment), it fails with explicit diagnostics.
4. **Deterministic 0/1/2.** `ameide test` never requires Kubernetes/Telepresence and must be reproducible offline.
5. **Truth is deployed.** E2E truth uses a deployed base URL resolved from environment ConfigMaps/Secrets in-cluster (no direct base-URL overrides).
6. **Human dev loop works from in-cluster workspaces.** No Telepresence intercepts; no local registries; no GitOps churn per run.

## Non-goals

- Designing preview environments (owned by GitOps architecture work, e.g. `backlog/465-applicationset-architecture-preview-envs-v2.md`).
- Supporting Telepresence or any traffic interception requiring elevated container privileges.
- Supporting local-registry-based workflows (explicitly disallowed).

## Cross references

- Normative phase split and `ameide test e2e`: `backlog/430-unified-test-infrastructure-v2-target.md`
- Inner-loop front doors: `backlog/621-ameide-cli-inner-loop-test-v2.md`
- Agentic workspace posture (Coder/Che): `backlog/650-agentic-coding-overview.md`, `backlog/652-agentic-coding-dev-workspace.md`, `backlog/690-agentic-dev-environments-coder-che.md`
- Historical “Tilt/Telepresence” loop (context only): `backlog/432-devcontainer-modes-offline-online.md`, `backlog/435-remote-first-development.md`

---

## CLI surface (normative)

### `ameide test`

Runs **Phase 0/1/2 only**:

0) Phase 0: contract (discovery/collect/list; emits JUnit evidence, synthetic if needed)  
1) Phase 1: unit (local-only, deterministic)  
2) Phase 2: integration (local-only, mocked/stubbed only; no Kubernetes/Telepresence)

**Invariants**

- MUST NOT require Kubernetes configuration, cluster access, Telepresence, or environment secrets.
- MUST fail-fast on missing toolchain or misconfigured runners (with synthetic JUnit evidence).
- MUST emit JUnit evidence for each phase under a deterministic run root.

**Config inputs**

- Optional env/config for repo-local behavior (paths, timeouts), but no flags.
- MUST NOT load `.env` best-effort; `.env` is not part of the contract.

**Outputs**

- A run root (e.g. `artifacts/agent-ci/<timestamp>/`) containing:
  - per-phase logs
  - per-phase JUnit output
  - optional additional artifacts (lint output, reports) as defined by the phase runner

### `ameide test e2e`

Runs **Playwright E2E only** (“Phase 3” of the overall verification story), against a **deployed target**:

- No mocks.
- Canonical truth for merge gates comes from these runs (preview env truth).

**Invariants**

- MUST target a deployed base URL (preview env ingress, or a real environment ingress).
- MUST NOT depend on workspace routing tricks (no header-gated innerloop routing; no workspace devserver).
- MUST be runnable from Coder/Che workspaces; MAY be runnable from CI runners if they can reach the base URL and obtain required secrets.

**Base URL resolution (fail-fast, no fallbacks)**

`ameide test e2e` MUST resolve its base URL from the deployed environment configuration in-cluster:

1) `AMEIDE_ENV_NAMESPACE` MUST be set (the target environment namespace).
2) The runner MUST read `ConfigMap/www-ameide-platform-config` in `AMEIDE_ENV_NAMESPACE`.
3) The runner MUST use `AUTH_URL` from that ConfigMap as the Playwright base URL.

If any of these are missing or invalid, the command MUST fail fast with an explicit error.

**Secrets**

- Persona and Auth.js secrets MUST be provided via Secret in the target environment namespace (e.g. `Secret/playwright-int-tests-secrets`).
- The runner MUST treat credentials as secrets and MUST avoid printing them in logs/artifacts.

**Outputs**

- JUnit output (mandatory)
- Optional Playwright HTML report
- Optional traces/videos/screenshots (policy-controlled; see secrecy guidance)

## `ameide dev` (human UI inner loop, Telepresence-free)

### Summary

`ameide dev` provides a human-friendly inner loop for `www_ameide_platform` from an in-cluster workspace (Coder/Che):

1) Starts the `www_ameide_platform` dev server in the workspace pod (hot reload).
2) Exposes it via a per-run Kubernetes `Service` in the workspace namespace.
3) Creates a per-run `HTTPRoute` that attaches to the environment Gateway and routes a **per-run hostname** to the workspace Service.
4) Prints the dev URL.
5) On exit, cleans up created resources.

This command is intentionally separate from `ameide test e2e`:

- `ameide dev` is for **human browsing** and rapid UI iteration.
- `ameide test e2e` is for **deployed-system truth** and merge gating.

### Command: `ameide dev`

**Invariants**

- MUST run inside a Kubernetes pod (Coder/Che workspace).
- MUST NOT depend on Telepresence.
- MUST NOT require GitOps changes per run (only creates ephemeral namespaced resources).
- MUST fail-fast if the Gateway attachment contract is not satisfied (allowedRoutes, RBAC, NetworkPolicy).

**Dev server**

- Runs: `pnpm -C services/www_ameide_platform dev`
- MUST bind: `0.0.0.0:3001`
- MUST confirm readiness (e.g. `http://127.0.0.1:3001/api/auth/providers`).

**Workspace hostname (human-friendly)**

- Format: `platform-dev-<workspace-env-id>.dev.ameide.io`
- `<workspace-env-id>` MUST be DNS-1123-safe (lowercase `[a-z0-9-]`) and MUST be derived deterministically from the workspace identity.
  - Coder: derive from the workspace namespace name (e.g. `ameide-ws-<uuid>` → `<uuid>`), or another stable workspace identifier exposed in the pod environment.
  - Che: derive from the DevWorkspace identity (or namespace) using the same DNS-safe normalization.
- The hostname is “human-friendly” because no custom headers are required.
- `run-id` remains an internal concept for labeling/TTL and diagnostics; it MUST NOT be embedded into the dev hostname.

**Auth strategy (normative choice)**

To make per-run subdomains behave correctly with OIDC/Auth.js:

- Keycloak MUST allow redirect URIs for `https://*.dev.ameide.io/*` (dev-only).
- Auth.js cookie domain MUST be compatible with subdomains (typically `.dev.ameide.io`), and callback URL rules MUST be validated.

If these conditions are not met, `ameide dev` MUST fail-fast with an explicit error stating that wildcard subdomain auth is not enabled.

**Gateway attachment**

- MUST attach to the same Gateway listener as the canonical platform route (HTTPS listener).
- MUST NOT hardcode `parentRefs`; it MUST resolve the platform parentRef using the existing resolver (`ResolvePlatformGatewayParentRef`) by reading the baseline platform route and deriving the correct `(gateway name, namespace, sectionName)`.

**Routing restrictions (safety)**

Workspace-created `HTTPRoute` resources for `ameide dev` MUST be restricted by admission policy (Kyverno/CEL) to:

- Hostname pattern: `platform-dev-*.dev.ameide.io` (where `*` is the workspace environment id)
- ParentRef: resolved platform Gateway only (name/namespace/sectionName)
- BackendRef: Service in the same namespace only
- Required TTL labels/annotations (see below)

**Per-run Kubernetes resources**

In the workspace namespace, `ameide dev` creates:

- `Service` (ClusterIP) selecting the workspace pod on `port:3001`
- `HTTPRoute` routing the per-run hostname to that Service

All created resources MUST include:

Labels:
- `ameide.devx/owner: ameide-cli`
- `ameide.devx/run-id: <run-id>`
- `ameide.devx/command: dev`

Annotations:
- `ameide.devx/created-at-epoch: "<unix-seconds>"`
- `ameide.devx/expires-at-epoch: "<unix-seconds>"`

**Cleanup**

- MUST delete the per-run `HTTPRoute` and `Service` on exit (success or failure).
- MUST stop the dev server if it was started by the CLI.

### Command: `ameide dev down` (optional, recommended)

Deletes leftover dev resources in the current workspace namespace:

- Deletes `HTTPRoute` and `Service` with `ameide.devx/owner=ameide-cli` and `ameide.devx/command=dev`
- Deletes only resources that are:
  - expired by `ameide.devx/expires-at-epoch`, OR
  - explicitly opted into deletion by label selector

This is a user convenience. A cluster-side janitor is still required as the safety net.

---

## Shared configuration & environment variables (no flags)

### Required (cluster-only commands)

- `AMEIDE_ENV_NAMESPACE` (required by `ameide test e2e` and `ameide dev`)

### Derived values (no hardcoding)

Commands running inside a workspace SHOULD derive:

- Workspace namespace from in-cluster serviceaccount `namespace` file.
- Workspace pod name from `HOSTNAME`.

If derivation fails, commands MUST error with a clear message; no “best-effort guessing”.

---

## Cluster enablement prerequisites (Phase 3 / dev)

These are prerequisites for `ameide test e2e` and `ameide dev`:

1) Workspace namespaces MUST be allowed to attach `HTTPRoute` to the platform Gateway (via `allowedRoutes` selector).
2) Workspace service accounts MUST have RBAC to:
   - create/delete `Service` in workspace namespace
   - create/delete `HTTPRoute` in workspace namespace
   - read `HTTPRoute` status
   - read required env ConfigMaps/Secrets in `AMEIDE_ENV_NAMESPACE`
3) NetworkPolicy MUST allow Envoy dataplane pods to reach workspace pod `:3001` for devserver routing.
4) Admission policy MUST constrain workspace-created Routes to safe shapes (host patterns, parentRef allowlist, same-namespace backends).
5) A cluster-side janitor MUST delete leaked `ameide.devx/owner=ameide-cli` resources past TTL.

## GitOps enablement status (tracking notes)

This section links existing GitOps enablement that already supports parts of Phase 3 / `ameide dev`.
It is tracking context only; the normative requirements are the sections above.

- Workspace RBAC bootstrap (Coder):
  - `sources/values/_shared/cluster/coder-workspace-phase3-rbac.yaml`
  - Provides: create/delete Services + HTTPRoutes in workspace namespaces; read env ConfigMaps/Secrets in `ameide-dev`.
- Workspace ingress NetworkPolicy automation (Kyverno generate policy):
  - `policies/generate-workspace-innerloop-devserver-networkpolicy.yaml`
  - Provides: Envoy dataplane reachability to workspace devserver port `3001`.

Remaining gaps (must be implemented to meet the spec):
- Admission policy restricting workspace-created HTTPRoutes (host pattern, parentRef allowlist, backendRefs).
- Cluster-side janitor for leaked innerloop resources (HTTPRoute/Service) based on TTL annotations/labels.
- Che-in-vCluster bridging: document and implement the supported host-cluster read/write path for `ameide test e2e` and `ameide dev`.

---

## Evidence and secrecy

1. Commands that run tests (`ameide test`, `ameide test e2e`) MUST emit JUnit evidence.
2. Credentials and capability tokens MUST NOT be written to:
   - logs
   - Playwright traces/HAR
   - persisted artifacts
3. `ameide dev` MUST avoid printing secrets. It MAY print the per-run hostname URL, since the URL is the intended human entrypoint.

---

## Rollout / migration notes

- Implementing this surface will likely deprecate older “Phase 3 inside ameide test” language and any `--inner-loop` flags/modes described historically (see `backlog/621-ameide-cli-inner-loop-test.md`).
- This surface is intentionally compatible with both Coder and Che workspaces. “In-cluster workspace” is the portability boundary.
