# 482 – Adding a New Service (Reference Checklist)

> **Deprecation notice (520):** This backlog assumes a CLI-driven scaffolding workflow (e.g., `ameide primitive scaffold`) that generates service/primitive skeletons. That approach is deprecated. Canonical v2 uses **`buf generate`** (pinned plugins, deterministic outputs, generated-only roots, regen-diff CI gate), and uses Backstage templates for repo bootstrapping. See `backlog/520-primitives-stack-v2.md`.

**Status:** Draft  
**Owner:** Architecture / Platform Enablement  
**Purpose:** Provide a single checklist and pointer map for spinning up a new Ameide service (platform service or primitive) with all cross-cutting requirements (security, testing, GitOps, documentation) enforced from day one.

> **Update (2026-01): 430v2 contract**
>
> Any references below to “integration packs”, `INTEGRATION_MODE`, or `tools/integration-runner` are legacy. Treat `backlog/430-unified-test-infrastructure-v2-target.md` as the normative repo contract (strict phases; native tooling; JUnit evidence; no pack scripts as the canonical path).

---

## 1. When to use this backlog

Use this checklist whenever you create a net-new service under `services/` or `service_catalog/`. It consolidates the required work from related backlogs so teams don’t miss key steps (Dockerfiles, AppSets, secrets, tests, documentation, etc.).

---

## 2. Topic Checklist & Source Backlogs

| Topic | Questions to answer | Primary references |
|-------|--------------------|--------------------|
| **Architecture & positioning** | Is this a primitive (Tier 2) or a platform service? How does it map to the extensibility model? | `481-service-catalog.md`, `479-ameide-extensibility-wasm.md`, `480-ameide-extensibility-wasm-service.md` |
| **Scaffolding & repo layout** | Which folders/files must exist (README, Dockerfiles, Tilt targets, catalog info)? | `service_catalog/*/_primitive/skeleton/README.md`, `service_catalog/*/_primitive/skeleton/Dockerfile.*`, `services/README.md` |
| **Dockerfiles & build pipelines** | Do we have dev/release Dockerfiles aligned with repo conventions? Are we leveraging multi-stage builds? | `service_catalog/.../Dockerfile.dev`, `Dockerfile.release`, `services/platform/Dockerfile.dev` (patterns) |
| **Secrets & configuration** | What secrets/config does the service need? Are ExternalSecrets defined (zero inline secrets per guardrail)? Does Vault bootstrap cover it? | `362-unified-secret-guardrails-v2.md`, `348-envs-secrets-harmonization.md` (historical), `services/<service>/README.md` examples |
| **Storage / dependencies** | Does the service rely on shared infrastructure (MinIO, Postgres, Temporal, etc.)? Where are the dependencies documented? | `380-minio-config-overview.md`, `442-environment-isolation.md`, service-specific backlogs (e.g., `359-resilient-deploy-sequencing.md` for sequencing) |
| **GitOps & deployment** | Which ApplicationSet/Helm chart entries are required? What sync wave & labels apply? | `364-argo-configuration-v3.md`, `375-rolling-sync-wave-redesign.md`, `367-bootstrap-v2.md`, `387/447-argocd-waves*.md`, `465-applicationset-architecture.md`, environment naming per `434-unified-environment-naming.md` |
| **SDK & workspace alignment** | Are protos consumed exclusively via Ameide SDKs, have Go/TS/Py packages been regenerated (`scripts/dev/check_sdk_alignment.py`), and do Dockerfiles follow the Ring 1/Ring 2 workspace-first rules? | `365-buf-sdks-v2.md`, `408-workspace-first-ring-2.md`, `405-docker-files.md` |
| **Networking & tenancy labels** | Are namespace/pod labels and NetworkPolicies configured correctly (tier, tenant, environment)? | `441-networking.md`, `434-unified-environment-naming.md`, `442-environment-isolation.md`, `459-httproute-ownership.md` |
| **Testing (unit/integration/e2e)** | Are tests classified and runnable under `ameide test` (Phase 0/1/2: contract → unit → integration) and `ameide test e2e` (cluster-only E2E), and do they emit JUnit evidence? | `backlog/430-unified-test-infrastructure-v2-target.md`, `backlog/537-primitive-testing-discipline.md`, `backlog/468-testing-front-door.md` |
| **Build/test verification & CI logs** | Have we run the repo front door and confirmed CI gates are aligned (JUnit evidence, strict phase semantics)? | `backlog/430-unified-test-infrastructure-v2-target.md`, `backlog/610-ci-rationalization.md`, `.github/workflows/*.yml` |
| **Observability & SLOs** | What metrics/logging/tracing does the service emit? Are dashboards updated? | `platform observability backlogs`, `services/README.md`, service-specific SLO docs |
| **Security & governance** | Does the service follow risk-tier rules, host-call policies, and secrets governance? | `476-security-and-trust.md`, `362-unified-secret-guardrails-v2.md`, `479-ameide-extensibility-wasm.md` (for Tier 1 runtime specifics) |
| **Documentation & ownership** | README, runbooks, rotation steps, owners? Has Backstage catalog info been created? | `service_catalog/.../README.md`, `backlog/481-service-catalog.md` (Backstage integration), `services/README.md` |
| **Tooling & developer experience** | Tilt target, devcontainer wiring, scripts? | `backlog/435-remote-first-development.md`, `backlog/581-parallel-ai-devcontainers.md`, `tools/*` docs |
| **CI/CD & release automation** | Are GitHub workflows/cd-service pipelines building, testing, and packaging Docker images? | `395-sdk-build-docker-tilt-north-star.md`, `405-docker-files.md`, repo-specific workflows |

---

## 3. Detailed Steps

1. **Decide service type**
  - Primitive (Domain/Process/Agent/UISurface) → use Backstage template (see `service_catalog/…/_primitive/template.yaml`).
   - Platform/legacy service → follow `services/README.md` contract.

2. **Scaffold files**
   - Create README (describe purpose, APIs, dependencies, rotation steps).
   - Add `Dockerfile.dev` & `Dockerfile.release` using skeleton patterns.
   - Add Tilt target (if applicable) and ensure `pnpm/golang` scripts exist.

3. **Define interfaces**
   - Proto definitions (if gRPC) under `packages/ameide_core_proto/...`.
   - Regenerate SDKs (`pnpm proto:generate`, Go tooling, Python packaging) and reference them. Run `scripts/dev/check_sdk_alignment.py` to confirm Go/TS/Py remain in sync. Services must consume APIs through the Ameide SDK packages as mandated by 365; no runtime imports of `packages/ameide_core_proto`. When services need primitive helpers (e.g., invoking `extensions-runtime`), add them to the SDKs instead of duplicating RPC glue.
   - Mirror the `extensions-runtime` workflow pattern: every service-specific CI job must install the Buf CLI and run the `scripts/ci/generate_*_stubs.sh` helpers before compiling or testing so workspace SDK stubs are refreshed from the BSR. Docker builds never invoke `buf generate`; they rely solely on the committed SDK artifacts.

4. **Secrets & config**
   - Identify required secrets, add ExternalSecrets referencing Vault keys (see backlog 362 for guardrails—no inline secrets or local fallbacks).
   - Update `foundation-vault-bootstrap` fixtures if new keys are needed.

5. **Storage/backing services**
   - Document dependencies (Postgres, Redis, MinIO). Reuse shared services where possible (e.g., MinIO per `380-minio-config-overview.md`).
   - Ensure required buckets or databases are provisioned via GitOps jobs, not manual steps.

6. **Testing**
   - Implement tests under the 430v2 contract (Unit → Integration → E2E) and ensure they run under the repo front doors:
     - CI gate: `ameide test` (Phase 0/1/2; must emit JUnit evidence).
     - Local agent/human inner loop: `ameide test` (Phase 0/1/2 only; local-only).
     - Cluster-only E2E: `ameide test e2e` (Playwright).
   - **Unit (Phase 1):** pure/local; no cluster tooling.
   - **Integration (Phase 2):** local mocked/stubbed only; no cluster tooling; no “mode” variables.
     - Go: use `//go:build integration` for Phase 2-only tests.
     - TS/Py: select integration tests via standard runner config (no per-component scripts).
   - **E2E (Phase 3):** cluster-only; Playwright-only (when applicable).

7. **GitOps deployment**
   - Create Helm chart (or reuse existing) under `charts/`.
   - Add component definitions/values under `gitops/ameide-gitops` following the dual ApplicationSet structure in backlog 465.
   - Label the component with the correct rollout phase (per backlog 447) so RollingSync orders it properly, and ensure environment naming matches backlog 434.

8. **Observability & health**
   - Add health endpoints (HTTP/grpc health) per `services/README.md`.
   - Wire metrics/logging/tracing (OTel exporters, etc.) and register dashboards/alerts.
   - Apply namespace/pod labels and NetworkPolicy hooks documented in backlog 441/442 (tier, tenant, environment).
   - Verify Dockerfiles and build scripts respect Ring 1/Ring 2 rules (workspace SDK copies only, no registry fetches) and that SDK publish pipelines remain separate (408).

9. **Documentation & runbooks**
   - Ensure README covers rotation steps, dependencies, integration tests, and owners.
   - Update catalog info (Backstage `catalog-info.yaml`) if applicable.

10. **Security/governance checks**
    - Validate host-call policies (if applicable), risk tier, and auth patterns.
    - Run `scripts/validate-hardened-charts.sh` (manual while guardrail CI is deferred).

11. **CI/CD wiring**
    - Add or update GitHub workflows (and `cd-service-images` entries where applicable) so the service runs unit/integration tests and builds both dev/release Docker images on PRs/main. Confirm the workspace-first Dockerfiles pass `policy/check_docker_sources.sh` (408/405) and the unit/integration suites are referenced in the workflow.
    - After wiring, re-run the workflows (e.g., `gh run list`, `gh run view <run-id>`) to ensure `CD / Packages` succeeds (no missing SDK surfaces per 365/393) and `CD / Service Images` publishes the dev/release images. Capture run IDs in the backlog for traceability, especially when shared components (e.g., `extensions-runtime`) introduce new SDK helpers or Docker targets that need to ship together.

---

## 4. Suggested References by Topic

| Topic | File(s) |
|-------|---------|
| MinIO details | `backlog/380-minio-config-overview.md`, `services/agents/README.md` (artifact usage) |
| GitOps/AppSet placement | `backlog/465-applicationset-architecture.md`, `backlog/447-waves-v3-cluster-scoped-operators.md`, `backlog/375-rolling-sync-wave-redesign.md`, `backlog/367-bootstrap-v2.md` |
| Secret guardrails | `backlog/362-unified-secret-guardrails-v2.md`, `backlog/348-envs-secrets-harmonization.md` |
| SDK + Ring alignment | `backlog/365-buf-sdks-v2.md`, `backlog/408-workspace-first-ring-2.md`, `backlog/405-docker-files.md` |
| Test infrastructure | `backlog/430-unified-test-infrastructure-v2-target.md`, `backlog/468-testing-front-door.md` |
| Environment naming & networking | `backlog/434-unified-environment-naming.md`, `backlog/441-networking.md`, `backlog/442-environment-isolation.md` |
| Extensibility/Tier1 runtime | `backlog/479-ameide-extensibility-wasm.md`, `backlog/480-ameide-extensibility-wasm-service.md` |
| Service catalog patterns | `backlog/481-service-catalog.md`, `service_catalog/domains/_primitive/skeleton/*` |
| Bootstrap/devcontainer | `backlog/435-remote-first-development.md`, `backlog/444-terraform.md` (fallback), `backlog/491-auto-contexts.md` |

---

## 5. Open Questions

1. ~~Should this checklist be automated (lint script or Backstage action) so new services must acknowledge each step?~~ **Answered:** See [484-ameide-cli](484-ameide-cli.md) — `ameide primitive scaffold` automates this checklist.
2. Where should we store reusable Helm chart boilerplate for platform services (vs primitive scaffolds)?
3. Do we need a "service inception" PR template referencing these checkpoints?

---

## 6. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [484-ameide-cli](484-ameide-cli.md) | CLI scaffolding guidance is deprecated; see 520 for canonical `buf generate` gates |
| [430v2 unified test infrastructure](430-unified-test-infrastructure-v2.md) | Test structure requirements (phases + JUnit evidence) |
| [434-unified-environment-naming](434-unified-environment-naming.md) | GitOps structure, labels |
| [481-service-catalog](481-service-catalog.md) | Backstage catalog integration |
