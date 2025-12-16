# backlog/367-4 – Release Governance & Analytics

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)

## Purpose
Standardize controlled release processes for automation outputs and agent runtimes across all methodology profiles, aligning with backlog/345 CI/CD expectations and preview/promotion flows (backlog/368).

## Scope
- Agent runtime release pipeline (`packages/codex-cli-platform`, executor images) with build → qualify → promote stages, SBOM/signing, manifest tracking.
- Artifact release bundles for automation runs including methodology context (Increment, PI Objective, ADM deliverable) plus evidence (tests, previews, docs) mapped to bronze/silver/gold lanes.
- Promotion tooling (UI/CLI) to alias previews, enforce gating policies (DoD, approvals, compliance), and capture change tickets/notifications.
- Analytics + audit exports: lead time, change failure rate, policy exceptions, release calendar integration, Monte Carlo forecast readiness.
- Policy-as-code engine enhancements allowing per-methodology overrides (e.g., ADM phase gate vs. Scrum DoD) with exception logging.
- Attestation registry – enforce the `governance.v1.Attestation` schema (kind, URI, checksum, produced_by_run) so promotion gates validate the same proof regardless of profile. Support aggregating attestations from multiple runs/timeboxes in a single release bundle.
- Exception lifecycle – capture request/approval/expiry/re-review metadata and automatically surface open exceptions on boards + release dashboards.
- Metrics built on method events (state changes, attestations, gate approvals, exceptions) flattened by the indexer so Scrum/SAFe/TOGAF dashboards consume the same facts.

## Deliverables
1. **Runtime release stack** – Build/scan/sign pipelines, manifest format (`release/agents/manifest.yaml`), version pinning in automation plans.
2. **Artifact promotion tooling** – CLI + UI for preview alias/promote, lane labeling, gating status, rollback actions.
3. **Governance policies** – Configurable rules referencing methodology artifacts (Increment accepted? Architecture Board sign-off?), blocking vs. advisory severity, exception flow with expiry + auto re-review notifications.
4. **Analytics dashboards** – Change metrics by methodology profile and lane; exception/waiver reporting; forecast readiness.
5. **Audit exports & integrations** – Structured bundles for compliance reviews, change calendars, incident correlation.

## Non-goals
- Defining methodology artifacts themselves (earlier stages).
- Dogfooding rollout logistics (Stage 3).

## Dependencies
- Automation orchestration producing structured evidence.
- Methodology profiles specifying DoD/phase gates.
- CI/CD infrastructure per backlog/345.
- Canonical element + relationship model from [backlog/300-ameide-metamodel.md](./300-ameide-metamodel.md) to keep release artifacts linked to the graph.

## Exit Criteria
- Every automation-derived change follows the same governed release pipeline with auditable manifests and policy checks.
- Stakeholders can inspect lane status, gating compliance, and readiness across methodologies via shared dashboards.
- Exceptions are captured with rationale/expiry, and promotion tooling supports fast rollback.

## Implementation guide

### Runtime release stack & manifest controls
- Reuse the existing image orchestration scripts to cover every agent runtime artifact. `build/scripts/build-all-images.sh` already tags and pushes the UI, `www_ameide_platform`, LangGraph runtime, and migrations with the Git SHA (`build/scripts/build-all-images.sh:35`), while `build/scripts/build-image.sh` handles the single-image CI path with reproducible tagging (`build/scripts/build-image.sh:4`). Extend both to (1) include the Codex CLI/executor images once `packages/codex-cli-platform` lands, (2) emit SBOMs + Cosign attestations per backlog/345, and (3) persist a machine-readable bundle under `release/agents/manifest.yaml` capturing image digests, SBOM URIs, Cosign statements, and the underlying plan metadata pulled from the LangGraph payload (`services/agents_runtime/src/agents_service/runtime.py:66`).
- The SDK release scripts (`release/publish.sh`, `release/publish_go.sh`, `release/publish_python.sh`) already derive versions from the Buf module and write manifests under `release/dist/*` (see `release/publish.sh:60`, `release/publish_go.sh:69`, `release/publish_python.sh:71`). Tie these manifests together with the runtime manifest so that a single promotion record contains every artifact (images + SDKs) consumed by automation plans (backlog/365).
- Promotion to production clusters today relies on mutating Helm value overrides via `scripts/update_release_manifests.py` (`scripts/update_release_manifests.py:19`) and then letting `infra/kubernetes/scripts/run-helm-layer.sh` enforce `helm test` + pending-release recovery (`infra/kubernetes/scripts/run-helm-layer.sh:143`). Keep those scripts as the “promote” stage but feed them with the manifest generated above (materials, signatures, SBOM URIs) so bronze/silver/gold lanes can reason over the exact build they are promoting.
- Align runtime bootstrap with the guardrails captured in backlog/362: the runtime pod + migrations already consume Vault-provisioned secrets (`services/agents_runtime/README.md`) and the preflight script `gitops/ameide-gitops/scripts/validate-hardened-charts.sh` validates the Layer 15 bundle before db-migrations. Promotion tooling must refuse to advance a lane if the guardrail script detects drift in ExternalSecrets or database URLs for the runtime.

### Shared contracts, SDK propagation, and release bundles
- The Buf workspace already exposes the data contracts needed to describe release governance: governance resources (`packages/ameide_core_proto/src/ameide_core_proto/governance/v1/governance.proto`), workflow lifecycle + automation hooks (`packages/ameide_core_proto/src/ameide_core_proto/workflows_runtime/v1/workflows_service.proto`), element/relationship projections (`packages/ameide_core_proto/src/ameide_core_proto/graph/v1/graph.proto` per backlog/300/303), and transformation context (`packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation.proto`). Extend these protos with explicit `ReleaseBundle`, `LaneStatus`, and `EvidenceAttachment` messages so that automation runs, governance cases, and promotion tooling speak the same types.
- Once the protos are extended, rely on the thin SDK pipeline (above) to regenerate TypeScript/Go/Python clients and surface release bundle helpers near where Workflow/Graph/Transformation clients are already exposed (`services/www_ameide_platform/lib/sdk/ameide-client.ts:18`). This keeps UI, CLI, and workers consistent with backlog/365’s managed SDK strategy.
- Wire release bundles back into the runtime by persisting LangGraph payload fingerprints, tool grants, and compiled artifact hashes when agents execute (`services/agents_runtime/src/agents_service/runtime.py:630`). Those fingerprints become the linkage between attestation objects in `governance.v1` and the manifest that policy gates evaluate.

### Promotion tooling & operator workflows
- The Workflows service already offers definition, versioning, rules, and Temporal orchestration primitives (`packages/ameide_core_proto/src/ameide_core_proto/workflows_runtime/v1/workflows_service.proto`) with concrete handlers in `services/workflows/src` (`definitions-graph.ts` for metadata, `rules-graph.ts` for scoped automation, `temporal/facade.ts` for execution, and `status-updates.ts` for telemetry). Build the “lane picker” and preview/promotion UX on top of this surface: every promotion should be a `WorkflowRun` annotated with the release bundle ID, lane (`bronze/silver/gold`), and gating policy snapshot.
- `services/workflows/src/repositories/rules-graph.ts:62` already models graph- or transformation-scoped automation rules; extend the schema to express gate types (DoD vs ADM phase) and reference methodology metadata from backlog/313/329 so that an Implementation Governance lane can require different checks from a Scrum increment.
- The platform UI currently mocks governance + release history via static data (`services/www_ameide_platform/features/graph/mock-data.ts:103` and `services/www_ameide_platform/app/(app)/org/[orgId]/governance/page.tsx:1`). Replace those mocks with live calls to the governance/workflows surfaces above so repository panels, transformation dashboards, and governance tabs show real-time gate status, exception banners, and lane progression.
- CLI parity: `packages/ameide_core_cli` already shells into the SDK publish scripts when running `ameide generate release` (`packages/ameide_core_cli/internal/commands/generate.go:78`). Add new subcommands that wrap promotion workflows (kick off the workflow, stream gate status, and surface rollback commands) so automation and operators follow the same happy path.

### Policy engine, evidence registry, and methodology overrides
- Governance data contracts exist but there is no backing service today—`packages/ameide_core_proto/src/ameide_core_proto/governance/v1/governance_service.proto` is only consumed by the web client (`services/www_ameide_platform/lib/sdk/ameide-client.ts:40`). Stand up a Governance service (Go or TS) that persists `GovernancePolicy`, `RequiredCheck`, `GovernanceCase`, and `CheckRun` entries, lines them up with organizations/tenants (backlog/329), and produces the policy snapshots consumed by promotion workflows.
- Treat evidence as first-class relationships in the graph. The LangGraph runtime already validates compiled artifacts, tool grants, and middleware (`services/agents_runtime/src/agents_service/runtime.py:112`), which can be serialized into the existing `graph.v1.Element`/`ElementRelationship` model defined in `packages/ameide_core_proto/src/ameide_core_proto/graph/v1/graph.proto`. Persist attestations with checksums + `produced_by_run` so that a promotion gate simply queries graph relationships rather than bespoke tables.
- Methodology overrides come from transformation context: `Transformation.stage`, `cadence`, and milestone readiness live in `packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation.proto`. When creating governance policies or workflow rules, pull those attributes to determine which gates are blocking, what expiry to use for waivers, and which dashboards (Scrum vs ADM) should surface the exception.

### Analytics, telemetry, and audit/export flows
- Telemetry plumbing already emits rich context: the workflows service wraps every call with request-scoped baggage and metrics (`services/workflows/src/common/request-context.ts:9`), records Temporal durations via histograms (`services/workflows/src/temporal/facade.ts:32`), and persists workflow status updates with tenant + execution metadata (`services/workflows/src/status-updates.ts:33`). The agents runtime exports OTLP spans/metrics as well (`services/agents_runtime/src/agents_service/telemetry.py:8`). Feed these streams into shared dashboards so lead time, change failure rate, and exception counts are derived from actual attestation/lifecycle events rather than mock data.
- Repository dashboards currently read from hard-coded fixtures (`services/www_ameide_platform/features/graph/mock-data.ts:171`). Replace those with queries that aggregate the workflow/governance telemetry above (e.g., average time between `WORKFLOW_VERSION_STATUS_PENDING_APPROVAL` → `PUBLISHED`, number of `CheckRun` failures per lane).
- Audit exports should bundle: the release manifest (images + SDKs), workflow run metadata, policy/evidence state, and helm promotion results. Extend `scripts/update_release_manifests.py` to produce a structured diff artifact and have promotion workflows attach it to the release bundle so change calendars and incident postmortems can replay the exact configuration that went live.
