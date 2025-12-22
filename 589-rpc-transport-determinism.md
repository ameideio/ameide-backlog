# 589: Deterministic RPC Transports (Connect + gRPC Target Architecture)

**Status:** In progress  
**Created:** 2025-12-22  
**Owner:** Platform Architecture / DX  
**Related:** `backlog/300-400/306-rpc-runtime-alignment.md`, `backlog/300-400/393-ameide-sdk-import-policy.md`, `backlog/428-onboarding.md`, `backlog/300-400/329-authz.md`, `backlog/300-400/333-realms.md`

---

## Executive Summary

We standardize RPC in Ameide so that **clients and gateways are deterministic** and protocol boundaries are explicit:

- **Internal canonical protocol:** gRPC over HTTP/2.
- **Browser access:** via a BFF (Next.js) using Connect, not direct-to-gRPC.
- **No runtime auto-detection** in SDKs. Environment guessing creates non-deterministic failures in modern ESM/Next.js runtimes.

This backlog item captures the end-state architecture and the required repo + GitOps changes to land it cleanly (no compatibility shims).

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

### 3.1 Gateway routing

1. **gRPC internal traffic**
   - Keep `GRPCRoute` rules for internal services (platform, threads, workflows, etc.).
   - Ensure the referenced Gateway listener section is an HTTP/2-capable listener (commonly `grpc-internal`).
   - Prefer `Exact` service matches to avoid regex portability problems.
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
      - `AMEIDE_GRPC_BASE_URL` (server-only) for gRPC upstream calls
      - `NEXT_PUBLIC_*` only for browser-facing configuration (or removed if not needed client-side)

- **Browser base URL**:
  - Browser should target `/api` (BFF) unless a dedicated public RPC route exists.

### 3.3 Platform service deployment toggle

Onboarding and org resolution depend on platform RPC availability. Ensure the platform Helm release is enabled and routable through the intended gRPC listener (see also `backlog/428-onboarding.md` blockers).

### 3.4 Service port semantics (recommended)

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
4. Gateway: confirm gRPC `GRPCRoute` is present for platform org methods and matches the listener used by server callers

### P1 (follow-up hardening)

1. Standardize server-only env vars for gRPC base URLs (remove `NEXT_PUBLIC_*` usage for internal gRPC endpoints)
2. Add integration tests that fail loudly on protocol mismatch (no more “mysterious unimplemented 404”)
3. Tighten org listing semantics and membership gating (align with authz policy)

---

## 6) Verification / Done Criteria

- Calling `OrganizationService/ListOrganizations` and `GetOrganization` from Next server runtime returns real RPC statuses (no HTTP 404).
- Browser continues to function via BFF `/api` without needing direct gRPC access.
- SDK usage is deterministic (import path makes the protocol choice explicit).
- GitOps gateway listener/routing is documented and matches the SDK transport choice.
