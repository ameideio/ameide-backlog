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

- Local-only verification (contract → unit → integration).
- Deployed-system truth (Playwright E2E).
- A human-friendly UI dev loop in an in-cluster workspace (Coder/Che) without Telepresence.

This document is a **CLI surface spec** (user-facing contract + invariants). Implementation is tracked separately.

## Current implementation mismatches / risks (must be reconciled)

This section records known mismatches between the normative surface described below and the current implementation
in `ameideio/ameide` (and cluster enablement in `ameideio/ameide-gitops`).

1) **`ameide test` vs `ameide test e2e` surface mismatch**

- The normative contract (430v2 + 621 + this doc) states:
  - `ameide test` runs contract/unit/integration only (local-only).
  - `ameide test e2e` is the explicit E2E front door (Playwright, deployed target truth).
- Current CLI wiring in the `ameide` repo has drifted such that `ameide test` runs Phase 0/1/2/3 by default and
  the `ameide test e2e` subcommand may not exist yet.

Reconciliation requirement:
- Bring the CLI surface back into alignment with this spec (either update the CLI or update the contract docs).

2) **Gateway parentRef must not be hardcoded**

Inner-loop dev routing (`ameide dev` and any legacy/innerloop routing helpers) must attach to the environment platform Gateway listener.
Hardcoding `(gateway name, namespace, sectionName)` is fragile across clusters/namespaces and is known to break when
the Gateway is deployed into the environment namespace (e.g. `ameide-dev`).

Reconciliation requirement:
- Always resolve the platform gateway parentRef by inspecting the baseline platform HTTPRoute in the target
  environment namespace (there is already a resolver helper for this).

Additional invariant (Gateway hostname intersection):
- The target Gateway listener hostname MUST intersect with `platform-dev-<workspace-env-id>.dev.ameide.io`.
  If the listener is restricted to `platform.dev.ameide.io` only (exact hostname), `ameide dev` routes will never match.
  Preferred listener hostname in dev is `*.dev.ameide.io` (or another pattern that covers `platform-dev-*.dev.ameide.io`).

Additional invariants (DNS + TLS coverage):
- DNS MUST resolve `platform-dev-<workspace-env-id>.dev.ameide.io` to the platform Gateway address (typically via a wildcard `*.dev.ameide.io` record).
- The Gateway listener TLS certificate MUST be valid for `platform-dev-<workspace-env-id>.dev.ameide.io` (typically via `*.dev.ameide.io` SAN coverage).

3) **Hostname routing increases exposure (security posture change)**

`ameide dev` intentionally swaps header-gated routing (tool-friendly) for per-run hostname routing (human-friendly).
This is a deliberate trade:

- Pros: browsers “just work” (no header injection), human UX is excellent.
- Cons: the devserver becomes reachable by anyone who can guess/obtain the per-run hostname. Since the devserver is
  started with real environment secrets (Auth.js/DB creds), exposure risk must be mitigated at multiple layers.

Mitigations (non-optional):
- Short TTL and cleanup on exit + janitor.
- Admission policy restricting workspace-created HTTPRoutes to safe shapes (host allowlist + parentRef allowlist +
  same-namespace Service backends only).
- NetworkPolicy constraining inbound to `:3001` to Envoy dataplane pods only.

4) **Auth/OIDC must support per-run subdomains**

`ameide dev` depends on Auth.js + OIDC redirect/callback behavior working on per-run subdomains.
If this is not enabled, the correct behavior is fail-fast with explicit diagnostics (not “best effort”).

Design decision:
- This spec chooses a **static redirect URI on the canonical host + bounce back** as the normative requirement.
- Rationale: wildcard subdomain redirect URIs are not a safe assumption across IdP versions/policies; treating them as
  “required” tends to create long-lived operational/security friction.
- This is also aligned with common IdP hardening guidance: do not rely on broad redirect URI wildcards for public-facing clients.
- If the team later decides to support wildcard subdomain redirects in dev, it MUST be treated as an explicit,
  dev-only, version-pinned capability with a clear rollback path.

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

Supported modes (must be made explicit in implementation + docs):
- **Mode A (sync/mirror):**
  - host ➜ vCluster: mirror required env ConfigMaps/Secrets (read-only mirror for `ameide test e2e`).
  - vCluster ➜ host: sync workspace `Service` + `HTTPRoute` into the host cluster so the host Gateway reconciles them (required by `ameide dev`; implies enabling sync and RBAC for those resource types).
- **Mode B (host-cluster creds):** provide host-cluster kubeconfig/token/context to the workspace; cluster-only commands MUST use it.

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
- Implementation plan: `backlog/692-cli-surface-implementation-plan.md`
- Agentic workspace posture (Coder/Che): `backlog/650-agentic-coding-overview.md`, `backlog/652-agentic-coding-dev-workspace.md`, `backlog/690-agentic-dev-environments-coder-che.md`
- Historical “Tilt/Telepresence” loop (context only): `backlog/432-devcontainer-modes-offline-online.md`, `backlog/435-remote-first-development.md`

## Alignment checklist (follow-up edits required)

This section records the docs that should be updated so the “diagram matches the code” once 691 is implemented.

- `backlog/654-agentic-coding-cli-surface.md`: promote “optional interactive checks” into the concrete `ameide dev` command, and clarify that `ameide dev` (not `ameide test e2e`) owns workspace routing.
- `backlog/653-agentic-coding-test-automation.md`: ensure the automation layer treats `ameide test e2e` as the deployed truth gate and `ameide dev` as the human hotreload loop.
- `backlog/621-ameide-cli-inner-loop-test-v2.md`: remains the normative source for `ameide test smoke`. This doc intentionally does not define `smoke` even though the filename still includes it historically.
- `backlog/441-networking.md`: cross-link the workspace dev routing contracts (allowedRoutes + Envoy→workspace NetworkPolicy + hostnames) to the general Gateway/DNS primitives.
- `backlog/427-platform-login-stability.md`: add an explicit note about the canonical-host auth bounce + cookie-domain requirements used by `ameide dev`, since it intersects with existing redirect_uri / cookie-domain failure modes.
- `backlog/686-aks-pool-principles-migration-plan.md`: treat pool layout/pinning as an indirect dependency for stable gateway/workspace assumptions (Envoy placement, workspace pool isolation).

---

## CLI surface (normative)

### `ameide test`

Runs contract/unit/integration only (local-only):

- Contract: discovery/collect/list; emits JUnit evidence, synthetic if needed
- Unit: local-only, deterministic
- Integration: local-only, mocked/stubbed only; no Kubernetes/Telepresence

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

Runs Playwright E2E only (deployed-system truth), against a **deployed target**:

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
  - Che: derive from the DevWorkspace identity (or namespace) using the same DNS-safe normalization. If a stable id beyond namespace is required, it MAY be read from the DevWorkspace resource (requires RBAC).
- The hostname is “human-friendly” because no custom headers are required.
- `run-id` remains an internal concept for labeling/TTL and diagnostics; it MUST NOT be embedded into the dev hostname.

**Auth strategy (normative choice)**

To make per-run subdomains behave correctly with OIDC/Auth.js without relying on IdP wildcard redirect URIs:

- Keycloak redirect URI MUST remain static on the canonical platform host (e.g. `https://platform.dev.ameide.io/...`).
- The callback handler MUST “bounce back” to the dev hostname using a validated `return_to`/`state` value.
  - The allowlist MUST be restricted to `https://platform-dev-*.dev.ameide.io/*` (and ideally tightened further).
- The implementation SHOULD use Auth.js redirect proxy support (`redirectProxyUrl` / `AUTH_REDIRECT_PROXY_URL`) rather than inventing a bespoke mechanism.
- Auth.js MUST trust proxy headers for correct callback URL construction behind the Gateway (e.g. `AUTH_TRUST_HOST=true` / equivalent).
- If using Auth.js redirect proxy, both the canonical host deployment and the workspace devserver MUST share the same `AUTH_SECRET`, and redirect proxy MUST be enabled by setting `AUTH_REDIRECT_PROXY_URL` appropriately.
- Auth.js cookie domain MUST be compatible with subdomains (typically `Domain=.dev.ameide.io`), and callback URL rules MUST be explicitly validated.
- Cookies MUST NOT use the `__Host-` prefix if a `Domain=` attribute is required for subdomain sharing.
- If subdomain session sharing is required, custom cookie options may be required (Domain + naming/prefix policy).

If these conditions are not met, `ameide dev` MUST fail-fast with an explicit error stating that the canonical-host bounce configuration is not enabled.

**Gateway attachment**

- MUST attach to the same Gateway listener as the canonical platform route (HTTPS listener).
- MUST NOT hardcode `parentRefs`; it MUST resolve the platform parentRef using the existing resolver (`ResolvePlatformGatewayParentRef`) by reading the baseline platform route and deriving the correct `(gateway name, namespace, sectionName)`.
- MUST fail fast based on the created `HTTPRoute` status:
  - Wait for `status.parents[*].conditions` for the resolved parentRef to include `Accepted=True` and `ResolvedRefs=True`.
  - On failure or timeout, print the relevant condition `message`/`reason` values as the primary diagnostic (do not guess).
  - If a `Programmed` or `Ready` condition is present, it SHOULD be `True` before printing the dev URL; otherwise keep waiting (or warn and continue only after the timeout policy is exhausted).

**Routing restrictions (safety)**

Workspace-created `HTTPRoute` resources for `ameide dev` MUST be restricted by admission policy (Kyverno/CEL) to:

- Hostnames MUST be concrete and MUST match `^platform-dev-[a-z0-9-]+\\.dev\\.ameide\\.io$` (wildcards are not used in `HTTPRoute.spec.hostnames`)
- ParentRef: resolved platform Gateway only (name/namespace/sectionName)
- BackendRef: Service in the same namespace only (deliberate: avoid cross-namespace backend refs and ReferenceGrant complexity)
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
- Deletes only resources that are expired by `ameide.devx/expires-at-epoch`.

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

These are prerequisites for cluster-only commands. They are intentionally split to avoid conflating “deployed truth E2E” with “workspace dev routing”.

### Prereqs: `ameide test e2e` (deployed truth; no workspace routing)

1) `AMEIDE_ENV_NAMESPACE` MUST be reachable and readable by the workspace identity.
2) Workspace identity MUST have RBAC to read:
   - `ConfigMap/www-ameide-platform-config` in `AMEIDE_ENV_NAMESPACE`
   - `Secret/playwright-int-tests-secrets` (and any required Auth.js secrets) in `AMEIDE_ENV_NAMESPACE`

### Prereqs: `ameide dev` (workspace dev routing)

1) Workspace namespaces MUST be allowed to attach `HTTPRoute` to the platform Gateway (via `allowedRoutes` selector).
2) Gateway listener hostname MUST cover `platform-dev-*.dev.ameide.io` (hostname intersection is required for routing).
3) Workspace service accounts MUST have RBAC to:
   - create/delete `Service` in workspace namespace
   - create/delete `HTTPRoute` in workspace namespace
   - read `HTTPRoute` status
   - read required env ConfigMaps/Secrets in `AMEIDE_ENV_NAMESPACE`
4) NetworkPolicy MUST allow Envoy dataplane pods to reach workspace pod `:3001` for devserver routing.
5) Admission policy MUST constrain workspace-created Routes to safe shapes (concrete hostnames matching the allowlist regex, parentRef allowlist, same-namespace backends).
6) A cluster-side janitor MUST delete leaked `ameide.devx/owner=ameide-cli` resources past TTL.

## GitOps enablement status (tracking notes)

This section links existing GitOps enablement that already supports parts of Phase 3 / `ameide dev`.
It is tracking context only; the normative requirements are the sections above.

- Workspace RBAC bootstrap (Coder):
  - `sources/values/_shared/cluster/coder-workspace-phase3-rbac.yaml`
  - Provides: create/delete Services + HTTPRoutes in workspace namespaces; read env ConfigMaps/Secrets in `ameide-dev` (for cluster-only commands).
- Workspace ingress NetworkPolicy automation (Coder workspaces):
  - `sources/values/_shared/cluster/coder-workspace-phase3-rbac.yaml`
  - Provides: Envoy dataplane reachability to workspace devserver port `3001` (required by `ameide dev`).
- Workspace innerloop HTTPRoute admission policy (ValidatingAdmissionPolicy, static allowlist):
  - `sources/values/_shared/cluster/workspace-innerloop-routing-policy.yaml`
  - Constrains workspace-created `HTTPRoute` to safe shapes (hostname allowlist regex, parentRef allowlist, same-namespace backendRef, TTL labels/annotations).
- Workspace innerloop janitor:
  - `sources/values/_shared/cluster/workspace-innerloop-janitor.yaml`
  - Deletes expired leaked `HTTPRoute`/`Service` resources created by `ameide dev`.
- Che-in-vCluster bridging (dev):
  - `sources/values/env/dev/platform/platform-vcluster-che.yaml`
  - Enables syncing `HTTPRoute` from vCluster → host, and mirrors required env ConfigMaps/Secrets from host → vCluster.

Remaining gaps (must be implemented to meet the spec):
- Decide whether parentRef enforcement should be a static allowlist (ValidatingAdmissionPolicy) or dynamically derived (Kyverno external data / published parentRef).

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
