# Architecture conformance analysis – scope & roadmap  

    1. Layered Architecture & Package Boundaries (folder layout, dependency direction). ← STARTED in this message.
    2. Model-Transformation Pipeline (parser → IR → validator → target) and SOLID separation.
    3. Proto ⇄ Domain ⇄ Storage pattern & Repository abstractions.
    4. Cross-cutting concerns: Observability, Build/SDK, Security hooks.

## Topic 1 – Layered architecture & package boundaries  

Expected from ADR-034
• Distinct layers: Infrastructure ↔ Storage ↔ Domain ↔ Service ↔ API ↔ SDK/Client.
• Domain layer must be technology-agnostic and have no outward imports to storage, DB or HTTP.
• Upper layers depend on abstractions (repositories, services) not on concrete persistence.

Evidence in the repo

    1. Folder / package mapping
       • infra/ → Infra layer (docker-compose, k8s manifests, OTel, etc.)
       • packages/ameide_core-storage → Storage layer (SQL, AGE, blob).
       • packages/ameide_core-domain → Domain layer; contains only Pydantic models & business rules.
       • packages/ameide_core-api → API layer; exposes FastAPI routes + GraphQL, sits atop “services”.
       • packages/ameide_core-…-model2* / workflows-* / agent-* → Transformation / service helpers (service layer).
       → Mapping aligns 1-to-1 with layered diagram.
    2. Import direction check
       • grep shows no reference to `sqlalchemy`, `fastapi`, or `ameide_core_storage` inside `core-domain` – domain is clean.
       • Storage packages (`core-storage/sql`) import domain models for mapping (`DomainUser`, `DomainWorkflow`) but not vice-versa → DIP respected.
       • Service layer (e.g. `core-api/services/user_service.py`) imports both domain and storage abstractions as expected.
    3. Technical boundaries
       • Domain objects (`agents.models.AgentRun`, `users.models.User`, …) are immutable and free of ORM annotations – matches “rich domain, tech-agnostic”.
       • Storage entities subclass `BaseEntity` (SQLAlchemy) and include `.to_domain()` / `.from_domain()` translators – good separation.
       • Graph provider (`core-graph-age/provider.py`) stays inside storage layer; higher packages refer only to its façade.
    4. Areas of drift / red flags
       A. Repository mismatch
          `UserService` injects `ameide_core_storage.sql.base.Repository` but passes domain objects to `.create()` / `.update()`.
          The generic Repository expects SQLAlchemy entities → compile-time error. Either
         • add a DomainRepository abstraction, or
         • call `User.from_domain()` before persisting.
          This violates boundary contract and must be fixed.
       B. “services” sub-folder lives inside `core-api`. While permissible, it slightly blurs API vs Service layers; consider promoting them to a dedicated package to avoid
accidental route/logic coupling.
       C. Some services (e.g. `AgentService`) work purely with proto models and ignore domain layer, acceptable for façade pattern but must not leak into other bounded contexts.
    5. Conclusion for Topic 1
       • Overall package structure and dependency direction follow the documented layered architecture.
       • One concrete deviation (Repository/domain mismatch) needs correction to keep layers truly independent.
       • Recommend adding automated import-graph checks (e.g. `flake8-import-order` or `layer-linter`) to stop future boundary erosion.

## Topic 2 Model-Transformation Pipeline (Parser → IR → Validator → Target)

    1. What we expect (ADR-034)
       • A strict, four-stage flow shared by all model types.
         Source model → Parser → IR (vendor-neutral) → Validator → Transformer → Target artefact/runtime.
       • SRP: each stage in its own package.
       • OCP: new validators / transformers can be plugged in without touching core code.
    2. Evidence in the code base
       a. Parsers
          • BPMN XML: packages/ameide_core-workflows-bpmn2model/parser.py (plus element-factory & mapper).
          • WFDESC, ArchiMate, etc. have analogous loader/mapper pairs (identical folder structure).
       b. IR layer
          • Workflows: core-workflows-model/WorkflowModel (rich, frozen Pydantic v2 object).
          • Agents:  core-agent-model/AgentModel (plus state schema, edges).
          • Both located in domain-type packages and import only stdlib / pydantic ⇒ tech-agnostic.
       c. Validators
          • Workflows: WorkflowValidator in core-workflows-model/validator.py (structure, flow, gateway checks).
          • Agents: validator_factory.py + node_validators strategy pattern; validate_agent_model() orchestrates.
       d. Transformers / Codegens
          • Workflow → Temporal: core-workflows-model2temporal/transformer.py then generator.py.
          • Workflow → Camunda, Agent → LangGraph, Agent → Airflow etc. follow the same pattern.
          • All transformers accept the IR and return plain dicts ready for templating.
       e. OCP hooks
          • ValidatorFactory allows runtime registration of custom node validators.
          • GeneratorFactory pattern for Temporal codegen injects language generators (Python/TS) via DI.
    3. Alignment score – mostly good
       ✔ Packages respect the stage boundaries; imports flow “down” only.
       ✔ Validators are fully decoupled; new ones can be registered without edits.
       ✔ Transformers never import storage/ORM or FastAPI; they remain pure functions.
       ✔ Codegen uses DI so additional languages or template engines can be dropped in.
    4. Deviations / risks discovered
       A. Validation not enforced before transformation
          generator.ContextBuilder calls transformer.transform() directly; WorkflowValidator is never invoked.
          ⇒ Risk of generating code from an invalid model.
       B. Duplicate graph utilities
          platform-common/utils/graph.py holds canonical algorithms, but core-workflows-model/graph_utils.py re-implements the same
functions (topological_sort, find_cycles, …).
          ⇒ Violates “single source of truth”, may diverge over time.
       C. Vendor hints leaking into IR
          WorkflowModel carries “impl” flags on tasks and Temporal-specific runtime_hints; this dilutes the “vendor-neutral IR” promise.
Safer: move vendor specifics to extensions or runtime_hints only.
       D. Parser → Validator linkage is manual
          There is no façade that guarantees “parse → validate → transform” as a single safe call. Responsibility is left to each
service.
       E. Minor naming duplication
          temporal_config appears twice in the dict returned by WorkflowToTemporal.transform(); harmless but noisy.
    5. Recommendations


        1. Add a lightweight Pipeline façade (e.g. WorkflowPipeline.run(bpmn_xml) -> validated IR) that always executes Validator before
Transformer; expose it in each *2target package.

        2. Delete duplicated graph_utils.py and import the generic functions from core-platform-common.utils.graph instead. Run tests to
be sure behaviour matches.

        3. Move task.impl and other runtime-specific flags into WorkflowModel.runtime_hints or extensions to keep the IR pure.

        4. In code-gen entry points (TemporalGenerator.generate, AgentLangGraphGenerator, etc.) assert validator.validate(model) before
transformation.

        5. Add a pre-commit rule or import-linter to prevent packages under *model* from importing their own graph algorithms when
platform-common provides them.


## Topic 3 Proto ⇄ Domain ⇄ Storage pattern & Repository abstraction

    1. Target pattern (ADR-034)
       Client request → Proto model (wire) → Domain model (business) → Storage entity (persistence).
       Responsibilities:
       • Proto models expose only transport-concerns (ids, auth, pagination…).
       • Domain models are technology-agnostic, immutable, rich in behaviour.
       • Storage layer owns ORM entities plus Repository façade; maps to/from Domain.
       • Service layer orchestrates mapping but knows nothing of ORM internals.
    2. Evidence in code base

       A. Proto layer
          • Located in packages/ameide_core-proto; each service has request/response/event models derived from base.ServiceRequest/Response.
          • Uses strict pydantic configs (schema_version, id generation, enum values).
          • No imports from domain or storage. ✔

       B. Domain layer
          • Located in packages/ameide_core-domain (+ workflows/agent dedicated packages).
          • Pure Pydantic v2, frozen, with behaviour methods (add_role, complete(), etc.).
          • No FastAPI/SQLAlchemy/network imports. ✔

       C. Storage layer
          • packages/ameide_core-storage/sql – SQLAlchemy models subclass BaseEntity; contain to_domain()/from_domain() methods.
          • Graph & Blob providers live in sibling folders with similar adapters.
          • Generic Repository[Entity] abstracts CRUD on AsyncSession.

       D. Mapping chain in practice
          • SQL entities call Domain.*.from_domain / to_domain exact as intended.
          • Service layer (core-api/services/*) usually takes proto request, performs business logic, returns proto response.
          • Example path (ideal): FastAPI route (JSON) → proto.AgentExecutionRequest → AgentService → domain logic → persistence via Repository → proto.AgentExecutionResponse.
    3. Misalignments & gaps


        1. Repository misuse in UserService
              UserService.constructor receives Repository but operates on Domain User objects, calling `graph.create(user)` where `user` is domain (not SQL) – violates contract;
will crash at runtime.
              ⇒ Fix: convert via storage.User.from_domain or introduce DomainRepository abstraction.

        2. Other services bypass persistence altogether
              AgentService & WorkflowService keep in-memory dicts; persistence layer is skipped so Proto ↔ Domain ↔ Storage chain is broken. Acceptable as stub, but annotate TODOs;
ensure tests don’t treat it as production.

        3. Bidirectional mapping lives inside storage entities only; no generic Mapper interface. This is fine but ties mapping knowledge to the storage package (acceptable per ADR yet
consider an explicit Mapper for multi-DB support).

        4. Domain models leak persistence ids
              WorkflowModel includes graph_uuid / src_id; AgentModel similar. They belong to storage/provenance concerns, not core business. Might be okay as metadata but violates
strict purity principle.

        5. No automatic conversion helpers in API layer
              Routes hand-craft response DTOs (`UserResponse(...)`) instead of a shared assembler; risk of drift when domain changes.
    4. Overall compliance rating
       • Proto and Domain layers are clean.
       • Storage layer clean but its Repository is mis-used in at least one service.
       • The chain is conceptually implemented but not enforced; stubs skip persistence.
       • Medium refactor effort required to bring UserService + future services in line and to decouple provenance ids from domain core.
    5. Recommendations
       a. Introduce `DomainRepository` protocol (`create(domain_obj)` → domain_obj) implemented by adapters (SqlUserRepository etc.) that wrap the generic SQL Repository internally.
       b. Refactor UserService to use that adapter, converting with `storage.User.from_domain`.
       c. Provide BaseMapper utility or `dataclass-mapper` pattern to avoid manual to_domain/from_domain duplication.
       d. Ensure all FastAPI responses are built via dedicated “Assembler” / “Presenter” functions to centralise mapping rules.
       e. Add unit tests asserting that no package in core-domain imports sqlalchemy / fastapi (enforce via import-linter).



## Topic 4 Cross-cutting concerns (Observability, Build & SDK, Security/Multi-tenancy)

A. Observability

Expected:
• OpenTelemetry instrumentation in every service, traces shipped to OTel-Collector → Tempo; metrics → Prometheus; logs → Loki (per ADR-034 diagram).
• Sensitive attributes filtered (PII, SQL statements).
• Health endpoints for each micro-service.

Findings:

    1. Infrastructure: `infra/observability/*` provides ready-to-run Grafana/Loki/Tempo/Prometheus/OTel-Collector stacks with sensible filters (`attributes` processor strips
db.statement, http.request.body). ✔
    2. Code: no occurrence of `opentelemetry`, `trace.get_tracer`, `MeterProvider`, or logging exporters in runtime packages (`core-api`, `services`, `*-service`). Routes and services
use standard logging only. ✖
    3. No FastAPI middleware for trace / log correlation (`LoggingMiddleware`, `OTLPSpanExporter`, etc.).
    4. Health checks: only OTel-Collector exposes `/health`; application services lack `/healthz` endpoints.
    5. Business-event trace filter configured (`filter/business_events`) but events are never tagged (`attributes[\"business_event\"] == true`) in code.

Impact: Tracing diagram exists in docs but runtime is silent; observability gap will surface once multiple services deploy.

Recommendations:
• Add OpenTelemetryMiddleware (fastapi-instrumentation) to core-api; export OTLP to collector.
• Provide helper in platform-common (instrument_app(app, service_name)) used by each service, sets resource attributes (service.name, version).
• Define logging handler that writes to standard out in JSON + otelSpanID correlation.
• Tag domain-level important operations (model.create, execution.fail) with business_event = true.

B. Build / SDK

Expected:
• SDK Phase 1 (client ops, CLI) already usable; CI builds, publishes wheel; just build generates API clients from proto.
• Codegen Manager orchestrates transpilers; CLI wraps common flows.

Findings:

    1. `packages/ameide_core-sdk` created: CLI scaffolding, sub-modules (artifact, client, codegen, runtime) present.
    2. CLI commands (`import_model`, `transpile`, `validate`, `query`) exist but all bodies are TODO stubs → no functional client yet.
    3. No generated API client inside SDK; no reference to proto definitions or OpenAPI URL.
    4. Build automation: `Makefile` in SDK missing; root `Makefile` has `make build` but not wired to SDK.
    5. No GitHub Actions / `justfile` entry that publishes SDK artifact.

Impact: SDK architecture complies in structure but is not yet implemented; doc states “SDK initiated ✅” yet functionality <10 %.

Recommendations:
• Short-term: Scaffold minimal AmeideClient using httpx + FastAPI’s /openapi.json or pre-generated stubs; wire CLI commands to that.
• Automate poetry build && twine upload in CI.
• Add “transpile” implementation that shells into respective *2target packages.
• Integrate ameide_sdk.codegen with validator before codegen (ties back to Topic 2 recos).

C. Security & Multi-tenancy

Expected:
• AuthN/Z, tenant isolation, secrets management; proto models already carry tenant_id, auth_token.
• API layer uses FastAPI dependencies to enforce auth; storage supports tenant filters; ADR-034 lists multi-tenant support as “near-term”.

Findings:

    1. Proto layer: `ServiceRequest.auth_token`, `tenant_id` fields present. ✔
    2. API layer: No JWT / OAuth2 dependencies; routes unauthenticated. `dependencies.py` config lacks current-user provider. ✖
    3. Storage: SQL models include `owner_id` but nothing for `tenant_id`; no row-level security.
    4. Secrets: `infra/docker-compose.yml` supplies DB credentials in plain YAML; no Vault / SOPS integration.
    5. Tests: no authorization cases.

Status Update (2025-01-24):

    1. ✅ Created `packages/ameide_core-platform-auth` with OAuth2/OIDC provider abstraction
    2. ✅ Implemented Auth0 and Keycloak providers with full JWT validation
    3. ✅ Added Keycloak service to docker-compose for local auth
    4. ✅ Created authenticated route wrappers with tenant isolation
    5. ✅ Added tenant_id to all SQL models with composite indexes
    6. ✅ Implemented TenantRepository for automatic row-level filtering
    7. ✅ Updated docker-compose with security documentation
    8. ⚠️  Security tests still pending
    9. ✅ Created ADR-036 for multi-tenancy strategy

Impact: Security gap; quick to exploit once public. Adding tenancy later may require large migrations.

Recommendations:
• Implement get_current_user() FastAPI dependency using JWT with RS256 keys; enforce in every router.
• Add tenant_id column to base entities; inject tenant filter in SQLAlchemy SessionEvents.do_orm_execute.
• Provide RBAC roles in domain (Role enum exists) and enforce via dependency.
• Move secrets to .env + Docker secrets; document rotation.
• Start ADR on tenancy strategy (schema-per-tenant vs row-level) now to avoid re-architecture.

Overall Topic 4 Verdict
Infrastructure is in place, but code instrumentation, SDK functionality and security controls lag behind documentation. Addressing these cross-cutting gaps is critical before moving
from proof-of-concept to production.


## Topic 5 Operational & Deployment Alignment (Docker / K8s / CI / Config)

    1. Container / docker-compose topology
       • `infra/docker/docker-compose.yml` spins up graph-db (Apache AGE + pgAdmin), model-api, React UI, Redis, LangGraph runtime, PostgreSQL + Temporal, full observability stack (OTel Collector, Prom, Loki, Tempo, Grafana).
       • All images are parameterised with env-vars; health-checks are defined; services share the single `ameide-network`.
       • OpenTelemetry endpoints are injected (`OTEL_EXPORTER_OTLP_ENDPOINT=http://observability-collector-otel:4317`). Good match with ADR.
       • Dev hot-reload supported via `DEV_MODE` flag and volume mounts.
       ✔ Local developer experience mirrors documented stack.
    2. Kubernetes reference
       • `backlog/800-k8s.md` provides an extensive multi-tenant cluster blueprint (namespaces, RBAC, network-policies, operator flow).
       • No actual helm charts or K8s manifests in `infra/k8s/`; blueprint is documentation-only at this stage.
       ✖ Implementation gap: Moving from docker-compose to cluster is still aspirational.
    3. CI/CD & Build automation
       • Root `Makefile` offers targets for tests, lint, package builds, docker stack up/down; good for local dev but nothing automates releases.
       • No `.github/workflows/`, Jenkinsfile or hosted CI definition => build pipeline undefined.
       • `justfile` referenced in Rust-sub-repo guidelines does not exist for Python part.
       • SDK, images and charts are not pushed to any registry automatically.
    4. Configuration & Secrets management
       • Env vars are read from host `.env` (dot-compose expects `${POSTGRES_USER}`…), but secrets sit in plaintext docker-compose and git.
       • K8s blueprint suggests Vault/ESO, but code / manifests not present.
       • No evidence of 12-factor config helper (e.g., pydantic-settings) in API code.
       
       Status Update (2025-01-24):
       • ✅ Added pydantic-settings based configuration in services/core-api/config.py
       • ✅ Created .env.example with all required variables documented
       • ✅ Added security documentation to docker-compose.yml
       • ✅ Created manage-secrets.sh script for Docker secrets in production
       • ⚠️  Still need to integrate with external secret managers (Vault/AWS)
    5. Operations observability
       • docker-compose wiring exposes OTel, Grafana, etc. – matches Topic 4 infra.
       • Makefile target `stack-logs` exists; helpful.
    6. Fit ↔ ADR
       Strengths:
       • Local stack faithfully implements services and observability diagram.
       • Healthchecks & resource labels embedded (OTEL_RESOURCE_ATTRIBUTES).
       Gaps / risks:
       • K8s manifests, Helm charts, ArgoCD apps, and Tenant-Operator referenced in docs are not in graph.
       • No CI pipeline; release process manual.
       • Secrets in plain text; tenancy isolation not yet codified.
       • Makefile builds packages but does not publish images to registry tagged by semver.
    7. Recommendations


        1. Bootstrap GitHub Actions (or preferred CI) workflows:
              • matrix: py 3.11 + OS; steps: lint → unit tests → build wheels → docker build → push to GHCR.
              • cache poetry & docker layers for speed.

        2. Create `infra/k8s/` with Helm charts or Kustomize overlays that replicate docker-compose config; include OTEL sidecars & health probes.

        3. Introduce `pydantic-settings` in each service, defaulting to `.env` but allowing secret injection (`/vault/secrets/...`).

        4. Add SOPS-encrypted `secrets.yaml` example and README for decrypt flow.

        5. Ship Tenant-Operator scaffold (Kopf) and CRDs referenced in blueprint.

        6. Provide `just up` / `just deploy-dev` shortcuts alongside Makefile to standardise commands.



## Topic 6 # Deep-dive: AMEIDE build-system

The graph’s build pipeline is a layered stack that takes source models & code and turns them into three kinds of artefacts:

• Python wheels (one per package)
• Generated runnable code (LangGraph agents, Temporal workflows, etc.)
• Docker images + docker-compose stack for local dev

It is stitched together through Poetry, a monorepo Makefile, helper scripts and per-service Dockerfiles.

Below is a component-level walkthrough and an assessment of robustness & gaps.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Python package build & dependency health

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Structure
• Every directory under packages/ is an independent Poetry project (pyproject.toml, dist/ wheels already built).
• Versions are manually maintained (no mono-versioning).
• Internal dependencies are expressed as path = "../../core-domain" references; external libs pinned in each package.

Automation
• Root Makefile build-packages iterates all pyprojects and runs poetry build, producing wheels in dist/.
• scripts/verify-dependencies.py is invoked from make verify-deps; it parses all pyprojects, detects:
  – cyclic internal deps
  – accidental duplicate naming
  – shows hierarchy layers.

Quality gates
• make lint → ruff + mypy
• make test → pytest over every package
• make test-services runs service-specific tests with their own Poetry venv to avoid heavy deps collisions.

Observations / issues
✓ Good isolation; any package can be released to PyPI independently.
✓ Dependency-cycle script is handy; few projects adopt this early.
✖ Version drift possible; no tool (e.g. poetry-version plugin or changeset) to bump coordinated releases.
✖ No cross-package type-checking; mypy runs per package path but not with “installed” wheels.
✖ No cached Poetry installs in CI because CI is not yet wired (see §5).

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Model & code generation build

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Goal: transform author-authored source (BPMN, WFDesc, ArchiMate) into executable agents/workflows.

Key pieces
• packages/ameide_core-workflows-*2model and core-agent-*2model - parse source → IR.
• packages/ameide_core-workflows-model2temporal and core-agent-model2langgraph – IR → language-specific artefacts using Jinja2 templates.
• packages/ameide_core-agent-build and core-workflows-build – “builder orchestration” wrappers that:

    1. Load/validate IR
    2. Call transformer + generators
    3. Drop artefacts into `build/generated/**`
    4. Optionally emit tests/docs/docker + `.env`

Example:
scripts/build-sample-order.py demonstrates end-to-end build:

BPMN → WorkflowModel → TemporalGenerator → build/generated/temporal_workflows
WFDesc → AgentModel → LangGraphGenerator → build/generated/langgraph_agents

Integration with runtime
Dockerfile for runtime-agent-langgraph has build-arg GENERATED_PATH=../../build/generated so generated code is copied during image build.

Observations
✓ Build scripts are asynchronous and streaming-log friendly.
✓ Runtime hints are injected at build time, keeping templates clean.
✖ Validation stage is optional; builder skips WorkflowValidator if caller forgets.
✖ Generated code is not cached; a change in a template forces full regenerate & image rebuild.
✖ No templating unit tests; breaking Jinja variables will surface only at runtime container build.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Database migration & storage build

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

SQL layer
• alembic.ini + migrations/ for Postgres/AGE.
• make migrate invokes Alembic against the compose Postgres.

Graph layer
• AGE specific bootstrap SQL scripts in infra/databases/postgres-age; executed by container entrypoint.

Builder integration
Builders do not automatically call migrations; developers must run make migrate before launching stack.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Container & local-stack build

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

docker-compose
• infra/docker/docker-compose.yml wires all runtime services + observability + db.
• Healthchecks, OTEL env, volume mounts for live code are configured.
• make dev (default path) = build all images (docker-compose build --parallel) then DEV_MODE=true docker-compose up.

Service Dockerfiles
• Multi-stage (python-slim builder → runtime).
• Agent runtime receives generated code via build-arg path mount.
• Temporal worker image (not shown here) copies build/generated/temporal_workflows.

Observations
✓ Parallel build reduces wait time.
✓ OTEL exporter configured but code still needs instrumentation.
✖ Secrets (POSTGRES_PASSWORD, OPENAI_API_KEY) are interpolated from host .env; no docker secrets or vault.
✖ On rebuild the entire monorepo is sent as context; .dockerignore exists but may be missing large artefacts .venv, etc.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. CI / CD pipeline (current state & missing parts)

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Current repo lacks .github/workflows or similar. Build reliability therefore depends on local execution of Make targets.

Suggested flow

    1. CI matrix: Python 3.10-3.11 + Ubuntu; steps:
       • cache Poetry, run `make lint`, `make test-all`, `scripts/verify-dependencies.py`
       • `make build-packages` produce wheels as artefacts
       • Build images for model-api, runtime-agent-langgraph, runtime-workflows-temporal
    2. On `main` tag: publish wheels to internal PyPI, images to GHCR, push generated docs to `gh-pages`.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Rust / codex-rs corner

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

(Not present in repo clone yet, but build policy is documented.)
When folder is added it will follow:
• justfile with just fmt, just fix, cargo test --all-features
• GitHub Action with just invocation.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Build-system strengths & pain points

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Strengths
• Clear separation: Python packaging ↔ codegen artefacts ↔ Docker images.
• Single entry Makefile covers dev lifecycle; easy onboarding.
• Dependency-cycle verifier prevents a common monorepo foot-gun.
• Generated artefacts are treated as build output, never committed.

Weaknesses / gaps

    1. No CI/CD yet – risk of “it works on my laptop”.
    2. Package versioning manual and unsynchronised; difficult to know compatible sets.
    3. Validation optional in generation; broken IR might compile.
    4. Instrumentation and security (SBOM, scan) missing from image build.
    5. Large docker build contexts; need `.dockerignore` polish & build-cache mounts.
    6. No build cache / remote caching for generated code; repeated local builds take minutes.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Recommendations

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Quick wins (1-2 days)
• Add GitHub Action with steps described above; cache Poetry and Docker layers.
• Enforce WorkflowValidator / validate_agent_model in builders before codegen.
• Add .dockerignore for **/.venv, **/__pycache__, *.pytest_cache, build/generated (the latter can be mounted via volume in dev mode).
• Introduce poetry-dynamic-versioning plugin and a root version constant to keep sub-packages aligned.

Medium term
• Move build orchestration from shell/Make to justfile or Nox to get cross-platform reproducibility.
• Add remote Cache (e.g., BuildKit local cache volume) and pass --mount=type=cache,target=/root/.cache/pip in Dockerfiles.
• Generate SBOM during image build (syft) and sign (cosign) in CI.
• Publish pre-built LangGraph and Temporal artefacts to an internal artefact graph so runtime images don’t need to COPY from monorepo context.

Long term
• Adopt a proper monorepo build orchestrator (Bazel or Pants) once code-gen & cross-language matrix grows; it will handle graph-based incremental builds natively.
• Embed provenance tags in every build (git SHA, dependency tree) and export to AGE for traceability.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


# Multi-tenancy review & implementation blueprint

Scope
Design goal is “many customers (or internal BUs) on one platform with hard data-plane isolation.” The repo already contains ADR 800-k8s.md that sketches a full tenancy architecture, but the running code only carries hints (tenant_id fields in proto models). Below is:

    1. Current state audit
    2. Gap analysis per layer
    3. Detailed implementation plan (MVP → hardened)
    4. Code & schema changes list you can action this sprint

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Current state audit

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Data model
• Proto: ServiceRequest includes tenant_id field (optional).
• Domain / Storage: SQL models have owner_id but no tenant_id column; no row-level security (RLS).
• API: no auth middleware; routes accept all requests.
• Graph (AGE): no logical tenant separation (schemas/labels shared).
• Object storage (MinIO) & Redis caches are shared instances.

Access control
• Role enum (ADMIN, USER, SERVICE, GUEST) in domain, but no enforcement.
• No JWT/RBAC or external IdP integration.

Deployment
• docker-compose stack is single-tenant; K8s blueprint describes namespace isolation but no manifests/Operator.

Observability
• OTel collector can accept tenant header, but span exporter doesn’t set it.

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Gap analysis

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Layer-by-layer

    1. Identity & AuthN
       – Need primary identity: user & service principals with tenant claim.
       – Token issuance (Auth0 / Keycloak / Azure AD) not wired yet.
    2. API & Service layer
       – Must reject requests lacking valid tenant in token.
       – Multi-tenant aware DI: repositories auto-scope queries.
    3. Storage
       a. SQL/Postgres/AGE
          • Add `tenant_id` column to every entity.
          • Enable Postgres RLS policies (`CREATE POLICY ... USING (tenant_id = current_setting('app.tenant_id')::uuid)`).
          • AGE inherits RLS automatically.
       b. MinIO / object store
          • Bucket-per-tenant or prefix policy with IAM.
       c. Redis
          • Key prefix strategy: `{tenant}:{key}` or multi-DB.
    4. GraphQL/FastAPI
       – Dependency `get_current_user()` injects tenant into request state.
       – Response DTO masks foreign tenant IDs.
    5. Observability
       – Middleware adds `tenant_id` attribute on spans, logs, metrics; OTel collector’s `attributes` processor already passes through.
    6. Build & CI
       – Unit tests for RLS; integration tests spin per-tenant namespaces with fixture tokens.
       – Tenant-operator controller code missing.

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Implementation blueprint

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Step 0 – agree on “row-level security + namespace isolation” hybrid model (matches ADR 800).

Step 1: Identity & Auth
a. Choose IdP (Keycloak for OSS).
b. Configure OIDC for services; public key cached.
c. Implement auth.py in core-api:

    oauth2_scheme = OAuth2AuthorizationCodeBearer(...)
    def get_current_user(token=Depends(oauth2_scheme)) -> UserContext:
        # decode, verify, extract sub, tenant, roles

d. Add FastAPI dependency to every router.

Step 2: Storage changes
SQL

    1. Alembic migration:      op.add_column('users', sa.Column('tenant_id', sa.String(), nullable=False))
           ...
           for tbl in ['agents', 'workflows', ...]:
               op.add_column(tbl, sa.Column('tenant_id', sa.String(), nullable=False))
    2. Configure RLS in same migration:      op.execute("ALTER TABLE users ENABLE ROW LEVEL SECURITY")
           op.execute(\"\"\"CREATE POLICY tenant_isolation
                        ON users USING (tenant_id = current_setting('app.tenant_id'))\"\"\")

       – Set `app.tenant_id` per-session: in `Repository.__init__` after obtaining AsyncSession:
         `await session.execute(text("SET app.tenant_id = :tid"), {'tid': tenant_id})`.

Graph (AGE)
• Use Postgres schemas: schema name = tenant UUID, set search_path.
• Tenant-operator pre-creates schema when onboarding.

Step 3: Repository adapter
Wrap existing Repository into TenantRepository:

    class TenantRepository(Repository):
        def __init__(self, session: AsyncSession, entity_cls, tenant_id: str):
            super().__init__(session, entity_cls)
            self.tenant_id = tenant_id
            await session.execute(sa.text("SET app.tenant_id = :tid"), {'tid': tenant_id})

DI layer creates repos with tenant_id from request context.

Step 4: Graph Provider isolation
AGE: pass search_path = tenant_uuid in connection string.
Neo4j: label all nodes with tenant_uuid property & use WHERE n.tenant = $tenant.

Step 5: API contract changes
– Mark tenant_id as required in ServiceRequest for non-internal calls.
– Accept header X-Tenant-ID as override for internal service-to-service when mTLS is established.

Step 6: Tenant-Operator (Kubernetes)
– Scaffold controller (python Kopf or Go) watching Tenant CRD; responsibilities:
  • create namespace tenant-<uuid>
  • apply networkpolicy, resourcequota, vault secrets
  • issue Postgres CREATE SCHEMA, run migrations.
– GitOps path: ArgoCD application per tenant.

Step 7: Observability
FastAPI middleware:

    class TenantTelemetryMiddleware(BaseHTTPMiddleware):
        async def dispatch(self, request, call_next):
            tenant = request.state.tenant_id
            with tracer.start_as_current_span('request', attributes={'tenant': tenant}):
                return await call_next(request)

OTel collector: ensure tenant attr is allowed, not dropped.

Step 8: Tests & migration scripts
– Pytest fixture tenant_token creates tenant, obtains JWT, sets header.
– Integration tests: user from tenant A cannot access tenant B’s workflows (assert 404).
– Load tests: run two namespaces concurrently.

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    1. Sprint-sized code change list

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Backend
• Add tenant_id column + RLS migration (4–6 hrs).
• Update SQL models’ from_domain / to_domain (1 hr).
• Implement auth.py & TenantRepository (4 hrs).
• Refactor dependencies.py to inject tenant-bound repos (2 hrs).
• Add FastAPI middleware to attach tenant_id to request.state (1 hr).

Graph layer
• Extend AGEProvider to accept schema param and issue SET search_path (2 hrs).

SDK
• Add client.login(tenant, user, pwd) returning JWT.
• CLI flag --tenant stored in config.

Dev & Ops
• .env.example add TENANT_ID.
• Update docker-compose: postgres POSTGRES_INITDB_ARGS: -c row_security=on.
• Provide scripts/create-tenant.py helper.

Documentation
• Write docs/multitenancy.md with API usage & RLS policy summary.

With these focused changes you will have end-to-end functional multi-tenancy (row-level security + schema isolation) performant enough for MVP and extensible toward the full namespace-operator vision in ADR 800.




# Deep-dive: test strategy, e2e vs. integration folders

The repo contains three testing layers:

    1. Unit (inside each package under `src/**/__tests__`) – already covered by `make test`.
    2. End-to-end (repo root `tests/e2e`) – exercise model→IR→code-gen pipelines without external services.
    3. Integration (repo root `tests/integration`) – exercise a *running* docker-compose stack.

Below is an audit of layers 2 & 3: what they cover, how they work, and where they need hardening.

----------------------------------------------------------------------------------------------------------------------------------

## A. End-to-End tests  (tests/e2e)

Purpose
• Validate deterministic transformations from authoring formats to runtime artefacts.

Structure
• test_agent_pipeline.py
  – WFDesc JSON → AgentModel → LangGraph code gen.
  – Verifies: node counts, metadata, generated python files contain expected functions/classes.
  – Uses NetworkX adapter to analyse cycles, unreachable nodes (good graph health check).
• test_workflows_pipeline.py (+ refactored variant)
  – BPMN XML → WorkflowModel → Temporal code gen; inspects tasks/gateways and generated workflows code.
• Shared fixtures in conftest.py; sample assets under assets/.

Strengths
✓ Do not require external DB or docker; run fast in CI.
✓ Cover full pipeline including builders (AgentBuilder & WorkflowBuilder).
✓ Validate generated code syntactically by scanning strings.

Weak spots / gaps

    1. They never import/compile the generated python to ensure syntactic validity; a template typo could pass string tests.
    2. Workflow-validator and agent-validator are **not** called; invalid source passes.
    3. No negative cases (expect failure on malformed BPMN).
    4. Fixture path hack in `conftest.py` only adds three specific packages; risk of ImportError when new ones appear.
    5. `test_workflows_pipeline_refactored.py` duplicates logic; should parameterise instead.

Quick fixes
• After code-gen, importlib.import_module generated files (use importlib.util.spec_from_file_location) → compile.
• Call WorkflowValidator.validate() and assert no exception.
• Add an invalid BPMN sample and assert validator raises WorkflowValidationError.
• Switch to pytest.mark.parametrize to avoid duplicate tests.
• Use sys.setdlopenflags() pattern or importlib.machinery.ModuleSpec to load generated package under unique name.

----------------------------------------------------------------------------------------------------------------------------------

## B. Integration tests  (tests/integration)

Purpose
• Validate that docker-compose stack works as a system and observability plumbing is functional.

Files
• test_services.sh – shell script pinging service endpoints for HTTP 200; quick smoke test.
• test_telemetry.sh – shell script that: health-checks OTel, Prom, Loki, Tempo, Grafana; makes API calls to generate traces;
queries Tempo & Prometheus to assert ingestion.
• test_telemetry_integration.py – Python version of telemetry script; generates activity event, polls Prometheus & Tempo.

Execution model
Requires developer to have run make dev (stack up). Tests are skipped in CI (there’s no CI yet). They are intended for local use.

Strengths
✓ Easy to run (bash tests/integration/test_services.sh).
✓ Covers full golden path for telemetry with real HTTP queries.
✓ Prints friendly summary.

Weaknesses / risks

    1. No `pytest` harness – results are not integrated into existing test report; a failing curl still exits 0 because of pipe
errors suppressed (`set -e` missing).
    2. Hard-coded localhost ports; fails if compose stack runs on custom network or inside CI service container.
    3. Not idempotent: Telemetry test sleeps fixed 5 s; may flake on slow machines.
    4. No automatic stack bring-up; developer must remember to run compose.
    5. Security: admin:admin password for Grafana embedded.
    6. No multi-tenant checks (relevant for forthcoming tenancy work).

Enhancement roadmap
• Convert shell tests to pytest with requests & assert so failures propagate; wrap in marker @pytest.mark.integration.
• Use docker SDK (pytest-docker or testcontainers-python) to spin up minimal stack inside test run; eliminates manual step and port
 collisions.
• Replace sleeps with polling helper until condition or timeout.
• Parameterise Grafana creds via env vars, default to GRAFANA_ADMIN_PASSWORD.
• Extend telemetry test:
  – tag generated span with tenant=test; query Tempo for attr filter.
  – assert Loki returns log lines with matching traceID.
• Add pytest --run-integration flag and skip otherwise; CI pipeline can run nightly.

----------------------------------------------------------------------------------------------------------------------------------

## C. Coverage summary

┌─────────────────────────┬───────────────────┬─────────┬─────────────────────┐
│ Module                  │ Unit              │ E2E     │ Integration         │
├─────────────────────────┼───────────────────┼─────────┼─────────────────────┤
│ Agent model pipeline    │ ✔                 │ ✔       │ —                   │
├─────────────────────────┼───────────────────┼─────────┼─────────────────────┤
│ Workflow model pipeline │ ✔                 │ ✔       │ —                   │
├─────────────────────────┼───────────────────┼─────────┼─────────────────────┤
│ Graph & mapping utils   │ partial           │ via e2e │ —                   │
├─────────────────────────┼───────────────────┼─────────┼─────────────────────┤
│ API routes              │ ✔ (package tests) │ —       │ smoke (services.sh) │
├─────────────────────────┼───────────────────┼─────────┼─────────────────────┤
│ Telemetry               │ —                 │ —       │ ✔                   │
├─────────────────────────┼───────────────────┼─────────┼─────────────────────┤
│ Security / auth         │ —                 │ —       │ —                   │
├─────────────────────────┼───────────────────┼─────────┼─────────────────────┤
│ Multi-tenancy           │ —                 │ —       │ —                   │
└─────────────────────────┴───────────────────┴─────────┴─────────────────────┘

----------------------------------------------------------------------------------------------------------------------------------

## D. Action list for the next sprint

    1. E2E
       • Compile & import generated code; fail on syntax error.
       • Add negative test cases.
       • Consolidate duplicate workflows test file.
    2. Integration
       • Port shell scripts to `pytest` + `testcontainers.Compose`.
       • Mark tests with `@pytest.mark.integration` & skip by default.
       • Poll instead of sleep; fail fast with explicit timeout.
       • Inject credentials via env; no hard-coded passwords.
    3. CI
       • GitHub Action matrix:
         – Stage 1: `make test` (unit & e2e).
         – Stage 2: `pytest -m integration` inside test-container if tag `nightly`.
       • Publish HTML coverage; require > 85 % on domain & transformers.
    4. Future
       • Add security tests: JWT required, RLS enforcement.
       • Multi-tenant integration tests once tenancy is implemented.

By tightening these areas the test suite will deliver fast feedback on functional regressions, infrastructure breakage
and—soon—tenant isolation, aligning quality gates with the ambitious architecture.
