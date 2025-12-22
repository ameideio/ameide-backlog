# 589: Deterministic RPC Transports (Connect + gRPC Target Architecture)

**Status:** In progress  
**Created:** 2025-12-22  
**Owner:** Platform Architecture / DX  
**Related:** `backlog/300-400/306-rpc-runtime-alignment.md`, `backlog/300-400/393-ameide-sdk-import-policy.md`, `backlog/428-onboarding.md`, `backlog/300-400/329-authz.md`, `backlog/300-400/333-realms.md`, `backlog/582-local-dev-seeding.md`, `backlog/540-sales-domain.md`, `backlog/527-transformation-agent.md`

---

## Executive Summary

We standardize RPC in Ameide so that **clients and gateways are deterministic** and protocol boundaries are explicit:

- **Internal canonical protocol:** gRPC over HTTP/2.
- **Browser access:** via a BFF (Next.js) using Connect, not direct-to-gRPC.
- **No runtime auto-detection** in SDKs. Environment guessing creates non-deterministic failures in modern ESM/Next.js runtimes.

This backlog item captures the end-state architecture and the required repo + GitOps changes to land it cleanly (no compatibility shims).

---

## Relationship to other backlogs (scope)

This item is intentionally a **runtime + GitOps “make it real” slice** across existing architectural backlogs. It is mostly additive and should avoid conflicts.

- `backlog/300-400/306-rpc-runtime-alignment.md`: umbrella for selecting runtimes, migrating services, and adding guardrails; this item is the concrete determinism + gateway/routing part.
- `backlog/300-400/393-ameide-sdk-import-policy.md`: defines deterministic import surfaces (`/node` vs `/browser`) and bans runtime auto-detection; this item ensures GitOps and runtimes actually satisfy those constraints.
- `backlog/428-onboarding.md`: names the symptom (`ConnectError: [unimplemented] HTTP 404`) and ties it to routing/protocol mismatch; this item is a P0 unblocker for onboarding stability.
- `backlog/300-400/329-authz.md` + `backlog/300-400/333-realms.md`: orthogonal in purpose (authz + tenancy model), but depend on a deterministic RPC boundary so failures are “real” (not misrouting).
- `backlog/582-local-dev-seeding.md`: same philosophy (“deterministic envs”), different layer (data vs transport/routing); they reinforce each other.
- Domain slices (`backlog/540-sales-domain.md`, `backlog/527-transformation-agent.md`): mostly separate; when they expose or consume RPC, they must comply with this item’s deterministic transport + gateway boundary rules.

---

## 1) Problem Statement

The platform currently exhibits intermittent and confusing RPC failures (example symptom):

- `ConnectError: [unimplemented] HTTP 404` when calling `ameide_core_proto.platform.v1.OrganizationService/*`

In Connect-ES, a plain HTTP 404 is commonly surfaced as `unimplemented`. In practice, we observed the 404 happen when:

- a server-side Node runtime (Next.js / ESM) used an SDK that **guessed** transports at runtime
- the SDK fell back to a Connect/HTTP transport in an environment where it should have used gRPC/HTTP2
- the request hit a gateway listener configured for **gRPC routing only** (`GRPCRoute`)
- the route did not match at the protocol layer → HTTP 404 → `unimplemented`

This is not a “method missing from proto” issue; it is a **protocol/transport determinism** issue.

---

## 2) Target Architecture (No Shims)

### 2.1 Protocol boundaries

**Browser → BFF**
- Browser uses **Connect protocol** to the Next.js BFF (HTTP/1.1 or HTTP/2 behind the scenes is fine).
- The browser does not require direct network access to internal gRPC endpoints.

**BFF / Services → Gateway → Services**
- Server-side calls use **gRPC protocol over HTTP/2**.
- Gateway routes gRPC via `GRPCRoute` (service/method matching).

### 2.2 SDK import surfaces (deterministic)

TypeScript SDK MUST NOT guess runtime.

- Node/server code imports `@ameideio/ameide-sdk-ts/node` (defaults to **gRPC** transport)
- Browser code imports `@ameideio/ameide-sdk-ts/browser` (defaults to **Connect** transport)
- Core `@ameideio/ameide-sdk-ts` root export remains types/helpers and requires explicit `transport` / `transportFactory` when constructing clients.

This aligns with `backlog/300-400/393-ameide-sdk-import-policy.md` (single supported SDK surface; no hidden runtime behaviour).

### 2.3 Service runtime alignment

TypeScript services standardize on Connect runtime (`@connectrpc/connect-node`) and serve RPCs on an **HTTP/2** listener.

This is consistent with the direction in `backlog/300-400/306-rpc-runtime-alignment.md` and existing service patterns (`threads`, `chat`, `workflows`).

---

## 3) GitOps Requirements (Gateway + Env + Routing)

> Source of truth: `ameideio/ameide-gitops` (this repo vendors it as the `gitops/ameide-gitops/` submodule).

### 3.0 Which GitOps files are authoritative?

The Envoy Gateway that fronts platform services is the **`platform-gateway` component**.

- Component definition (what chart + values are used):
  - `gitops/ameide-gitops/environments/_shared/components/platform/control-plane/gateway/component.yaml`
- Values layering (6-layer composition used by ArgoCD ApplicationSet):
  - `gitops/ameide-gitops/argocd/applicationsets/ameide.yaml`
- Values files that actually apply to `platform-gateway`:
  - `gitops/ameide-gitops/sources/values/_shared/platform/platform-gateway.yaml`
  - `gitops/ameide-gitops/sources/values/env/<env>/platform/platform-gateway.yaml`

There are similarly named values files under `sources/values/env/<env>/apps/` (for app workloads) that are **not wired** into `platform-gateway` and should not be treated as the source of truth for platform gateway routing.

### 3.1 Gateway routing

1. **gRPC internal traffic**
   - Keep `GRPCRoute` rules for internal services (platform, threads, workflows, etc.).
   - Ensure the referenced Gateway listener section is an HTTP/2-capable listener (commonly `grpc-internal`).
   - Prefer `Exact` service matches to avoid regex portability problems.
   - Ensure `grpc-internal` listener exists everywhere and is stable:
     - `port: 9000`
     - `hostname: envoy-grpc`
   - Do not set `GRPCRoute.spec.hostnames` to `api.*` when attaching to `grpc-internal` (listener hostname is `envoy-grpc`); routes will not attach.
   - Ensure all required platform APIs are routed (for example, add `ameide_core_proto.platform.v1.InvitationService` if/when SDK callers use it via the gateway).
   - Concrete references (gitops repo):
     - `gitops/ameide-gitops/sources/charts/apps/gateway/templates/grpcroute-platform.yaml`

2. **Browser-facing traffic**
   - Default stance: browser does **not** call internal gRPC listeners directly.
   - If we intentionally expose browser-to-RPC, do it explicitly with `HTTPRoute` and a known protocol (Connect or gRPC-Web) and document it as a separate surface.

### 3.2 Environment variables and secrets (clean separation)

We must stop mixing “public” and “server-only” endpoints.

- **Server-only gRPC base URL** (BFF + service-to-service):
   - Provide a dedicated env var (example: `AMEIDE_GRPC_BASE_URL`) via ConfigMap/Secret.
   - Do not rely on `NEXT_PUBLIC_*` variables for internal gRPC routing.
   - Concrete references (gitops repo):
     - `gitops/ameide-gitops/sources/charts/apps/www-ameide-platform/templates/configmap.yaml` currently wires `NEXT_PUBLIC_ENVOY_URL` with a comment that server routes depend on it; this should be split into:
      - `AMEIDE_GRPC_BASE_URL` (server-only) for gRPC upstream calls (internal gateway / DNS)
      - `NEXT_PUBLIC_*` only for browser-facing configuration (or removed if not needed client-side)

- **Browser base URL**:
   - Browser should target `/api` (BFF) unless a dedicated public RPC route exists.

### 3.3 Platform service deployment toggle

Onboarding and org resolution depend on platform RPC availability. Ensure the platform Helm release is enabled and routable through the intended gRPC listener (see also `backlog/428-onboarding.md` blockers).

### 3.4 Concrete GitOps deltas (P0/P1)

P0 — make gRPC-internal deterministically usable by server callers:

1. Ensure `grpc-internal` listener exists everywhere (port 9000, hostname `envoy-grpc`) and `grpcRoutes.*.sectionName` attaches to it.
   - **Security requirement:** `grpc-internal` must be non-public (no port 9000 on a public LB). In GitOps this is implemented as a **separate internal Gateway** running in the control-plane namespace (`argocd`) with a **ClusterIP** data plane Service.
   - In-cluster callers use `http://envoy-grpc:9000` (an `ExternalName` alias in the environment namespace) which resolves to the stable ClusterIP service in `argocd`.
2. Ensure platform routing is enabled in `platform-gateway` values for each environment:
   - `gitops/ameide-gitops/sources/values/env/dev/platform/platform-gateway.yaml` already enables `grpcRoutes.platformService.enabled: true`
   - Add equivalent `grpcRoutes.platformService.enabled: true` (and required ports) to:
     - `gitops/ameide-gitops/sources/values/env/staging/platform/platform-gateway.yaml`
     - `gitops/ameide-gitops/sources/values/env/production/platform/platform-gateway.yaml`
3. For internal routes (`sectionName: grpc-internal`), omit `grpcRoutes.*.hostname` (or set it to `envoy-grpc`); do not set it to `api.<env>.ameide.io`.
4. Split Next.js env vars so server calls do not depend on `NEXT_PUBLIC_ENVOY_URL`:
   - Add `AMEIDE_GRPC_BASE_URL` to `gitops/ameide-gitops/sources/charts/apps/www-ameide-platform/templates/configmap.yaml`
   - Update app/server code to require server-only base URL and remove implicit reliance on `NEXT_PUBLIC_*` for internal gRPC.

P1 — preempt route gaps and improve clarity:

1. Add `ameide_core_proto.platform.v1.InvitationService` to:
   - `gitops/ameide-gitops/sources/charts/apps/gateway/templates/grpcroute-platform.yaml`
2. Rename platform service port semantics to reflect gRPC:
   - `gitops/ameide-gitops/sources/charts/apps/platform/templates/service.yaml` (`name: grpc` and/or `appProtocol: grpc`)

### 3.5 Service port semantics (recommended)

To make gRPC intent obvious to proxies and humans, update service port naming/appProtocol where applicable:

- Platform service chart currently names the service/container port `http`:
  - `gitops/ameide-gitops/sources/charts/apps/platform/templates/service.yaml`
  - `gitops/ameide-gitops/sources/charts/apps/platform/templates/deployment.yaml`
- Prefer `name: grpc` and/or `appProtocol: grpc` for gRPC backends (while continuing to run TCP probes for health).

---

## 4) API Semantics Follow-ups (Proto + AuthZ)

Transport determinism fixes routing failures, but org/onboarding correctness still requires contract hardening:

- Prefer “identity from auth context”, not user-supplied IDs (`backlog/300-400/329-authz.md`).
- Add explicit org resolution RPCs (`GetOrganizationByKey` / `ResolveOrganizationIdentifier`) to avoid list+scan patterns (ties to `backlog/428-onboarding.md`).
- Evaluate removing `orgKeys` filters from `ListOrganizationsRequest` when realm/membership shape stabilizes (`backlog/300-400/333-realms.md` and org proto reviews).

---

## 5) Work Items

### P0 (must ship together)

1. SDK: deterministic entrypoints and removal of runtime auto-detection
2. Next.js: server-side calls use Node/gRPC entrypoint; browser uses browser/Connect entrypoint
3. Platform service: migrate to connect-node HTTP/2 server runtime
4. Gateway/GitOps: ensure `platform-gateway` enables platform GRPCRoutes and attaches to `grpc-internal` deterministically (no hostname mismatch)
5. Next/GitOps: split server-only RPC base URL (`AMEIDE_GRPC_BASE_URL`) from any `NEXT_PUBLIC_*` config; server must not depend on public vars

### P1 (follow-up hardening)

1. Add `InvitationService` to platform GRPCRoute (preempt onboarding route gaps)
2. Rename platform service port semantics to reflect gRPC (`name: grpc` and/or `appProtocol: grpc`)
3. Add integration tests that fail loudly on protocol mismatch (no more “mysterious unimplemented 404”)
4. Tighten org listing semantics and membership gating (align with authz policy)

---

## 6) Verification / Done Criteria

- Calling `OrganizationService/ListOrganizations` and `GetOrganization` from Next server runtime returns real RPC statuses (no HTTP 404).
- Browser continues to function via BFF `/api` without needing direct gRPC access.
- SDK usage is deterministic (import path makes the protocol choice explicit).
- GitOps gateway listener/routing is documented and matches the SDK transport choice.

---

## GitOps Implementation Status (landed in `ameide-gitops`)

### Landed

- **`grpc-internal` is non-public**: internal Gateway (`<env-ns>-grpc-internal`) runs in `argocd` with a ClusterIP data plane, so port `9000` is never exposed on the public LB. `envoy-grpc` in the environment namespace is an `ExternalName` alias to `argocd/envoy-<env-ns>-grpc-internal`.
- **Key Gateway templates** (for review / diffs): `gitops/ameide-gitops/sources/charts/apps/gateway/templates/gateway.yaml`, `gitops/ameide-gitops/sources/charts/apps/gateway/templates/envoyproxy-internal.yaml`, `gitops/ameide-gitops/sources/charts/apps/gateway/templates/envoy-grpc-internal-service.yaml`, `gitops/ameide-gitops/sources/charts/apps/gateway/templates/envoy-grpc-service.yaml`.
- **Gateway routes enabled for staging/prod**: `grpcRoutes.platformService/workflowsService/inferenceService` enabled and attached to `grpc-internal` (no hostnames) in:
  - `gitops/ameide-gitops/sources/values/env/staging/platform/platform-gateway.yaml`
  - `gitops/ameide-gitops/sources/values/env/production/platform/platform-gateway.yaml`
- **Platform GRPCRoute coverage**: added `ameide_core_proto.platform.v1.InvitationService` match in `gitops/ameide-gitops/sources/charts/apps/gateway/templates/grpcroute-platform.yaml`.
- **Server-only vs public Envoy env vars**:
  - `www-ameide-platform` now renders `AMEIDE_GRPC_BASE_URL` from `envoy.url` and requires `NEXT_PUBLIC_ENVOY_URL` from `envoy.publicUrl` in `gitops/ameide-gitops/sources/charts/apps/www-ameide-platform/templates/configmap.yaml`.
  - `www-ameide` now follows the same split and adds `envoy.publicUrl` to env value files (`gitops/ameide-gitops/sources/charts/apps/www-ameide/templates/configmap.yaml` + `gitops/ameide-gitops/sources/values/env/*/apps/www-ameide.yaml`).
- **Service port semantics**: platform Service/container port renamed to `grpc` and `appProtocol: grpc` in `gitops/ameide-gitops/sources/charts/apps/platform/templates/service.yaml`.

### Tests added

- Helm unit coverage for determinism and env separation:
  - `gitops/ameide-gitops/sources/charts/apps/gateway/tests/rpc_determinism_env_test.yaml`
  - `gitops/ameide-gitops/sources/charts/apps/www-ameide-platform/tests/rpc_env_test.yaml`
  - `gitops/ameide-gitops/sources/charts/apps/www-ameide/tests/configmap_test.yaml`

To run locally:

- `helm unittest sources/charts/apps/gateway`
- `helm unittest sources/charts/apps/www-ameide-platform`
- `helm unittest sources/charts/apps/www-ameide`

### ArgoCD verification (example: local)

- `local-platform-gateway` is `Synced`, and `GRPCRoute/platform` contains `ameide_core_proto.platform.v1.InvitationService`.
- Public Envoy Service does not expose port `9000`; `envoy-<env-ns>-grpc-internal` exists as ClusterIP in `argocd`.
- Confirm `www-ameide-platform-config` has both vars:
  - `AMEIDE_GRPC_BASE_URL=http://envoy-grpc:9000`
  - `NEXT_PUBLIC_ENVOY_URL=https://api.<env>.ameide.io`
- Confirm platform Service port is `grpc` and has `appProtocol: grpc`.

### Remaining TODOs (non-GitOps)

- `ameideio/ameide` (`services/www_ameide_platform`): remove server-side dependency on `NEXT_PUBLIC_ENVOY_URL`; require `AMEIDE_GRPC_BASE_URL` for server gRPC upstream; keep browser on Connect/BFF (`/api`).
- CI/release: ensure images used by staging/prod integration jobs include the Next.js env-var change, then keep the test-job env wiring aligned (`AMEIDE_GRPC_BASE_URL` internal; `NEXT_PUBLIC_ENVOY_URL` public).
