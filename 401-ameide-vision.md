# 401 – Ameide Vision (Current State, MECE)
**Status:** Draft (2025-11-23)  
**Intent:** Provide a MECE snapshot of what Ameide delivers today, anchored in shipped services, SDKs, guardrails, and pipelines. This is the “reality layer” to contrast with forward-looking scenarios.

> ⚠️ **Remote-first update:** Delivery and DevContainer sections below describe the legacy k3d-based workflow (local registry + Argo bootstrap). The current source of truth for remote-first development is [435-remote-first-development.md](435-remote-first-development.md); treat any k3d references here as historical context.

## 1) Data & Governance Plane (metamodel, org, transformation)
- **Graph/Repository (knowledge system):** `services/graph` and `services/repository` expose CRUD + version history over the unified metamodel via Connect/gRPC, backed by Postgres with Flyway, pg-mem unit tests, and Testcontainers e2e (`services/graph/README.md`, `services/repository/README.md`). Repository dual-writes metadata + elements; Graph hosts the element/relationship handlers.
- **Platform (org/teams/roles):** `services/platform` serves `ameide_core_proto.platform.v1` for organizations, teams, roles, users, invitations (`services/platform/README.md`). Tilt/Helm deploy with Vault-managed DB creds.
- **Transformation (governance/runtime):** `services/transformation` runs lifecycle APIs (initiatives → transformations, stages, milestones, metrics) and projects runtime facts back into the graph (`services/transformation/README.md`). It is the execution plane for methodologies and role maps.
- **Workflows (orchestration surface):** `services/workflows` provides the gRPC workflow API; Python workers (`services/workflows_runtime`, `services/workflow_worker`) execute Temporal workflows with strict `TEMPORAL_*` env validation and integration tests against Temporal test servers (`services/workflows_runtime/README.md`).
- **Guardrails:** Charts require ExternalSecrets for DB access (Vault), db-migration hooks fail fast, and `gitops/ameide-gitops/scripts/validate-hardened-charts.sh` renders overlays in CI (referenced across service READMEs).

## 2) Agentic & AI Plane
- **Agents control plane:** `services/agents` (Go) stores node definitions and tenant-scoped agents, supports catalog streaming of node snapshots, routing simulation, and MinIO-backed artifact uploads (LangGraph compilation TODO). gRPC surface: `ameide_core_proto.agents.v1` (`services/agents/README.md`).
- **Inference runtimes:**  
  - **Inference Service (Python):** LangGraph + FastAPI SSE/WS with OpenAI + Langfuse integration (`services/inference/README.md`). Currently still uses raw `ameide_core_proto` stubs for some calls (gap noted in 393/389).  
- **Agents Runtime (Python “Inference v2”):** Hydrates agents from the Agents service over gRPC, evaluates declarative routes, streams responses; falls back to plan-driven stub when no LLM configured (`services/agents_runtime/README.md`).  
  - **Inference Gateway (Go):** Connect/gRPC-Web entry, auth, rate limiting, optional Redis buffering, protocol translation to inference runtime (`services/inference_gateway/README.md`).
- **Conversation storage:** `services/chat` and `services/threads` persist conversations/threads in Postgres and delegate generation to inference (`services/chat/README.md`, `services/threads/README.md`).
- **Runtime workers:** Temporal workers (`workflows_runtime`, `workflow_worker`) execute orchestration tasks and callbacks.
- **Artifacts & storage:** MinIO for compiled agent artifacts (`services/agents/README.md`), Redis optional in inference-gateway for buffering, Postgres for state across services.

## 3) SDKs, Contracts, and Clients
- **Single proto source:** `packages/ameide_core_proto` is the authoritative schema source, but runtime services do **not** import it directly. Services consume contracts via the wrapper SDK surfaces (see `backlog/715-v6-contract-spine-doctrine.md` and `backlog/300-400/393-ameide-sdk-import-policy.md`).
- **SDK surfaces:** TS (`packages/ameide_sdk_ts`), Go (`packages/ameide_sdk_go` / `github.com/ameideio/ameide-sdk-go`), Python (`packages/ameide_sdk_python`) wrap transports, auth, retries, tracing, and export proto types at runtime.
- **AmeideClient contract:** Unified option defaults, metadata/auth/timeout/retry/tracing interceptors, request ID injection, and helper factories are defined and tested via manifest (`sdk/manifest/ameide-client.json`, `backlog/389-ameide-sdk-ameideclient.md`). TS/Go/Python implementations now align; integration coverage still evolving.
- **Import policy guardrails:** Runtime code must consume SDKs. Unit tests may import proto types when needed, but Scenario Slice / capability E2E runners must call primitives via wrapper SDK clients only (see `backlog/715-v6-contract-spine-doctrine.md`). Policy scripts enforce:
  - no direct `@buf/*` / `buf.build/gen/*` / `packages/ameide_core_proto/**` imports in runtime code,
  - TS service proto barrels re-export from `@ameideio/ameide-sdk-ts/proto.js`,
  - no Go `replace` hacks that bypass the intended SDK resolution,
  - lock hygiene (`scripts/policy/run_all.sh`), see `backlog/300-400/393-ameide-sdk-import-policy.md`.
- **Current publish status (evidence from 388/390):** Go SDK tagged `v0.1.0` with GHCR aliases; Python SDK on PyPI (`2.10.1`) but GHCR lacks SemVer tags; TS SDK not on npmjs and GHCR only carries dev/hash; configuration package lacks SemVer tags (`backlog/388-ameide-sdks-north-star.md`, `backlog/390-ameide-sdk-versioning.md`).

## 4) Frontend & UX Plane
- **Authenticated platform:** `services/www_ameide_platform` (Next.js) with Keycloak SSO via NextAuth v5, Redis-backed sessions, seeded personas, and Playwright e2e suites for auth/token/role flows (`services/www_ameide_platform/README.md`). Uses SDK clients via local proto barrel for dev/test.
- **Public portal:** `services/www_ameide` provides marketing/docs and redirects to the platform entrypoint; uses SDK proto barrel for dev/test, no auth (`services/www_ameide/README.md`).
- **Proto clients in UI:** Both apps re-export proto namespaces and SDK clients from `src/proto` shims; no Buf/Verdaccio dependencies remain (per import policy).

## 5) Delivery, Ops, and Guardrails
- **GitOps & bootstrap:** Argo CD (single control plane in the `argocd` namespace) reconciles every namespace (`ameide-dev`, `ameide-staging`, `ameide-prod`). DevContainers now only hydrate tooling (`pnpm`, Azure CLI, kubelogin, Telepresence) and fetch AKS credentials; there is no local k3d bootstrap. The `ameide` CLI owns the inner loop (Telepresence connect/intercept + hot reload + verification) targeting stable ingress URLs (see `README.md`, `backlog/435-remote-first-development.md`).
- **Ingress & TLS:** Envoy Gateway remains the sole TLS terminator. Wildcard DNS for `*.dev.ameide.io` plus the Telepresence-only `*.local.ameide.io` listener is managed under `.devcontainer/` (`README.md`, `backlog/417-envoy-route-tracking.md`).
- **Secrets & compliance:** ExternalSecrets + Vault are mandatory; charts fail without `existingSecret` references. db-migration hooks and `gitops/ameide-gitops/scripts/validate-hardened-charts.sh` enforce credentials and template validity (called out in service READMEs).
- **Build & images:** BuildKit scripts under `build/scripts/` produce OCI images; service Dockerfiles copy `pnpm-lock.yaml` before frozen installs to pin deps (documented in per-service READMEs and policy in 393). Helm/Argo use image tags from publishes; GitOps repo lives at `gitops/ameide-gitops` (nested).

## 6) Release, Versioning, and Artifacts
- **Authority:** Git tags `vX.Y.Z` drive release versions; non-tag builds use dev/snapshot versions (`backlog/390-ameide-sdk-versioning.md`).
- **Pipeline:** `.github/workflows/cd-packages.yml` computes versions, builds/tests TS/Go/Python/config, publishes to npm/PyPI/GHCR, and signs SHA tags (blueprint in `backlog/388-ameide-sdks-north-star.md`).
- **State of play:** Go SDK aligned (repo + GHCR aliases). TS SDK missing public SemVer publish; GHCR TS/Python/Configuration lack SemVer/`latest` aliases. Lock hygiene scripts exist but CI still runs TS services against workspace links; inference service still bypasses SDK in places (gaps in 388/393).

## 7) Observability, Security, Compliance
- **Tracing & metrics:** SDKs emit tracing metadata; Go SDK uses otelgrpc; TS/Python tracing interceptors aligned per manifest (`backlog/389-ameide-sdk-ameideclient.md`). Langfuse wired in inference runtime (`services/inference/README.md`). ClickHouse dependency noted for data plane in GitOps (`README.md`).
- **Health & readiness:** HTTP/Connect/gRPC health endpoints standardized across services (examples in `services/README.md` and service-specific READMEs).
- **Auth & identity:** Keycloak realms managed via onboarding; platform app handles auth/SSO; Redis sessions; Backchannel logout and token storage documented in `services/www_ameide_platform/README.md`. No separate governance service yet; policy is via Git/PR.

## 8) Gaps vs the long-term vision
- **Catalog/governance plane absent:** No tenant-facing service catalog, feature specs, or risk-tiered governance engine. Change control is Git/PR + Helm/Argo guardrails, not policy-aware automation.
- **Agent pipeline maturity:** Agents service streams node snapshots only; compiled LangGraph artifact publishing and deployment flow are TODO. Inference service still uses raw proto stubs instead of SDK clients; stub fallback remains default when no LLM configured.
- **SDK distribution gaps:** TS SDK not published to npmjs; GHCR missing SemVer aliases for TS/Python/Configuration; CI still leans on workspace links rather than published tarballs for TS.
- **Preview/promotion flow:** Preview environments exist via Tilt/Argo patterns but not tied to tenant-scoped feature sets or governance checks; no “feature spec” pipeline.
- **Telemetry depth:** Tracing exists at SDK layer; service-level metrics/OTel coverage uneven; ClickHouse noted but not fully described here.

## Where to look next (evidence map)
- **Service capabilities:** `services/**/README.md` (Graph, Repository, Platform, Transformation, Workflows, Agents, Agents Runtime, Inference, Inference Gateway, Chat, Threads, www_ameide_platform, www_ameide).
- **SDK policy & client contract:** `backlog/393-ameide-sdk-import-policy.md`, `backlog/389-ameide-sdk-ameideclient.md`, `sdk/manifest/ameide-client.json`.
- **Versioning & publishing:** `backlog/388-ameide-sdks-north-star.md`, `backlog/390-ameide-sdk-versioning.md`, `.github/workflows/cd-packages.yml`.
- **Ops/bootstrapping:** `ameide-gitops/bootstrap/bootstrap.sh` now owns the GitOps/bootstrap CLI while `ameide/.devcontainer/postCreate.sh` + `tools/dev/bootstrap-contexts.sh` (in `ameideio/ameide`) provide the developer bootstrap; supporting scripts remain in `README.md`, `gitops/ameide-gitops/scripts/validate-hardened-charts.sh`, and `build/scripts/*`.
