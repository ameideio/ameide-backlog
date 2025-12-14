# Primitives Stack v2 (Backlog 520)

This document is the consolidated, normative version of the Domain / Process / Agent / UISurface / Projection / Integration research set. The core decision is to standardize on one architecture everywhere: **Kubernetes operators (control plane)** + **Protobuf/Buƒ codegen (behavior plane)** + **CI gates (guardrails plane)**.

## Contents

- Principles (normative)
- How To Read
- Decision Log
- Non-Goals
- Terminology
- Architecture: one stack, three planes
- Shared contracts: generation, ownership, imports, secrets
- Primitive kinds: Domain, Process, Agent, UISurface, Projection, Integration
- Conformance Checklists
- Evolution Policy
- Why this is not reinventing the wheel
- References

---

## Principles (platform constitution)

This document uses direct requirement statements; treat them as non-negotiable for the v2 stack.

1. **Proto is the behavior schema.** Protos define APIs, messages, events/facts, tools/ports (shape + minimal intent), not environment configuration or runtime policy logic.
2. **`buf generate` is canonical for internal generation.** Use `buf generate` and standard tooling for SDKs and deterministic, generated-only outputs. Plugins are invoked via Buf remote plugins (or an internal mirror of those remote plugins) with pinned versions/revisions; offline/restricted environments are a first-class requirement (see “Remote plugins: offline + supply chain” below).

   The Ameide CLI is still valuable, but its job is orchestration and guardrails: scaffolding implementation-owned files, wiring GitOps, and running standard generation/verification commands. The CLI does not replace Buf plugins; it calls them.
3. **Generation is deterministic.** Pin plugin versions (and ideally revisions), use `clean: true`, and enforce “regen-diff” in CI (run generation, fail if `git diff` is non-empty). ([Buf remote plugins][5], [Buf generate][6], [buf.gen.yaml v2][18])
4. **Generated outputs are clobber-safe.** Generators write only to generated-only dirs/files. Implementation-owned code lives outside those directories. Regeneration deletes outputs. (`clean: true` makes deletion/renames correct.) ([buf.gen.yaml v2][18])
5. **Operators are operational only.** Operators reconcile Kubernetes resources and surface status/conditions; they never interpret behavior semantics or contain language/framework logic. ([Kubernetes operator pattern][1])
6. **Runtime code imports generated SDK outputs only.** Enforce import policy so runtime code never imports the proto source tree directly (avoid “proto-in-repo” coupling).
7. **Fail early on drift.** Regeneration causes compile failures and/or failing tests until human implementations are updated—no silent runtime rot. ([gRPC Go quickstart][4])
8. **Custom options are allowed, but protos are not a behavior DSL.** Use protobuf custom options/extensions for intent and metadata; keep routing/policy/prompt/provider behavior in `_impl` code + tests. ([Proto guide][20], [Proto guide][21])
9. **Secrets stay in Kubernetes.** No secret literals in protos or generated artifacts. Operators inject secrets/config at runtime (env/volumes/secret refs, or platform-specific parameter stores). ([Kubernetes secrets][12])
10. **Implementation-owned surface stays tiny.** The generator creates almost everything; the human “escape hatch” is a small set of `_impl` files and behavior tests.

---

## How To Read

- If you are implementing or reviewing the platform: read **Principles**, then **Shared contracts**, then **Conformance checklists**.
- If you are building a new primitive implementation: read the **Primitive kind** section for your kind, then follow the **CI gate checklist** and **Operator↔runtime contract**.
- The `backlog/520-primitives-stack-v2-research-*.md` files are supporting background; this document is the normative spec.

---

## Decision Log

These are the lock-in decisions that keep the stack coherent:

- `buf generate` is the canonical generator runner for internal artifacts; the CLI orchestrates external wiring (repo layout, GitOps) and guardrails around it.
- `clean: true` + regen-diff in CI is mandatory; generation must be clobber-safe and deterministic.
- Generated outputs are written only to approved generated roots (`packages/ameide_sdk_ts/src/proto/gen/ts/**`, `packages/ameide_sdk_python/gen/python/**`, `packages/ameide_sdk_go/gen/go/**`, `primitives/**/internal/gen/**`, `build/generated/**`); never to `.`.
- Operators are operational only (Kubernetes resources + conditions); behavior/policy stays in runtime code.
- Protos are contracts, not a behavior DSL; custom options are “intent metadata”, not decision logic.
- Per-environment bindings (URLs, brokers, topic names, secrets) never live in proto; they live in CRDs/runtime config and are injected at runtime.
- Event/fact metadata is shared across primitives (one shared annotation module), not reinvented per kind.

---

## Non-Goals

This stack explicitly does not do the following:

- Define a new orchestration layer above Kubernetes operators (“operators interpreting behavior semantics”).
- Encode policy logic in `.proto` (prompts, routing rules, model/provider settings, env-specific endpoints).
- Provide a parallel “second generator pipeline” that duplicates protoc/Buƒ plugin behavior (e.g., hand-parsing protos and reimplementing descriptor resolution).
- Allow generators to write into implementation-owned directories or produce non-deterministic output.
- Hide drift until runtime (drift must surface as regen-diff, compile errors, or failing tests).

---

## Terminology

- **Fact/Event**: a published payload message; use a stable external identifier (e.g., `type`) independent of package/type refactors.
- **`stream_ref`**: a logical stream identifier in proto (e.g., `orders.facts`), bound to actual topics/subjects per environment via CRDs/runtime config.
- **`binding_ref` / `endpoint_ref`**: logical references in proto used to bind to environment-specific transport configuration (Kafka subscriptions, HTTP endpoints, etc.).
- **Approved generated root**: any directory that is safe to delete/recreate on regen and enforced by CI. In this repo, CI-enforced generator outputs land only in:
  - `packages/ameide_sdk_ts/src/proto/gen/ts/**` (TypeScript/ES SDK outputs)
  - `packages/ameide_sdk_python/gen/python/**` (Python SDK outputs)
  - `packages/ameide_sdk_go/gen/go/**` (Go SDK outputs)
  - `primitives/**/internal/gen/**` (per-primitive generated glue/tests; clobber-safe)
  - `build/generated/**` (other generated artifacts; clobber-safe)
- **SDK outputs**: generated language packages consumed by runtimes (the only way runtimes depend on proto contracts).

---

## Architecture: one stack, three planes

### Control plane (Kubernetes operators)

Each primitive kind is a Kubernetes API (CRD + controller) that reconciles a small, language-agnostic contract:

- Workloads: Deployments/Jobs/HPAs ([HPA][13])
- Networking: Services + Gateway API HTTPRoute for HTTP exposure ([HTTPRoute][2], [Gateway API][27])
- Configuration: env/envFrom, projected volumes, Secret/ConfigMap references ([Define env vars][15], [Projected volumes][16], [Kubernetes secrets][12])
- Status: conditions surfaced via standard Kubernetes conventions (`metav1.Condition`). ([metav1][38])

Operators follow standard operator/controller good practices (idempotent reconcile, watch-driven behavior). ([Kubebuilder good practices][22], [Watching resources][23])

For Temporal-backed primitives, “operator manages Temporal concerns as Kubernetes resources and surfaces conditions” matches established ecosystem patterns. ([Temporal Operator][39])

### Behavior plane (Protos + Buf plugins)

Protos are the source of truth; generation is the mechanism for keeping runtime scaffolds and SDKs aligned.

- Code generation is executed by Buf (`buf generate`) using pinned remote plugins. ([Buf remote plugins][5], [Buf generate][6])
- Code generation is executed by Buf (`buf generate`) using pinned remote plugins (or a pinned internal mirror). ([Buf remote plugins][5], [Buf generate][6])
- Generators emit:
  - SDKs (Go/TS/Py) for runtime usage
  - framework wiring skeletons (gRPC server wrappers, Temporal worker wiring, LangGraph graph/state stubs)
  - structural tests/harnesses that enforce contracts

Protobuf custom options (descriptor extensions) are the mechanism for embedding minimal intent/metadata in protos in a generator-friendly way. ([Proto guide][20], [Buf descriptors][10])

### Guardrails plane (standard CI gates)

CI is the enforcement layer:

- Buf lint + breaking change detection on proto APIs. ([Buf lint][36], [Buf breaking][37])
- Regen-diff gate: run `buf generate`, fail if git diff. (Determinism + drift detection.)
- Import policy checks to ensure runtimes depend on generated SDKs, not proto sources.
- Generated structural tests plus human-written behavior tests (unit/integration).

---

## Shared contracts

### 0) Proto package conventions (canonical in this repo)

Use the repo’s existing proto namespace and layout conventions (see `backlog/509-proto-naming-conventions.md`):

- Proto package root: `package ameide_core_proto.<bounded_context>[.<subcontext>].v<major>;`
- Proto source tree: `packages/ameide_core_proto/src/ameide_core_proto/**`

### 1) Deterministic generation contract

Minimum required:

- `buf.gen.yaml` uses v2 format and pins remote plugin versions (and optionally revisions). ([Buf remote plugins][5], [buf.gen.yaml v2][18])
- `clean: true` is used for any plugin `out` that is fully generated. ([buf.gen.yaml v2][18])
- Generation is deterministic (no timestamps/random IDs; stable ordering) and enforced by CI regen-diff.

### 1a) Option retention + source descriptors contract

Protobuf supports **option retention** (e.g., SOURCE-retained options) so “design metadata” can remain available to codegen without leaking into runtime descriptor pools or generated artifacts that do reflection.

Rules:

- **All Ameide-specific custom options are SOURCE-retained by default.** Use runtime retention only when there is a deliberate, documented runtime reflection use case.
- **Generators read Ameide options from Protobuf source descriptors** (i.e., `CodeGeneratorRequest.source_file_descriptors`) and fall back to `proto_file` only when source descriptors are unavailable.
- **Buf plugins handle this correctly** or they will silently miss options and generate incorrect output.

### 2) Ownership contract (generated vs human)

Hard rule:

- Generated code/artifacts live in generated-only directories and are overwritten on every generation run.
- Implementation-owned extension points live outside all generator outputs and are never overwritten (e.g., `*_impl.go`, `nodes_impl.py`, `policy_impl.py`). This allows safe regeneration without destroying logic.

This pattern is especially important for Agent (LangGraph) scaffolds where node logic and routing policy must remain implementation-owned. ([LangGraph application structure][14])

**Implication for `clean: true`:**

- Any directory used as a plugin `out` is generated-only and safe to delete wholesale.
- Therefore: do **not** rely on generators to “create implementation-owned files once” inside a cleaned `out` directory. Implementation-owned scaffolding is owned by the CLI orchestrator; Buf plugins only emit clobber-safe outputs (e.g., `primitives/**/internal/gen/**`).

### 2b) Tool split (Buf vs CLI vs CI vs operators)

Hard boundary:

- Everything derived from protobuf descriptors is reproducible by running `buf generate` in a clean checkout.
- Everything else (repo layout, GitOps wiring, multi-step dev loops) is CLI orchestration and/or implementation-owned.

Responsibility matrix:

| Concern / artifact | Proto | Buf plugins (`buf generate`) | CLI orchestrator | Operators |
| --- | --- | --- | --- | --- |
| API/event schemas + custom options | ✅ source of truth | reads descriptors | selects targets only | ❌ |
| SDKs (Go/TS/Py) | ❌ | ✅ generate | runs generators | ❌ |
| Generated glue inside primitives | ❌ | ✅ generate to generated roots | runs generators | ❌ |
| Implementation-owned primitive skeleton | ❌ | ❌ | ✅ create/update | ❌ |
| GitOps components / CR instances | ❌ | ❌ | ✅ create/update | reconciles CRs |
| Canonical gate | CI uses proto+buf+tests | ✅ | optional wrapper | runtime only |

**Repo layout (canonical pattern):**

- **Proto sources**: `packages/ameide_core_proto/src/ameide_core_proto/**`.
- **SDK outputs (Buf)**:
  - `packages/ameide_sdk_ts/src/proto/gen/ts/**` (TypeScript/ES)
  - `packages/ameide_sdk_python/gen/python/**` (Python)
  - `packages/ameide_sdk_go/gen/go/**` (Go)
- **Per-primitive generated glue/tests (codegen)**: `primitives/**/internal/gen/**` (pure generated; safe to delete).
- **Other generated artifacts (codegen)**: `build/generated/**` (pure generated; safe to delete).
- **Human primitive implementations**: `primitives/**` (never generated/cleaned).
- **Operators**: `operators/**` (never generated/cleaned; CRDs rendered under `operators/helm/**`).

This keeps Buf `clean: true` safe and makes the generated/human boundary uniform across Domain/Process/Agent/UISurface/Projection/Integration.

### 2a) Repository hierarchy (authoritative; where to change what)

This section is the “where does this live?” contract for all implementation stages.

- **Design/spec**: `backlog/520-primitives-stack-v2.md` (normative), `backlog/520-primitives-stack-v2-research-*.md` (background).
- **Proto sources**: `packages/ameide_core_proto/src/ameide_core_proto/**`.
- **Proto SDK outputs (Buf)**:
  - TypeScript/ES: `packages/ameide_sdk_ts/src/proto/gen/ts/**`
  - Python: `packages/ameide_sdk_python/gen/python/**`
  - Go: `packages/ameide_sdk_go/gen/go/**`
- **Primitive runtime implementations** (implementation-owned): `primitives/**` (per-kind subtrees under `primitives/agent/**`, `primitives/domain/**`, `primitives/process/**`, etc.).
- **Per-primitive generated glue/tests** (clobber-safe): `primitives/**/internal/gen/**`.
- **Other generated artifacts** (clobber-safe): `build/generated/**`.
- **Operators** (implementation-owned controllers + APIs): `operators/**` (e.g., `operators/domain-operator/**`, `operators/process-operator/**`).
- **CRDs/manifests/examples**: `operators/helm/**` and `gitops/**`.
- **Codegen and CI tooling**: `build/tools/**` and `tools/**`.

Flow (required):

1. Edit protos under `packages/ameide_core_proto/src/ameide_core_proto/**`.
2. Run `buf generate` from `packages/ameide_core_proto/` to refresh SDK outputs under `packages/ameide_sdk_ts/src/proto/gen/ts/**`, `packages/ameide_sdk_python/gen/python/**`, and `packages/ameide_sdk_go/gen/go/**`.
3. Run primitive-kind generators via `buf generate` (or the canonical codegen entrypoint) to refresh `primitives/**/internal/gen/**` (and `build/generated/**` when relevant).
4. Implement behavior in `primitives/**` and operators in `operators/**` until CI gates pass.

### 3) “Operators don’t interpret behavior” contract

Operators:

- manage lifecycle and readiness via Kubernetes resources and conditions
- inject configuration/secrets
- surface external dependency health (e.g., Temporal availability, NiFi flow sync)

Operators do not:

- parse proto descriptors
- implement workflow graphs
- implement business routing logic

### 3a) Reconcile discipline (idempotent + short)

Operator reconciliation is not a “script runner”; it is a control loop. Two non-negotiables:

- **Idempotent:** running reconcile repeatedly converges on the same desired state (safe retries; no “do it once” side effects).
- **Short/fast:** reconcile avoids long-running work so other resources are not starved and UX remains responsive.

Pattern for heavy operations (migrations, backfills, NiFi flow syncs, large installs):

- Push long operations into **Jobs** / **async work queues** / **external orchestrators**.
- Reconcile (a) creates/patches the work resource, (b) observes progress, and (c) updates `status.conditions[]` with actionable `reason`/`message`.
- Reconcile requeues from observed state changes (watch Job/Route/Deployment status) and does not use sleep/poll loops.

### 4) Shared event/fact metadata spine

Multiple primitives need consistent event/fact metadata (stable type identifier, partition key/idempotency hints, schema subject/compatibility). Standardize this once as a shared module under the repo’s proto root (e.g., `ameide_core_proto.common.v1`, implemented in `packages/ameide_core_proto/src/ameide_core_proto/common/v1/eventing.proto`) and have primitive-specific generators reuse it rather than reinventing per family.

Minimum contract for the shared module:

- **MessageOptions** annotation for event/fact messages:
  - `type` (stable external identifier; does not change when package paths refactor)
  - `schema_subject` (schema registry subject naming; set on every fact/event message)
  - `stream_ref` (logical stream identifier; environment binds to actual topic/subject names via CRDs/runtime config)
  - `partition_key_fields[]` and `idempotency_key_fields[]` (field-path hints)

Hard rule: do not place per-environment topic names/broker URLs in proto; bind those via CRDs/runtime config and inject at runtime.

**CloudEvents compatibility**

To reduce bespoke “event envelope” design and improve interoperability, align emitted facts/events with **CloudEvents**:

- Map `event_fact.type` → CloudEvents `type` (stable identifier).
- Emit a CloudEvents `id` (derived from the idempotency key or an explicit event id), `source` (producer identity), `time`, and (when relevant) `subject`.
- Carry schema pointers via CloudEvents `dataschema` and content type via `datacontenttype` where applicable.
- Keep `stream_ref` as a *logical routing identifier* (topic/subject binding) outside the CloudEvents attribute set.

**Trace context**

Propagate distributed tracing consistently by carrying W3C Trace Context (e.g., `traceparent`) alongside CloudEvents (headers/attributes depending on transport). This avoids inventing a separate tracing metadata scheme per primitive.

### 5) Operator↔runtime contract (baseline)

Every primitive runtime exposes a small, predictable “narrow waist” that operators can target across languages:

- **Health endpoints**: `GET /healthz` (liveness), `GET /readyz` (readiness), `GET /metrics` (Prometheus).
- **Ports**: operator sets `PORT_HTTP` (and `PORT_GRPC` where applicable). Runtimes do not introduce primitive-specific port env vars. Any existing primitive-specific port env vars are aliases of the standard `PORT_HTTP` / `PORT_GRPC`.
- **Identity**: operator sets a stable identifier env var (e.g., `DOMAIN_NAME`, `PROCESS_NAME`, `AGENT_ID`, `PROJECTION_NAME`) so the runtime can load the correct generated wiring.

### 6) Conditions glossary (baseline)

All primitive CRDs use `status.conditions[]` (`metav1.Condition`) and reuse a baseline vocabulary so tooling can reason across kinds:

- **Schema:** use `metav1.Condition` fields as intended:
  - set `.status.observedGeneration` to the primitive CR’s `.metadata.generation` on every status update
  - set `observedGeneration` to the primitive CR’s `.metadata.generation` when updating the condition (prevents “stale status” ambiguity)
  - set `reason` and `message` with human-actionable troubleshooting detail
  - update `lastTransitionTime` only when `status` changes
- **Polarity:** follow Kubernetes/Gateway API “positive polarity” conventions:
  - condition types name the *happy-path achieved state* (`SecretsReady`, not `SecretsMissing`)
  - `status=True` means achieved; `status=False` means not achieved; avoid double-negatives
  - Use specific `reason` values (e.g., `SecretMissing`, `DialFailed`, `InvalidSpec`) over creating ad-hoc negative condition types

- `SecretsReady`: all referenced Secrets/ConfigMaps exist and are mounted/rendered.
- `DependenciesReady`: external dependencies are reachable (DB/broker/Temporal/NiFi/Registry/etc.).
- `WorkloadReady`: Deployments/Jobs are running/Available as expected.
- `RouteReady`: HTTPRoute is accepted/programmed/resolved by the Gateway implementation.
- `Ready`: umbrella condition (true when all required conditions are true for the primitive kind).

**Gateway API alignment (when exposing HTTPRoute):**

- When a primitive exposes an HTTPRoute, map `RouteReady` directly from the Route’s own conditions (positive polarity) where available: `Accepted`, `ResolvedRefs`, `Programmed`. Operators surface the underlying Gateway API `reason` and `message` in the primitive’s `RouteReady`.

Primitive-specific conditions (examples): `TemporalReady`, `NamespaceReady`, `OutboxReady`, `FlowSynced`, `StoreReady`, `MigrationSucceeded`, `IngestionHealthy`, `BackpressureOK`.

### 7) CI gate checklist (minimal)

Make the inner loop and CI gates identical across primitives:

- `buf lint` + `buf breaking` on proto modules.
- `buf generate` with pinned plugins + `clean: true` for generated-only dirs.
- Regen-diff: `git diff --exit-code` must be clean after generation.
- Import policy: runtime code imports only generated SDK outputs (no proto source-tree imports).
- Tests: generated structural tests always run; human-written behavior tests run at least per primitive kind.

### 8) Upgrade playbook (generation + schema evolution)

Treat plugin and proto evolution as a first-class operational workflow:

- **Plugin upgrades**: change plugin version/revision in one PR, run `buf generate`, commit the full regen diff, and keep the regen-diff CI gate strict.
- **Generated/human boundary**: generators only write under generated roots (`packages/ameide_sdk_ts/src/proto/gen/ts/**`, `packages/ameide_sdk_python/gen/python/**`, `packages/ameide_sdk_go/gen/go/**`, `primitives/**/internal/gen/**`, `build/generated/**`); never write to `.` or mixed human directories when `clean: true` is enabled.
- **Schema evolution**:
  - additive proto changes are default
  - breaking changes require a major version bump (`…v2`) and/or parallel message variants for replay-safe systems (especially Temporal/Process)
  - stable event/fact `type` identifiers come from the shared metadata spine, not from package/type names.
  - runtimes tolerate mixed-version payloads (unknown fields, missing fields, and new `oneof` variants) without crashing or wedging ingestion.

### 9) Plugin implementation discipline (avoid subtle protoc/Buƒ bugs)

Primitive-kind generators are “supply-chain critical”: they sit between the source-of-truth protos and your runtime artifacts. Treat generator correctness and determinism as first-class.

Minimum required:

- **Use a well-maintained protoc plugin harness** (e.g., Buf’s `bufbuild/protoplugin` or an equivalent) to avoid common response-format and feature-negotiation bugs.
- **Hermetic generators only**: generators are pure functions of `CodeGeneratorRequest`; they do not read the repo filesystem, other plugin outputs, or the network, and they do not depend on environment variables or clocks (no timestamps).
- **Modern plugin handshake**: generators set `CodeGeneratorResponse.supported_features` correctly and support Protobuf Editions and `proto3 optional` without requiring generator changes. ([Protobuf Editions][46])
- **Golden tests on deterministic inputs**: run the generator against a fixed `FileDescriptorSet` / `CodeGeneratorRequest` fixture and assert byte-for-byte stable outputs (no timestamps, stable ordering, stable formatting).
- **Deterministic emission rules**: sort all emitted files, symbols, tables, fields, and edges deterministically; never rely on map iteration order.
- **Retention-aware parsing**: read SOURCE-retained options from `CodeGeneratorRequest.source_file_descriptors` (see 1a).
- **Descriptor-first symbol resolution**: never parse `.proto` text for type/service/method resolution; use `CodeGeneratorRequest` descriptors and resolve symbols across imports (e.g., return types like `google.protobuf.Empty`) so generators don’t break on cross-file dependencies.
- **Import graph completeness**: generators resolve and validate the full transitive import graph (including WKTs like `google.protobuf.Empty`) for every generated output.

**Scaffold stance (per primitive kind)**

Proto-driven “service skeleton” generators tend to fail when they try to be “too helpful.” Decide up front which of these you want:

- **Compile-clean but unimplemented**: generated scaffolds compile, but default behavior returns “unimplemented” or placeholder results until humans implement extension points.
- **Won’t compile until wired** (default for Ameide runtimes): generated scaffolds intentionally require explicit wiring/implementation so drift and missing dependencies fail at compile time or CI time, not at runtime.

Default stance:

- **Domain/Process/Projection**: “won’t compile until wired” (forces handler/workflow impl + required config boundaries).
- **Agent**: compile-clean scaffold + generated tests that fail until node/policy implementations exist.
- **Integration**: compile-clean artifact generation, but structural tests fail if required ports/params/artifacts are missing or inconsistent.

### 10) Remote plugins: offline + supply chain

Remote plugins are a good default for consistency and pinning, but they introduce two real operational risks:

1) **Offline/restricted networks**: remote plugin downloads can fail when network access is unavailable.
2) **Supply chain hygiene**: if a large fraction of the repo is generated, generator provenance and pinning become part of the project’s integrity story.

**Guardrails (required)**

- **Never generate into `.` or any implementation-owned directory.** With `clean: true`, Buf deletes the plugin `out` directory before writing; this is only safe when `out` is generated-only.
- **CI validates generator outputs are safe to delete.** Reject any generation template executed in CI where a plugin `out` is `.` or outside approved generated roots (`packages/ameide_sdk_ts/src/proto/gen/ts/**`, `packages/ameide_sdk_python/gen/python/**`, `packages/ameide_sdk_go/gen/go/**`, `primitives/**/internal/gen/**`, `build/generated/**`).

**Offline strategy: mirror + pre-warm**

- Mirror remote plugins into an internal registry/artifact store and reference the mirrored location (still pinned by version/revision).
- Bake CI images with a pre-populated Buf cache for the pinned plugins/modules so `buf generate` is offline-safe in CI.
- CI and dev inner loops do not download plugins from the public BSR.

**Supply chain / provenance**

- Treat the generation step as a build: generator identity (plugin name + version + revision or digest), inputs (proto module + lockfiles), and outputs (generated file roots) are part of the reviewable change.
- CI emits and stores provenance metadata (SLSA vocabulary) for generation runs.
- Keep regen-diff strict: if provenance or plugin pins change, review and commit the full regen diff in the same PR.

---

## Primitive kinds (shape + responsibilities)

Each primitive kind follows the same three planes; what varies is the behavior-plane target and the runtime conventions.

### Domain

**Intent:** transactional domain services (RPC APIs + domain facts/events).

- **Proto:** domain services + fact/event message schemas; custom options for domain intent and routing metadata.
- **Generation:** gRPC stubs plus server skeletons that force compile-time drift detection; an HTTP plane for `/healthz`, `/readyz`, `/metrics`; and an outbox implementation whenever the proto defines any facts/events for the domain.
- **Operator:** reconciles Deployment/Service/HTTPRoute, runs migrations as Jobs, injects DB/broker secrets and runtime env, surfaces readiness/EDA/outbox status.

### Process

**Intent:** long-lived workflow/process orchestration (Temporal-backed), driven by facts/commands.

- **Proto:** process envelope + typed payloads; explicit versioning markers; options that declare workflow identity and ingress bindings.
- **Generation:** Temporal worker wiring + routing skeletons; structural tests to enforce determinism constraints and versioning contracts.
- **Operator:** reconciles worker Deployments/HPAs and ingress; treats Temporal as an external dependency and reports conditions rather than hiding failures behind crashing pods.

Temporal namespaces are a first-class isolation boundary; treat namespace configuration/availability as an external dependency surfaced via conditions rather than “hidden” in app crash loops. ([Temporal namespaces][40])

### Agent

**Intent:** agent runtime (LangGraph/LangChain) with typed state + typed tool contracts; policy remains in code.

- **Proto:** agent API envelopes, tool I/O schemas, state schema; minimal graph hints (entry node, required tools); field-level reducer hints when needed.
- **Generation:** Pydantic/state models, LangGraph skeleton, tool adapters, and tests (graph compiles, required tools registered, deterministic test harness). ([LangGraph Graph API][9], [LangGraph persistence][7], [LangGraph testing][8], [LangChain tools][11], [Tool calling blog][17])
- **Operator:** injects model/provider configuration and secrets via Kubernetes Secrets/ConfigMaps; never stores prompts/secrets in proto/generated outputs. ([Kubernetes secrets][12])

### UISurface

**Intent:** web UI surfaces as first-class primitives, generated from proto-defined interfaces and deployed via operator-managed workloads/routing.

- **Proto:** UI API contracts (RPC/HTTP shapes) plus typed view models as needed.
- **Generation:** SDKs and framework skeletons (client wiring, schema-driven forms/components where applicable).
- **Operator:** reconciles UI workload, HTTPRoute exposure, env injection, and readiness.

### Projection

**Intent:** read-optimized materialized views / analytical query services built from transactional data and/or domain facts/events.

- **Proto:** query APIs + projected schema declarations; input bindings are required and are declared as either fact/event consumption or CDC.
- **Generation:** SDKs, ingestion stubs, schema/migration outputs, and harness/tests.
- **Operator:** reconciles workloads plus backing stores/references, migrations/backfills, HTTPRoute exposure, and status conditions. Prefer idempotent, watch-driven reconcile patterns. ([Kubebuilder good practices][22], [controller-runtime reconcile][24])

Projection storage options are deliberately pluggable:

- Postgres materialized views (explicit refresh semantics). ([CREATE MATERIALIZED VIEW][28], [REFRESH MATERIALIZED VIEW][29])
- Default store: event-driven projection tables in Postgres.
- Columnstores like ClickHouse when required by workload. ([ClickHouse MergeTree][30], [ClickHouse CREATE TABLE][31], [ClickHouse TTL][32])
- CDC from transactional DBs when events are not available (logical replication/decoding). ([Postgres subscription][33], [Logical decoding][34])
- Outbox-based event routing remains a core pattern for reliable event emission. ([Debezium outbox router][35])

### Integration

**Intent:** “flows as code” integrations (defaulting to NiFi) with strong day-2 lifecycle management.

- **Proto:** ports/contracts declared as annotated service methods; message schemas describe what is consumed/emitted; parameters remain placeholders.
- **Generation:** flow artifacts (registry versioned flow snapshots), parameter context templates (placeholders only), descriptor sets/schemas, structural tests/harness.
- **Operator:** reuse an upstream NiFi control plane (NiFiKop) for NiFi cluster/registry/dataflow/parameter-context reconciliation, and implement the Integration primitive as a thin adapter that translates proto-defined intent into those CRDs; surface conditions based on NiFi health signals (status, bulletins, queues/backpressure).

---

## Conformance Checklists

### Global (all primitive kinds)

- Protos live under `packages/ameide_core_proto/src/ameide_core_proto/**` and follow `backlog/509-proto-naming-conventions.md`.
- All generators are invoked via `buf generate` with pinned versions/revisions and `clean: true` for generated roots.
- Every plugin `out` directory is generated-only and safe to delete; no implementation-owned files live under any cleaned `out`.
- CI validates `buf.gen.yaml` plugin `out` paths (reject `.` and any path outside approved generated roots).
- CI enforces regen-diff (`git diff --exit-code` clean after generation).
- Ameide custom options are SOURCE-retained by default; generators read options from `CodeGeneratorRequest.source_file_descriptors` (fallback to `proto_file` only when needed).
- Primitive-kind generators use a standard plugin harness (e.g., `bufbuild/protoplugin`) and have golden tests that assert byte-for-byte deterministic outputs.
- Primitive-kind generators are hermetic: no filesystem reads (outside the request), no network, no timestamps, and no dependence on other plugin outputs.
- Primitive-kind generators implement the modern plugin handshake and support Protobuf Editions and `proto3 optional`.
- Offline strategy is implemented: mirror remote plugins internally and pre-warm caches; CI and dev inner loops do not download plugins from the public BSR.
- Runtimes import only generated SDK outputs (no proto source-tree imports).
- Reconcile loops are idempotent and short; long-running work is delegated to Jobs/async workflows with progress surfaced via conditions.
- Operators set `.status.observedGeneration` on every status update and set `status.conditions[].observedGeneration` for every condition update; condition types follow positive-polarity naming aligned with Gateway API; baseline types are `SecretsReady`, `DependenciesReady`, `WorkloadReady`, `RouteReady` (if exposed), and `Ready`.
- Observability uses OpenTelemetry conventions: generated runtimes propagate trace context across RPC/HTTP/event boundaries and follow OpenTelemetry semantic conventions for span/metric naming to avoid bespoke “metric name wars”.
- Facts/events emitted across system boundaries are CloudEvents-compatible, with `event_fact.type` mapped to CloudEvents `type`.
- CI emits generation provenance metadata (SLSA vocabulary) as a build artifact when generation outputs are part of the repo.

### Domain

- Emits facts/events annotated with the shared `(ameide_core_proto.common.v1.event_fact)` option.
- Has an outbox story (if event-driven) that is testable and drift-checked.
- Any generated gRPC-Go server stub embeds `Unimplemented<Service>Server` **by value**, not pointer, and includes a unit test that asserts a zero-value server does not panic when an unimplemented RPC is called (guards the known nil-embed foot-gun).
- Generated stubs preserve compile/CI drift detection: do not rely on `Unimplemented<Service>Server` to “cover missing methods”; generate explicit per-RPC stubs and/or a required handler interface so missing wiring fails early.

### Process

- Workflow inputs are replay-safe: start args versioned explicitly; breaking payload changes handled via parallel variants.
- Generated guardrails enforce determinism constraints and ban non-deterministic imports/calls in workflow impl.
- Generated wiring uses shared constants/config for task queue/workflow type to avoid silent stalls from name mismatches, and always sets `WorkflowIDReusePolicy` explicitly for Signal-With-Start (do not rely on SDK defaults).
- Generated code does not assume legacy/experimental Temporal Worker Versioning methods (vendor docs warn pre-2025 experimental support is removed from Temporal Server March 2026); generated code uses replay-safe code versioning patterns and stable worker deployment/versioning approaches.

### Agent

- Proto defines only tool I/O + state schema + minimal graph hints; prompts/policy remain in `_impl` code.
- Persistence/state discipline is enforced: requests provide a stable `thread_id` as a first-class request field; persisted state stays small (IDs/cursors/keys), and large artifacts are stored out-of-band with references in state; tool outputs and structured responses are runtime-validated against generated schemas.

### Projection

- Schema/migrations are generator-owned under generated roots; backfill/rebuild is explicitly supported and observable.
- Processing semantics are explicit (at-least-once vs exactly-once). Generated ingestion includes an offset/checkpoint store schema and handler skeletons that enforce idempotency keys; exactly-once requires “offset+side effects” committed in the same transaction when using a transactional sink.

### Integration

- Generated artifacts contain placeholders only; sensitive values are injected at runtime (e.g., via NiFi Parameter Context values).

---

## Evolution Policy

- **Proto evolution**:
  - Additive changes are the default.
  - When you cannot evolve additively, bump major version (`…v2`) and keep v1 side-by-side until consumers migrate.
  - Do not treat package/type names as stable external identifiers for events; use the shared `event_fact.type`.
- **Plugin evolution**:
  - Treat each primitive-kind plugin as a runtime API surface with strict semver.
  - Any change that alters generated file layout or runtime contract requires a semver major bump.
- **Operator API evolution**:
  - CRD schema changes that break existing manifests require a new version (or a conversion strategy) and explicit migration guidance.

---

## Plugin strategy: monolithic per primitive-kind per runtime target

Default strategy:

- One “monolithic” plugin per primitive kind per runtime target language (e.g., `domain-go`, `process-go`, `agent-py`, `projection-go`, `integration-nifi`, `uisurface-web`).

Reasoning:

- fewer pinned versions/revisions to manage
- cross-cutting conventions stay coherent (env vars, health, routing, tests)
- one authoritative runtime contract per family (better drift control)

Buf supports remote plugins and deterministic generation; keep plugins pure/deterministic and driven only by the code generation request. ([Buf remote plugins][5], [Buf custom plugins][19])

---

## Why this isn’t reinventing the wheel

This stack reuses existing “wheels” at the correct layer:

- **Operators:** standard Kubernetes operator/controller pattern. ([Kubernetes operator pattern][1])
- **Routing:** Gateway API / HTTPRoute as the standard L7 ingress API. ([Gateway API][27], [HTTPRoute][2])
- **Codegen:** standard Protobuf compiler plugin mechanism + Buf remote plugins. ([Buf generate][6], [Buf remote plugins][5], [Protocol Buffers][20])
- **Service scaffolds:** same model as ConnectRPC (proto → stubs, you implement) without a framework-specific CLI. ([ConnectRPC Go][25])
- **Service scaffold CLIs:** frameworks like go-zero, Kratos, and Kitex ship proto-driven scaffolding generators; this stack keeps the runner standardized on Buf/protoc plugins and makes the “primitive runtime contract” the output. ([go-zero proto scaffolding][41], [Kratos proto generators][44], [Kitex code generation][45])
- **Control planes:** aligns with Crossplane and Knative CRD/controller patterns. ([Crossplane][3], [Knative Serving][26])

The common industry traps (and how this stack avoids them):

- **Don’t parse `.proto` text**: use descriptor-driven plugins so imported types resolve correctly (e.g., `google.protobuf.Empty`) and SOURCE-retained options are not missed.
- **Don’t mix generated + human dirs**: `clean: true` deletes plugin `out`; only generate into generated-only roots and enforce this in CI.
- **Choose your scaffold stance**: either compile-clean-but-unimplemented or intentionally-not-compiling-until-wired; encode this per primitive kind so teams don’t discover it by accident.

You start reinventing the wheel if you:

- grow a bespoke CLI that replicates proto parsing + templating + plugin orchestration
- make operators interpret behavior/policy semantics
- allow generated/human boundaries to blur so regen becomes unsafe

---

## References

[1]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/ "Operator pattern | Kubernetes"
[2]: https://gateway-api.sigs.k8s.io/api-types/httproute/ "HTTPRoute - Kubernetes Gateway API"
[3]: https://docs.crossplane.io/latest/whats-crossplane/ "What's Crossplane?"
[4]: https://grpc.io/docs/languages/go/quickstart/ "Quick start | Go | gRPC"
[5]: https://buf.build/docs/bsr/remote-plugins/ "Remote plugins - Buf Docs"
[6]: https://buf.build/docs/generate/ "Code Generation - Buf Docs"
[7]: https://docs.langchain.com/oss/python/langgraph/persistence "Persistence - Docs by LangChain"
[8]: https://docs.langchain.com/oss/python/langgraph/test "Test - Docs by LangChain"
[9]: https://docs.langchain.com/oss/python/langgraph/graph-api "Graph API overview - Docs by LangChain"
[10]: https://buf.build/docs/reference/descriptors/ "Descriptors - Buf Docs"
[11]: https://reference.langchain.com/python/langchain/tools/ "Tools | LangChain Reference"
[12]: https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/ "Distribute Credentials Securely Using Secrets | Kubernetes"
[13]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/ "Horizontal Pod Autoscaling | Kubernetes"
[14]: https://docs.langchain.com/oss/python/langgraph/application-structure "Application structure - Docs by LangChain"
[15]: https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/ "Define Environment Variables for a Container | Kubernetes"
[16]: https://kubernetes.io/docs/concepts/storage/projected-volumes/ "Projected Volumes | Kubernetes"
[17]: https://blog.langchain.com/tool-calling-with-langchain/ "Tool Calling with LangChain"
[18]: https://buf.build/docs/configuration/v2/buf-gen-yaml/ "buf.gen.yaml - Buf Docs"
[19]: https://buf.build/docs/bsr/remote-plugins/custom-plugins/ "Custom plugins - Buf Docs"
[20]: https://protobuf.dev/programming-guides/proto2/ "Language Guide (proto2) | Protocol Buffers Documentation"
[21]: https://protobuf.dev/programming-guides/proto3/ "Language Guide (proto3) | Protocol Buffers Documentation"
[22]: https://book.kubebuilder.io/reference/good-practices "Good Practices - Kubebuilder Book"
[23]: https://book.kubebuilder.io/reference/watching-resources "Watching Resources - Kubebuilder Book"
[24]: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile "reconcile package - sigs.k8s.io/controller-runtime"
[25]: https://buf.build/connectrpc/go "connectrpc/go"
[26]: https://knative.dev/docs/serving/ "Knative Serving"
[27]: https://kubernetes.io/docs/concepts/services-networking/gateway/ "Gateway API | Kubernetes"
[28]: https://www.postgresql.org/docs/current/sql-creatematerializedview.html "PostgreSQL: CREATE MATERIALIZED VIEW"
[29]: https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html "PostgreSQL: REFRESH MATERIALIZED VIEW"
[30]: https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree "MergeTree table engine | ClickHouse Docs"
[31]: https://clickhouse.com/docs/sql-reference/statements/create/table "CREATE TABLE | ClickHouse Docs"
[32]: https://clickhouse.com/docs/guides/developer/ttl "Manage Data with TTL (Time-to-live) | ClickHouse Docs"
[33]: https://www.postgresql.org/docs/current/logical-replication-subscription.html "PostgreSQL: Subscription (logical replication)"
[34]: https://www.postgresql.org/docs/current/logicaldecoding-explanation.html "PostgreSQL: Logical Decoding Concepts"
[35]: https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html "Debezium Outbox Event Router"
[36]: https://buf.build/docs/lint/ "Linting and code standards - Buf Docs"
[37]: https://buf.build/docs/breaking/ "Breaking change detection - Buf Docs"
[38]: https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1 "v1 package - k8s.io/apimachinery/pkg/apis/meta/v1"
[39]: https://temporal-operator.pages.dev/ "Temporal Operator: Overview"
[40]: https://docs.temporal.io/namespaces "Temporal Namespace | Temporal Platform Documentation"
[41]: https://go-zero.dev/en/docs/tutorials/cli/rpc "goctl rpc | go-zero Documentation"
[42]: https://github.com/cloudevents/spec "CloudEvents specification (CNCF)"
[43]: https://opentelemetry.io/docs/specs/semconv/ "OpenTelemetry semantic conventions"
[44]: https://go-kratos.dev/docs/getting-started/usage/ "Kratos: code generation / proto-first workflow"
[45]: https://www.cloudwego.io/docs/kitex/tutorials/code-gen/code_generation/ "Kitex code generation"
[46]: https://protobuf.dev/editions/implementation/ "Implementing Editions Support"
