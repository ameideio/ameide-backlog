# 481 – Adding a New Service (Reference Checklist)

**Status:** Draft  
**Owner:** Architecture / Platform Enablement  
**Purpose:** Provide a single checklist and pointer map for spinning up a new Ameide service (platform service or primitive) with all cross-cutting requirements (security, testing, GitOps, documentation) enforced from day one.

---

## 1. When to use this backlog

Use this checklist whenever you create a net-new service under `services/` or `service_catalog/`. It consolidates the required work from related backlogs so teams don’t miss key steps (Dockerfiles, AppSets, secrets, tests, documentation, etc.).

---

## 2. Topic Checklist & Source Backlogs

| Topic | Questions to answer | Primary references |
|-------|--------------------|--------------------|
| **Architecture & positioning** | Is this a primitive (Tier 2) or a platform service? How does it map to the extensibility model? | `481-service-catalog.md`, `479-ameide-extensibility-wasm.md`, `480-ameide-extensibility-wasm-service.md` |
| **Scaffolding & repo layout** | Which folders/files must exist (README, Dockerfiles, Tilt targets, catalog info)? | `service_catalog/*/_controller/skeleton/README.md`, `service_catalog/*/_controller/skeleton/Dockerfile.*`, `services/README.md` |
| **Dockerfiles & build pipelines** | Do we have dev/release Dockerfiles aligned with repo conventions? Are we leveraging multi-stage builds? | `service_catalog/.../Dockerfile.dev`, `Dockerfile.release`, `services/platform/Dockerfile.dev` (patterns) |
| **Secrets & configuration** | What secrets/config does the service need? Are ExternalSecrets defined (zero inline secrets per guardrail)? Does Vault bootstrap cover it? | `362-unified-secret-guardrails-v2.md`, `348-envs-secrets-harmonization.md` (historical), `services/<service>/README.md` examples |
| **Storage / dependencies** | Does the service rely on shared infrastructure (MinIO, Postgres, Temporal, etc.)? Where are the dependencies documented? | `380-minio-config-overview.md`, `442-environment-isolation.md`, service-specific backlogs (e.g., `359-resilient-deploy-sequencing.md` for sequencing) |
| **GitOps & deployment** | Which ApplicationSet/Helm chart entries are required? What sync wave & labels apply? | `364-argo-configuration-v3.md`, `375-rolling-sync-wave-redesign.md`, `367-bootstrap-v2.md`, `387/447-argocd-waves*.md`, `465-applicationset-architecture.md`, environment naming per `434-unified-environment-naming.md` |
| **SDK & workspace alignment** | Are protos consumed exclusively via Ameide SDKs, have Go/TS/Py packages been regenerated (`scripts/dev/check_sdk_alignment.py`), and do Dockerfiles follow the Ring 1/Ring 2 workspace-first rules? | `365-buf-sdks-v2.md`, `408-workspace-first-ring-2.md`, `405-docker-files.md` |
| **Networking & tenancy labels** | Are namespace/pod labels and NetworkPolicies configured correctly (tier, tenant, environment)? | `441-networking.md`, `434-unified-environment-naming.md`, `442-environment-isolation.md`, `459-httproute-ownership.md` |
| **Testing (mock/cluster)** | Do integration packs follow the unified test contract (single implementation, mock/cluster modes) and exercise critical flows (e.g., primitive → `extensions-runtime` → host-call)? | `430-unified-test-infrastructure.md`, `480-ameide-extensibility-wasm-service.md`, `services/_templates/README.md`, `tools/integration-runner/README.md` |
| **Build/test verification & CI logs** | Have we run unit + integration suites locally, verified `docker build` (dev + release) succeeds, and confirmed GitHub workflows (`extensions-runtime.yml`, `cd-service-images.yml`) include the new service? Did the latest `gh run view <id>` logs pass SDK alignment and publishing checks? | `430-unified-test-infrastructure.md`, `395-sdk-build-docker-tilt-north-star.md`, `405-docker-files.md`, `408-workspace-first-ring-2.md`, `.github/workflows/*.yml` |
| **Observability & SLOs** | What metrics/logging/tracing does the service emit? Are dashboards updated? | `platform observability backlogs`, `services/README.md`, service-specific SLO docs |
| **Security & governance** | Does the service follow risk-tier rules, host-call policies, and secrets governance? | `476-security-and-trust.md`, `362-unified-secret-guardrails-v2.md`, `479-ameide-extensibility-wasm.md` (for Tier 1 runtime specifics) |
| **Documentation & ownership** | README, runbooks, rotation steps, owners? Has Backstage catalog info been created? | `service_catalog/.../README.md`, `backlog/481-service-catalog.md` (Backstage integration), `services/README.md` |
| **Tooling & developer experience** | Tilt target, devcontainer wiring, scripts? | `429-devcontainer-bootstrap.md`, `367-bootstrap-v2.md`, `tools/*` docs |
| **CI/CD & release automation** | Are GitHub workflows/cd-service pipelines building, testing, and packaging Docker images? | `395-sdk-build-docker-tilt-north-star.md`, `405-docker-files.md`, repo-specific workflows |

---

## 3. Detailed Steps

1. **Decide service type**
   - Primitive (Domain/Process/Agent/UISurface) → use Backstage template (see `service_catalog/…/_controller/template.yaml`).
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
   - Set up `__mocks__/` and `__tests__/integration/` per backlog 430 (single implementation, mock + cluster modes).
   - Ensure `run_integration_tests.sh` uses the standard tooling and fails fast on missing envs. Include end-to-end scenarios when the service depends on shared components (e.g., primitive → `extensions-runtime` → host-call) so regression tests cover cross-service seams.

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
| Test infrastructure | `backlog/430-unified-test-infrastructure.md`, `tools/integration-runner/README.md`, `services/_templates/README.md` |
| Environment naming & networking | `backlog/434-unified-environment-naming.md`, `backlog/441-networking.md`, `backlog/442-environment-isolation.md` |
| Extensibility/Tier1 runtime | `backlog/479-ameide-extensibility-wasm.md`, `backlog/480-ameide-extensibility-wasm-service.md` |
| Service catalog patterns | `backlog/481-service-catalog.md`, `service_catalog/domains/_controller/skeleton/*` |
| Bootstrap/devcontainer | `backlog/429-devcontainer-bootstrap.md`, `backlog/367-bootstrap-v2.md`, `435-remote-first-development.md` |

---

## 5. Open Questions

1. ~~Should this checklist be automated (lint script or Backstage action) so new services must acknowledge each step?~~ **Answered:** See [484-ameide-cli](484-ameide-cli.md) — `ameide primitive scaffold` automates this checklist.
2. Where should we store reusable Helm chart boilerplate for platform services (vs primitive scaffolds)?
3. Do we need a "service inception" PR template referencing these checkpoints?

---

## 6. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [484-ameide-cli](484-ameide-cli.md) | CLI `scaffold` command automates this checklist |
| [430-unified-test-infrastructure](430-unified-test-infrastructure.md) | Test structure requirements |
| [434-unified-environment-naming](434-unified-environment-naming.md) | GitOps structure, labels |
| [481-service-catalog](481-service-catalog.md) | Backstage catalog integration |
