---
title: "670 – Gateway-based inner loop for Playwright (agent-friendly traffic filtering)"
status: draft
owners:
  - platform-devx
  - test-infra
created: 2026-01-17
suite: "650-agentic-coding"
source_of_truth: false
---

# 670 – Gateway-based inner loop for Playwright (agent-friendly traffic filtering)

## 0) Purpose

Provide an **end-to-end, Telepresence-free** inner loop that allows humans or coding agents to run Playwright against local `www` UI changes **without** waiting for preview environments or touching GitOps, and without affecting other users.

This replaces the historical “Tilt + Telepresence intercept” loop (see `backlog/432-devcontainer-modes-offline-online.md`, `backlog/435-remote-first-development.md`, `backlog/581-parallel-ai-devcontainers.md`, `backlog/621-ameide-cli-inner-loop-test-v2.md`) with a Gateway API routing model that works inside the current **Coder-in-cluster** posture (`backlog/650-agentic-coding-overview.md`, `backlog/652-agentic-coding-dev-workspace.md`, `backlog/653-agentic-coding-test-automation.md`).

## 1) Scope

### 1.1 In scope

* A short inner loop where one command (`ameide test`) can:
  * Run the `services/www_ameide_platform` Next.js dev server with hot reload **inside the workspace pod**.
  * Expose the dev server via a Kubernetes `Service` reachable inside the cluster.
  * Configure Gateway API routing rules that divert only the caller’s traffic (header match) to that dev server.
  * Run Playwright against the **same HTTPS origin** as deployed environments (e.g. `https://platform.dev.ameide.io`), preserving Keycloak/Auth.js origin semantics.
* Parallel agent support (multiple developers/agents simultaneously using the routing scheme).
* Cleanup guarantees (routes and services removed after the run).
* Evidence emission compatible with Phase 3 artifacts (JUnit + logs; inner-loop disables trace/video due to session secrecy).

### 1.2 Out of scope

* Preview environments / PR images (owned by `backlog/465-applicationset-architecture-preview-envs-v2.md`).
* Telepresence-based intercepts (explicitly unsupported per `backlog/650-agentic-coding-overview.md` §1.2).
* Per-run GitOps changes (inner-loop uses ephemeral, per-run resources in workspace namespaces).

## 2) Definitions

* **Baseline origin**: The deployed HTTPS origin of the target environment (e.g. `https://platform.dev.ameide.io`), discovered the same way as Phase 3 and treated as the canonical browser-visible origin.
* **Inner-loop run**: A run that routes a subset of baseline-origin traffic to an agent’s dev server (UI-only override) while preserving browser-visible origin semantics.
* **Agent identity**: `AMEIDE_AGENT_ID` (routing key).
* **Agent header**: `X-Ameide-Agent: <AMEIDE_AGENT_ID>`.
* **Session header**: `X-Ameide-Session: <AMEIDE_E2E_SESSION>` where `<...>` is random per run (≥128-bit, URL-safe).
* **Run ID**: a non-secret identifier used for resource names + TTL labels (`ameide.devx/run-id`).
* **Gateway**: the platform’s Gateway API `Gateway` resource serving the baseline origin.
* **Dev server**: a Next.js `next dev` process for `services/www_ameide_platform` bound to `0.0.0.0:3001`.

## 3) Requirements (normative)

### 3.1 Core requirements

1. **No Telepresence dependency.** Inner-loop MUST NOT require Telepresence.
2. **Gateway-based routing.** Inner-loop MUST use Gateway API `HTTPRoute` attached to the existing platform `Gateway`.
3. **Origin correctness.** Playwright MUST target the baseline HTTPS origin (e.g. `https://platform.dev.ameide.io`), not `localhost`.
4. **Traffic isolation.** Only requests matching both `X-Ameide-Agent` and `X-Ameide-Session` MUST be routed to the dev server.
5. **Path allowlist only.** Only these prefixes are eligible for override: `/_next/`, `/api/auth/`, `/`.
6. **Parallel-safe.** Session header uniqueness MUST prevent collisions across concurrent runs.
7. **Cleanup.** The CLI MUST delete created `Service` and `HTTPRoute` on success and failure.
8. **Session secrecy.** `X-Ameide-Session` MUST be treated as a capability token and MUST be kept out of:
   * CLI logs
   * Playwright traces/video/HAR (inner-loop disables trace/video)
   * persisted artifacts (best-effort redaction for text artifacts)
9. **TTL enforcement.** All per-run resources MUST be time-bounded and eligible for janitor cleanup.

### 3.2 Routing contract

**Hostname**
* Shared hostname mode only.
* Allowed hostnames: `platform.<env>.ameide.io` (no wildcards).

**Headers**
* Both headers MUST be required on every match:
  * `X-Ameide-Agent: <AMEIDE_AGENT_ID>`
  * `X-Ameide-Session: <AMEIDE_E2E_SESSION>`
* Header match MUST be exact equality.

**Paths**
Only these path prefixes may be overridden by inner-loop routes:
* `/_next/`
* `/api/auth/`
* `/`

### 3.3 Proof-of-routing preflight (required)

The CLI MUST prove that tagged requests are actually hitting the workspace dev server (not the baseline backend) before running Playwright.

Implementation contract:

* The dev server MUST serve `GET /__ameide_innerloop__/probe` and:
  * return `404` unless `AMEIDE_INNERLOOP_RUN_ID` is set, and
  * return `200` JSON containing that run id when it is set.
* The CLI MUST request `https://platform.<env>.ameide.io/__ameide_innerloop__/probe` with both routing headers and validate the returned run id matches its current run id.

This avoids relying on Envoy Gateway filter support for response marker headers.

## 4) Kubernetes resources (per run)

All per-run resources are created in the **workspace namespace** and attach cross-namespace to the platform `Gateway` via `parentRefs`.

### 4.1 Required labels/annotations (TTL cleanup)

Labels:
* `ameide.devx/owner: ameide-cli`
* `ameide.devx/agent-id: <AMEIDE_AGENT_ID>`
* `ameide.devx/run-id: <run-id>` (non-secret)

Annotations:
* `ameide.devx/created-at-epoch: "<unix-seconds>"`
* `ameide.devx/expires-at-epoch: "<unix-seconds>"`

### 4.2 Per-run Service

* Name: `ameide-innerloop-www-<run-id>`
* Type: `ClusterIP`
* Selector: MUST target exactly one workspace pod.
* Port: `3001` → `targetPort: 3001`

### 4.3 Per-run HTTPRoute

* Name: `ameide-innerloop-www-route-<run-id>`
* Namespace: workspace namespace
* `parentRefs`: attach to the platform HTTPS listener (section `https`) of the real platform `Gateway`.
* `hostnames`: `[platform.<env>.ameide.io]`
* `rules`: three rules with `PathPrefix` matches for `/_next/`, `/api/auth/`, `/`, each requiring both headers and forwarding to the per-run Service port 3001.

### 4.4 Status readiness gate

The CLI MUST wait for the `HTTPRoute` to be ready before running Playwright. Minimum required checks:

* For the status parent corresponding to the route’s `parentRef`:
  * `Accepted=True`
  * `ResolvedRefs=True`

## 5) CLI contract (`ameide test` / `ameide test ci`)

One-command flow (agent-friendly, no human intervention):

1. Determine the target environment namespace from `AMEIDE_AKS_NAMESPACE` (provisioned by the workspace template; explicit override allowed).
2. Read the platform runtime configuration for `www-ameide-platform` from:
   * `ConfigMap/www-ameide-platform-config` (in the environment namespace)
   * `Secret/www-ameide-platform-auth` (in the environment namespace)
   * `Secret/www-ameide-platform-db` (in the environment namespace)
3. Generate:
   * `AMEIDE_AGENT_ID` (stable identifier)
   * `AMEIDE_E2E_SESSION` (random secret capability token)
   * `run-id` (non-secret)
4. Start dev server:
   * `pnpm -C services/www_ameide_platform dev` on `0.0.0.0:3001`
   * with `AMEIDE_INNERLOOP_RUN_ID=<run-id>`
5. Create per-run `Service` + `HTTPRoute` (TTL-labeled) in the workspace namespace.
6. Wait for `HTTPRoute` readiness.
7. Proof-of-routing preflight: fetch `GET /__ameide_innerloop__/probe` through the real origin with both headers and validate run-id.
8. Run Playwright against the real origin while injecting the two headers only for requests to the platform host.
9. Cleanup ALWAYS deletes the route + service.

Evidence:
* JUnit output is always produced.
* Inner-loop disables Playwright trace/video, and CLI performs best-effort redaction of the session token from text artifacts.

## 6) Cluster enablement (GitOps-owned)

Inner-loop requires cluster-side enablement (owned by `ameide-gitops`):

* Workspace namespaces labeled:
  * `gateway-access=allowed` (to satisfy platform Gateway `allowedRoutes` selector)
  * `ameide.devx/workspace=true` (so admission policies can target only workspace namespaces)
* Workspace service accounts RBAC:
  * CRUD on `services`
  * CRUD on `httproutes.gateway.networking.k8s.io`
  * get/list/watch on `httproutes` for status checks
  * get on `ConfigMap/www-ameide-platform-config` and `Secret/www-ameide-platform-auth`, `Secret/www-ameide-platform-db`, `Secret/playwright-int-tests-secrets` in the environment namespace (bound per-workspace via RoleBinding)
* NetworkPolicy permitting Envoy Gateway dataplane pods to reach workspace pods on TCP/3001.
* Admission control restricting workspace-created `HTTPRoute` objects to the inner-loop contract.
* A janitor CronJob to delete expired per-run `Service` + `HTTPRoute` resources.

## 7) Parallelism and outside access

### 7.1 Parallel runs

Parallel runs are safe because:
* `X-Ameide-Session` is unique per run and required for routing.
* `run-id` makes Kubernetes resource names unique per run.

### 7.2 External reachability

The dev override is externally reachable only to callers who can supply both headers. `X-Ameide-Session` is non-guessable (≥128-bit) and MUST be kept out of logs and artifacts.

## 8) Vendor alignment notes (summary)

* Gateway API supports shared-host routing with hostname + path + header matches; precedence is defined by match specificity (controllers must conform).
* Envoy Gateway supports HTTPRoute header/path routing and listener binding via `parentRefs[].sectionName`.
* Playwright `extraHTTPHeaders` are broad; this design injects headers selectively via request routing logic and disables trace/video in inner-loop mode to protect the session token.
* Envoy access logs do not include arbitrary headers unless configured to do so; platform telemetry/log config MUST avoid logging `X-Ameide-Session`.
* ValidatingAdmissionPolicy (CEL) enforcement requires cluster support (Kubernetes 1.30+ GA); if a cluster cannot support it, a Kyverno equivalent is the fallback enforcement mechanism.
