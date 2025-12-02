### 🚀 NEXT.JS FRONTEND – BACKEND & API-GATEWAY INTEGRATION

**Status:** _Proposed_ &nbsp;&nbsp;|&nbsp;&nbsp; **Target Release:** `v0.3`

---

## 1. Context

We are adding a public-facing Next.js application (`services/web-app`).
Backend capabilities (domain graph, workflows runtime, provenance, etc.) are already
exposed through `core-api` and will soon be fronted by the new **Envoy
Gateway** (see backlog #011).

The web-app must:

1. Consume the same contracts (OpenAPI JSON, gRPC-web, protobuf TS types) that
   power internal clients.
2. Authenticate users via OIDC (Auth0 / Keycloak) once at the edge and forward
   an access token to downstream services.
3. Remain developer-friendly (`npm run dev`) while being fully reproducible in
   Bazel / OCI builds.

---

## 2. Goals / Definition of Done

1. **Type-safe SDK** – Generated TypeScript client (`@ameide/core-api-sdk`) from
   OpenAPI & protobuf definitions; consumed by Next.js via React Query hooks.
2. **Gateway routing** – HTTPRoute `/api/` forwards to `core-api` with JWT
   propagation; CORS enabled for web-origins.
3. **SSR compatibility** – Calls that require credentials happen only on the
   server (getServerSideProps / Route Handlers).  Public calls permitted in the
   browser use `NEXT_PUBLIC_API_BASE_URL`.
4. **Bazel build** – `bazel build //services/web-app:next_build` succeeds in CI;
   image published with commit SHA tag.
5. **Local DX** – `bazel run //services/web-app:serve` starts dev server with
   hot-reloading and uses a local Envoy config or proxy fallback.

---

## 3. High-level Tasks

| # | Task | Owner | Notes |
|---|------|-------|-------|
| 1 | Add rules_js & npm_translate_lock to MODULE.bazel |  | version 2.2.2 |
| 2 | Scaffold `services/web-app` (Next.js 14, App Router, TypeScript) |  |  |
| 3 | Generate `core-api` TS client with `openapi-generator-cli` in `packages/ameide_core-proto-tools` |  | publish as internal npm pkg |
| 4 | Integrate React Query / SWR hooks consuming SDK |  |  |
| 5 | Implement OIDC login flow via `next-auth` |  | env vars from Gateway ext-auth cookie |
| 6 | Configure env: `NEXT_PUBLIC_API_BASE_URL` for each cluster |  | configMap |
| 7 | BUILD.bazel: npm_package, js_library, oci_image rules |  | similar to backlog #011 spec |
| 8 | Helm/Kustomize manifests for Deployment & Service |  | reuse Gateway `/` path |
| 9 | E2E Cypress tests through Gateway (edge auth) |  | run in CI |

---

## 4. Architectural Decisions

* **gRPC-web vs. REST** – Initially REST using FastAPI-generated OpenAPI → less
  client complexity.  Re-evaluate gRPC-web when realtime streaming is needed.
* **Edge Auth** – Gateway performs OIDC code-flow, sets `id_token` cookie.
  Backend services validate JWT; frontend reads cookie only on server side.
* **Client-side Networking** – Use fetch-based wrapper; no axios to keep bundle
  size small and leverage Next.js Runtime.

---

## 5. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Bundle size explosion from SDK | Tree-shakeable ESM output, Webpack/SWC analyse step |
| SameSite / Secure cookie issues across sub-domains | Use single apex domain `ameide.ai` with `Path=/` |
| CORS misconfiguration | Automated e2e test validating preflight & credentials |
| Bazel learning curve for FE devs | Provide `make dev` shortcut that calls Bazel under the hood |

---

## 6. Acceptance Criteria

* Visiting `https://app.ameide.ai` shows login page; after OIDC sign-in user can
  list workflows via SDK call routed through Envoy `/api/workflows`.
* CI green: `bazel test //services/web-app/...` + Cypress tests.
* Documentation updated in `docs/frontend/`.

---

_Created: 2025-07-25_

