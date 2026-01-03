# 510 – Domain Primitive Scaffolding (Go, opinionated)

> **Status update (520/521):** This backlog specifies the Domain scaffold produced by the Ameide CLI (`ameide primitive scaffold`). The consolidated approach is a split: the CLI orchestrates scaffolding + external wiring (repo layout, GitOps), and `buf generate` + plugins handle internal deterministic generation (SDKs, generated-only glue). See `backlog/520-primitives-stack-v2.md` and `backlog/521c-internal-generation-improvements.md`.

**Status:** Active reference (aligned with 520/521)  
**Audience:** AI agents (primary), Go developers (secondary), CLI implementers  
**Scope:** Exact scaffold shape and patterns for **Domain** primitives. One opinionated pattern, aligned with `514-primitive-sdk-isolation.md` (SDK-only, self-contained primitives), no extra Domain-specific CLI parameters beyond the canonical `ameide primitive scaffold` flags.

---

## Primitive/operator alignment

- **Operator responsibilities (495, 498, 497):** The Domain operator reconciles `Domain` CRDs into infrastructure – CNPG schemas, migration Jobs, Deployments, Services, HPAs, NetworkPolicies, ServiceMonitors – and surfaces readiness via conditions. It never embeds business logic or outbox/dispatcher behavior; those are implementation details of the Domain image.  
- **Primitive responsibilities (this backlog, 496):** The Domain scaffold produces a gRPC service + EDA shape (handlers, outbox port/adapter, dispatcher, migrations) that run inside the container image referenced by the Domain CRD. It owns aggregate rules, event semantics, and SDK-based outbound calls.  
- **Boundary:** Operators are the **control plane** (CRD → K8s/external infra), primitives are the **data/behavior plane** (SDK-based gRPC service + outbox/dispatcher). 510’s scaffold is explicitly designed so operators only need an image and configuration (DB, resources, rollout); they do not depend on or reimplement Domain business logic. This alignment is codified in `495-ameide-operators.md`, `497-operator-implementation-patterns.md`, and `498-domain-operator.md`.

---

## General implementation guidelines (for CLI scaffolder)

- One fixed, opinionated scaffold per primitive kind; **no additional CLI flags** beyond the existing `ameide primitive scaffold` parameters.  
- All human‑facing instructions for the scaffold **must live in template files** (README/templates, code comment templates), not hard‑coded inline strings in the CLI implementation.  
- Scaffolded documentation (README + comments) must be **self‑contained and exhaustive** for implementers; it must **not reference external backlogs by ID** (this file and others are for design, not for runtime instructions).  

---

## Grounding & cross‑references

- **Primitive stack:** `477-primitive-stack.md` (Domain primitives in `primitives/domain/{name}` and GitOps under `gitops/primitives/domain/{name}`).  
- **Primitive/operator contract:** `495-ameide-operators.md` (shared CRD/spec/status patterns), `497-operator-implementation-patterns.md` (controller-runtime patterns), `498-domain-operator.md` (Domain operator implementation).  
- **EDA / outbox principles:** `470-ameide-vision.md` (§8–13), `472-ameide-information-application.md` (§3.3), `496-eda-principles.md`.  
- **CLI workflows & verification:** `484-ameide-cli-overview.md`, `484a-ameide-cli-primitive-workflows.md`, `484b-ameide-cli-proto-contract.md`, `484f-ameide-cli-scaffold-implementation.md`.
- **Testing discipline:** `537-primitive-testing-discipline.md` (RED→GREEN TDD pattern, CI enforcement, per-primitive test invariants).  
- **Domain operator / vertical slice:** `498-domain-operator.md`, `502-domain-vertical-slice.md`.  
- **Scrum example:** `506-scrum-vertical-v2.md`, `508-scrum-protos.md` (Transformation Scrum domain).
- **Capability implementation DAG:** `backlog/533-capability-implementation-playbook.md` (Domain scaffold + implementation nodes).

---

## 1. Canonical scaffold command (Domain / Go)

For Domain primitives we use **one opinionated pattern**:

```bash
ameide primitive scaffold \
  --kind domain \
  --name <name> \
  --include-gitops \
  --include-test-harness
```

- `--kind`, `--name`, `--include-gitops`, `--include-test-harness` are the **canonical** CLI flags for Domain scaffolds.  
- The implementation must **implicitly choose Go** as the language for Domain scaffolds (`--lang` is effectively fixed to `go` and treated as a compatibility flag only).  
- After scaffolding, the CLI should **automatically add the new module to `go.work`** (via `go work use ./primitives/domain/<name>`) so `go build ./primitives/domain/<name>/...` works without additional wiring.  
- The expected location for the scaffold is:

```text
primitives/domain/<name>/
gitops/primitives/domain/<name>/
```

---

## 2. Generated structure (Domain / Go)

The Domain scaffold emits **shape‑only** code and tests, but with a **fixed EDA pattern** baked in and a runnable gRPC entrypoint.

```text
primitives/domain/<name>/
├── README.md                            # Scaffold command, domain checklist (rendered from templates)
├── catalog-info.yaml                    # Backstage component
├── go.mod                               # Go module with ameide-sdk-go
├── Dockerfile                           # Multi-stage build (domain service)
├── cmd/
│   ├── main.go                          # Domain gRPC server entrypoint (registers service, health, reflection)
│   └── dispatcher/
│       └── main.go                      # Dispatcher entrypoint (configures poll interval/batch size, runs loop)
└── internal/
    ├── handlers/
    │   └── handlers.go                  # RPC stubs implementing the generated <Service>Server, call into domain logic + EventOutbox
    ├── ports/
    │   ├── outbox.go                    # EventOutbox interface (no Watermill/broker imports)
    │   └── outbound.go                  # Outbound ports for SDK-only cross-primitive calls
    ├── adapters/
    │   ├── postgres/
    │   │   └── outbox.go                # PostgresEventOutbox implementation
    │   └── sdk/
    │       └── clients.go               # Example SDK-backed outbound adapter stubs
    ├── dispatcher/
    │   └── dispatcher.go                # Outbox → broker dispatcher skeleton (polls DB, calls OutboxPublisher)
    └── tests/
        ├── <rpc>_test.go                # Per‑RPC failing tests (call handler)
        └── integration_mode.go          # Test harness mode switch

primitives/domain/<name>/tests/
└── run_integration_tests.sh             # 430‑aligned integration runner
```

GitOps when `--include-gitops` is used:

```text
gitops/primitives/domain/<name>/
├── values.yaml                          # Domain Deployment + dispatcher settings
├── component.yaml                       # Argo CD component descriptor
└── kustomization.yaml                   # Kustomize stub
```

---

## 3. Opinionated EDA pattern (Domain)

Domain scaffolds must always follow the **outbox → dispatcher** pattern from `496-eda-principles.md`:

- **Domain handlers**
  - Operate on aggregates and **never import broker or Watermill packages**.
  - Depend on an `EventOutbox` port and a DB handle (or transaction).
  - For each state‑changing operation they:
    - persist aggregate changes, then  
    - write a typed event into the outbox table via `EventOutbox.Insert`.
  - When calling other primitives, use Ameide SDK clients (for example `ameide-sdk-go`) from adapter layers; do **not** import other primitives’ proto packages directly for outbound calls (see `514-primitive-sdk-isolation.md`).  

- **EventOutbox port (`internal/ports/outbox.go`)**
  - Interface with methods like:

    ```go
    type EventOutbox interface {
        Insert(ctx context.Context, tx *sql.Tx, event OutboxEvent) error
    }
    ```

  - No broker, Watermill, or NATS/Kafka imports; pure port. The payload is a serialized envelope (for example protobuf bytes) so handlers can keep persistence code transport-agnostic.
  - `OutboxEvent` includes:
    - `Topic`, `Payload`
    - `Metadata` (`tenant_id`, `message_id`, correlation/causation, W3C trace context, `occurred_at`, `schema_version`, `payload_type`)
    - Aggregate linkage (`aggregate_type`, `aggregate_id`, `aggregate_version`)

- **Postgres outbox adapter (`internal/adapters/postgres/outbox.go`)**
  - Implements `EventOutbox` by writing JSON/bytes into an outbox table (see `496-eda-principles.md` for schema guidance).
  - Runs in the **same transaction** as the aggregate update.

- **Dispatcher (`internal/dispatcher/dispatcher.go`, `cmd/dispatcher/main.go`)**
  - Separate process that:
    - reads pending outbox rows in batches (e.g. via `SELECT … FOR UPDATE SKIP LOCKED`),
    - publishes them to the configured broker/topic using a `OutboxPublisher` abstraction (Watermill adapter lives here if used), and
    - marks them as published / increments retry counters.
  - This is the only place Watermill/broker clients appear; handlers must not import broker packages directly.

- **Flyway migrations (`migrations/V1__domain_outbox.sql`, plus follow-ups)**
  - Canonical inbox/outbox table schema (idempotency inbox + transactional outbox).
  - Domains are expected to ship Flyway migrations as part of the primitive (single source of truth), and the Domain operator runs them via `spec.db.migrationJobImage`.
  - Migration location contract: the migration image must contain SQL under `/flyway/sql` and must support `flyway migrate` with `FLYWAY_LOCATIONS=filesystem:/flyway/sql`.
- **Migration image (`migrations/Dockerfile.dev`, `migrations/Dockerfile.release`)**
  - A per-domain Flyway image built from the primitive’s own migrations directory.
  - Used by the Domain operator migration Job (`spec.db.migrationJobImage`).

- **Outbound SDK adapters (`internal/ports/outbound.go`, `internal/adapters/sdk/clients.go`)**
  - Provide the **only allowed path for cross-primitive calls** from a Domain.
  - `internal/ports/outbound.go` defines outbound ports; `internal/adapters/sdk/clients.go` holds Ameide SDK-backed implementations.
  - Domain handlers call outbound ports; adapters use SDK clients. Handlers never import other primitives’ proto packages directly.

Topic naming and envelope semantics follow `509-proto-naming-conventions.md` and the relevant domain proto (stable topic families with aggregator messages like `*DomainIntent` / `*DomainFact` / `*ProcessFact`).

**Progress semantics (Domain vs Process):**

- Domain facts represent **business truth** (including business-state transitions that may be rendered as “progress” in a UI).
- Domains MUST NOT emit orchestration-phase/coordinator progress noise (phase/gate/awaiting) as domain facts; that coordination truth belongs in **process facts** emitted by Process primitives.

---

## 4. Handler and test semantics (Domain / Go)

Scaffolded handlers:

- Live under `internal/handlers/handlers.go`.  
- Implement the SDK-generated `<ServiceName>Server` interface (the method set comes from proto via `buf generate`, not from bespoke CLI proto parsing).  
- Any proto-driven registration glue lives in generated-only roots (e.g., `internal/gen/**`) and is safe to delete/regenerate.  
- Use `codes.Unimplemented` placeholders only as a scaffold starting point; generated tests/harnesses and/or compile-time interfaces should force RED→GREEN when contracts evolve.  
- Provide a simple `New()` constructor; concrete primitives are expected to extend the handler to accept dependencies (DB handle, outbox, SDK-backed adapters) as needed.

Scaffolded tests:

- Live under `internal/tests/<rpc>_test.go`.  
- Are intentionally RED:
  - they construct `handlers.New(mockOutbox, ...)`,
  - call the RPC with a minimal request, and
  - `t.Fatalf("scaffold test ... replace with real assertions (RED→GREEN→REFACTOR)")`.

Implementers (humans or coding agents) are expected to:

1. Replace `codes.Unimplemented` with real domain logic + outbox calls.  
2. Replace `t.Fatalf` with real assertions (including checks that `mockOutbox` saw the right topic/event).  
3. Keep the outbox usage aligned with the EDA rules from `496-eda-principles.md`.

---

## 5. Verification expectations

`ameide primitive verify --kind domain --name <name>` is expected to enforce:

- Presence of `internal/ports/outbox.go` and `internal/adapters/postgres/outbox.go`.  
- Presence of dispatcher code under `internal/dispatcher/` and `cmd/dispatcher/`.  
- Flyway migrations exist under `migrations/` (starting at `migrations/V1__domain_outbox.sql`) and a per-domain migration image exists (`migrations/Dockerfile.*`).  
- For Scrum‑like domains, events/facts follow the proto contracts (`ScrumDomainFact`, etc.).
- An **Imports** check passes only when runtime Go code under `primitives/domain/<name>`:
  - Does **not** import `packages/ameide_core_proto` or `buf.build/gen/go/**` directly (SDK-only rule; tests may import proto types directly).  
  - Does **not** import other primitives’ modules under `github.com/ameideio/ameide/primitives/...` (cross-primitive imports must flow through SDK adapters).  
  - Ensures `internal/handlers/**` contains **no** Watermill/broker imports; broker wiring belongs in dispatcher/adapters.

In addition:

- The `EDA` check for Domain primitives should:
  - **FAIL** when outbox/dispatcher scaffolding files are missing.
  - **FAIL** when Flyway migrations / migration image are missing (domains own their DB schema).
  - **PASS** when outbox port, Postgres adapter, dispatcher, Flyway migrations, and migration image are all present.

Vertical slices (e.g., `502-domain-vertical-slice.md`, `506-scrum-vertical-v2.md`) remain the source of truth for **business semantics**; this backlog only defines the **scaffold shape and patterns** for Domain primitives.

---

## 6. Implementation progress (CLI & scaffold)

This section tracks how much of 510 is implemented in the current CLI (`packages/ameide_core_cli`) and repo scaffolds. It is descriptive, not normative; the rest of this file remains the spec.

### 6.1 Scaffolder: `ameide primitive scaffold --kind domain`

**Status:** Implemented for Domain/Go in `primitive_scaffold.go` and `templates_domain.go`.

- **Canonical command wiring**
  - Domain scaffolds are created via `runScaffold` in `primitive_scaffold.go`, with:
    - `kind = PRIMITIVE_KIND_DOMAIN`
    - `lang` defaulted to `go` if empty, per 510.
  - The CLI enforces:
    - SDK-based, Go-only scaffolds for Domains (no proto-path required).
    - `--include-gitops` and `--include-test-harness` supported.
  - `buildPrimitivePaths` places scaffolds at:
    - `primitives/domain/<name>`
    - `gitops/primitives/domain/<name>` when `--include-gitops` is set.

- **Go module and workspace**
  - Each Domain scaffold gets a `go.mod` with:
    - `module github.com/ameideio/ameide/primitives/domain/<name>`
    - `require github.com/ameideio/ameide-sdk-go` and `google.golang.org/grpc`.
  - `ensureGoWorkIncludesModule` best-effort adds `./primitives/domain/<name>` to `go.work` when scaffolding runs with `--repo-root .`.

- **README and documentation (templates)**
  - Domain/Go README content is rendered from:
    - `templates/domain/readme.md.tmpl` via `renderDomainTemplate` in `templates_domain.go`.
  - The README includes:
    - Scaffold command (with `--kind domain`, `--name`, `--include-gitops`, `--include-test-harness`).
    - Explanation of outbox/dispatcher pattern and SDK-only outbound calls.
    - “Running locally” instructions:
      - `go build ./cmd/...`
      - `PORT=8080 go run ./cmd/main.go`
      - `DISPATCHER_POLL_INTERVAL=5s DISPATCHER_BATCH_SIZE=10 go run ./cmd/dispatcher`
      - Apply Flyway migrations under `migrations/` (starting at `migrations/V1__domain_outbox.sql`).
    - Development checklist that references `ameide primitive verify`.

- **Handlers and gRPC server**
  - The CLI no longer parses proto files for Domains; scaffolds are SDK/shape only.
  - `buildHandlersContent` for Domain/Go:
    - Generates a minimal `Handler` type and `New()` constructor with no RPC methods when no SDK service is specified.
    - Leaves it to the primitive author to wire specific SDK services and RPC methods.
  - `buildMainContent` for Domain/Go:
    - Emits `cmd/main.go` that:
      - Logs a scaffold bootstrap message and instantiates `handlers.New()`.
      - Is intended to be replaced/extended once the Domain’s SDK service and gRPC wiring are known.

- **EDA scaffolding**
  - Outbox port:
    - `internal/ports/outbox.go` generated from `templates/domain/internal/ports/outbox_port.go.tmpl`.
    - Uses the signature from §3:
      - `Insert(ctx, tx, event OutboxEvent)`.
  - Postgres adapter:
    - `internal/adapters/postgres/outbox.go` generated from `templates/domain/internal/adapters/postgres/outbox_postgres.go.tmpl`.
    - Implements `EventOutbox` with an `OutboxEvent` input (topic/payload + metadata + aggregate linkage).
    - Contains TODO comments about inserting into the outbox table and storing envelope metadata.
  - Dispatcher:
    - `internal/dispatcher/dispatcher.go` generated from `templates/domain/internal/dispatcher/dispatcher.go.tmpl`.
    - Defines:
      - `OutboxPublisher` interface.
      - `Dispatcher` struct with `*sql.DB`, `OutboxPublisher`, `pollInterval`, `batchSize`.
      - `WithPollInterval` and `WithBatchSize` options.
      - `Run(ctx)` ticker loop with TODO for `SELECT … FOR UPDATE SKIP LOCKED` polling.
    - `cmd/dispatcher/main.go` (rendered from `templates/domain/cmd/dispatcher_main.go.tmpl`) wires:
      - `DISPATCHER_POLL_INTERVAL` and `DISPATCHER_BATCH_SIZE` via `envDuration`/`envInt`.
      - Instantiates `Dispatcher` with `New(db, publisher, ...)` (db/publisher still TODO).
  - Outbox migration:
    - `migrations/V1__domain_outbox.sql` generated by `buildDomainOutboxMigration`.
    - `migrations/Dockerfile.dev` and `migrations/Dockerfile.release` generated to build a per-domain Flyway migration image.
    - Defines the canonical `outbox` table with `payload`, `aggregate_*`, timestamps, `attempts`, `last_error`, and an index on `(published_at, created_at)`.

- **Outbound SDK adapters**
  - `internal/ports/outbound.go` scaffolded with an `ExampleOutboundPort` interface.
  - `internal/adapters/sdk/clients.go` scaffolded with an `ExampleClient` that documents where to add Ameide SDK-backed calls to other primitives.

- **Tests and harness**
  - Per-RPC failing tests under `internal/tests/<rpc>_test.go` call the generated handler and intentionally fail with a RED message.
  - `tests/run_integration_tests.sh` created when `--include-test-harness` is set, following the shared integration runner pattern.

### 6.2 Templates vs inline strings

**Status:** Fully template-driven for Domain/Go; README, mains, and core EDA helpers now come from templates, and guardrail tests prevent backlog IDs from leaking into generated docs.

- Domain/Go files rendered from templates:
  - README: `templates/domain/readme.md.tmpl`.
  - gRPC main: `templates/domain/cmd/main.go.tmpl`.
  - Dispatcher main: `templates/domain/cmd/dispatcher_main.go.tmpl`.
  - Outbox port: `templates/domain/internal/ports/outbox_port.go.tmpl`.
  - Postgres outbox adapter: `templates/domain/internal/adapters/postgres/outbox_postgres.go.tmpl`.
  - Dispatcher: `templates/domain/internal/dispatcher/dispatcher.go.tmpl`.
  - SDK adapters: `templates/domain/internal/adapters/sdk/clients.go.tmpl`.
- Guardrails for generated docs:
  - `templates_domain_test.go` includes `TestDomainReadmeTemplateHasNoBacklogIds`, which renders the Domain README template and fails if any `NNN-*.md` backlog references appear.
  - Agent README templates are similarly guarded by `TestAgentReadmeTemplateHasNoBacklogIds` (see 512), keeping scaffolded docs self-contained per 510/514.
- Non-Domain scaffolds:
  - Process/UISurface scaffolds still rely on the generic `buildReadmeContent` and inline builders; dedicated templates will be introduced as 511/513 are implemented.

### 6.3 Verify: `ameide primitive verify --kind domain`

**Status:** Imports + EDA checks implemented; Buf/GitOps checks enforced at repo level.

- `Imports` check:
  - Implemented in `checkPrimitiveImportPolicy`:
    - For Domains (Go), walks non-test `.go` files under `primitives/domain/<name>` and:
      - Fails when runtime code imports:
        - `packages/ameide_core_proto`.
        - `buf.build/gen/go/**` modules.
      - Detects cross-primitive imports:
        - Reads `module` from `go.mod`.
        - Flags imports under `github.com/ameideio/ameide/primitives/...` that are not the primitive’s own module or subpackages.
      - Enforces handler-level broker isolation:
        - Flags Watermill imports in `internal/handlers/**`.
    - The same `Imports` check is reused for other primitives and languages; see 514 §6.1 for TS/Python coverage.
- `EDA` check:
  - Implemented in `checkEDAReadiness(kind, serviceDir)`:
    - Applies only to Domain kind; skips other kinds with an explanatory message.
    - Requires the following files, otherwise **FAIL**:
      - `internal/ports/outbox.go`
      - `internal/adapters/postgres/outbox.go`
      - `internal/dispatcher/dispatcher.go`
      - `cmd/dispatcher/main.go`
    - Fails when Flyway migrations and/or the per-domain migration image are missing:
      - `migrations/V1__domain_outbox.sql`
      - `migrations/Dockerfile.dev`
      - `migrations/Dockerfile.release`
    - Passes when all of the above exist:
      - “EDA scaffolding (outbox + dispatcher + migration) detected”.
- Repo-wide checks that also run (independent of 510):
  - `GitOps` check (looks for central manifests under `gitops/environments/_shared/primitives/...`).
  - `BufLint` and `BufBreaking` checks for `packages/ameide_core_proto`.

### 6.4 Reference implementation: Transformation Domain

**Status:** Regenerated to match this backlog; used as the canonical example.

- Scaffold command used (from 506/508 context):
  - `ameide primitive scaffold --kind domain --name transformation --include-gitops --include-test-harness`.
  - Then run `buf generate` to refresh SDKs and any generated-only glue/tests from the Scrum protos; scaffold does not parse proto descriptors or emit generated code.
- Current state under `primitives/domain/transformation`:
  - README generated from the new Domain template with:
    - Outbox/dispatcher/migration/SDK adapter bullets.
    - Local run instructions.
    - DEV checklist matching 510.
  - `cmd/main.go`:
    - gRPC server on `:PORT`, registers `ScrumQueryService`, health, reflection.
  - EDA files:
    - `internal/ports/outbox.go` and `internal/adapters/postgres/outbox.go` with the `Insert(ctx, tx, event OutboxEvent)` outbox signature.
    - `internal/dispatcher/dispatcher.go` and `cmd/dispatcher/main.go` with the polling loop/interval/batch pattern.
    - `migrations/V1__domain_outbox.sql` with canonical inbox/outbox tables and pending index.
  - Outbound scaffolding:
    - `internal/ports/outbound.go` and `internal/adapters/sdk/clients.go` present.
- Verification snapshot (repo mode, imports + EDA only):
  - `EDA`: **PASS** (all required files + migration present).
  - `Imports`: **PASS** (no forbidden proto/buf/cross-primitive/Watermill imports in runtime code).
  - Remaining `verify` failures (BufLint, BufBreaking, GitOps manifest) are repo-level concerns, not Domain scaffold issues.

### 6.5 Known gaps and next steps

- Move remaining Domain-specific inline strings (cmd mains, SDK adapters) into templates.
- Add a short, copy-pasteable example (template or README snippet) showing how to:
  - Marshal a protobuf message into `[]byte` and populate `payload_type` / `schema_version` metadata.
  - Call `EventOutbox.Insert` from a handler while keeping handlers free of direct proto dependencies for outbox persistence.
- Consider adding a Domain-specific dev runner template (e.g., `scripts/dev_domain.sh` or a `docker-compose.yml` snippet) in a future backlog, if desired.  
