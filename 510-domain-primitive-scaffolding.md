# 510 – Domain Primitive Scaffolding (Go, opinionated)

This backlog defines the **canonical target scaffold** for **Domain** primitives.

- **Audience:** AI agents (primary), Go developers (secondary), CLI implementers
- **Scope:** One opinionated pattern, aligned with `backlog/520-primitives-stack-v2.md` and `514-primitive-sdk-isolation.md` (SDK-only, self-contained primitives). The CLI orchestrates repo/GitOps wiring; `buf generate` + plugins handle deterministic generation (SDKs, generated-only glue).

> **Update (2026-01): no-brainer scaffolding + 430v2 test contract**
>
> - Scaffolding should avoid optional flags by default (no “include-*” zoo); generated primitives should include the repo-required shape (including GitOps stubs and tests) without extra switches.
> - Test scaffolding should align with `backlog/430-unified-test-infrastructure-v2-target.md` (Unit/Integration/E2E phases; native tooling; JUnit evidence; no `INTEGRATION_MODE`; no `run_integration_tests.sh` packs as the canonical path).
>
> **Update (2026-01, 670): CI-owned GitOps wiring**
>
> - GitOps wiring is authored in `ameide-gitops` via the CI-owned scaffolding workflow (workflow → PR → merge). See `backlog/670-gitops-authoritative-write-path-for-scaffolding.md`.
> - Domain scaffolding in the core repo creates runtime code + tests only; it does not directly write GitOps files as the canonical path.

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

- **Primitive stack:** `477-primitive-stack.md` (Domain primitives in `primitives/domain/{name}`; GitOps wiring is in `ameide-gitops` under `environments/_shared/components/**` + `sources/values/**`).  
- **Primitive/operator contract:** `495-ameide-operators.md` (shared CRD/spec/status patterns), `497-operator-implementation-patterns.md` (controller-runtime patterns), `498-domain-operator.md` (Domain operator implementation).  
- **EDA / outbox principles:** `470-ameide-vision.md` (§8–13), `472-ameide-information-application.md` (§3.3), `496-eda-principles-v6.md`.  
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
  --name <name>
```

- `--kind`, `--name` are the canonical inputs for Domain scaffolds.  
- The implementation must **implicitly choose Go** as the language for Domain scaffolds (`--lang` is effectively fixed to `go` and treated as a compatibility flag only).  
- After scaffolding, the CLI should **automatically add the new module to `go.work`** (via `go work use ./primitives/domain/<name>`) so `go build ./primitives/domain/<name>/...` works without additional wiring.  
- The expected location for the scaffold is:

```text
primitives/domain/<name>/
```

GitOps wiring for the Domain is created in the `ameide-gitops` repo and lands under:

```text
environments/_shared/components/apps/primitives/domain-<name>-<version>/component.yaml
sources/values/_shared/apps/domain-<name>-<version>.yaml
```

via the authoritative workflow described in `backlog/670-gitops-authoritative-write-path-for-scaffolding.md`.

---

## 2. Generated structure (Domain / Go)

The Domain scaffold emits shape-only code and tests, but with a fixed EDA pattern baked in (outbox + dispatcher + migrations) and a runnable service entrypoint. This section describes the intended target shape for Domain primitives.

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
    │   └── handlers_test.go             # Unit tests (Phase 1)
    ├── ports/
    │   ├── outbox.go                    # EventOutbox interface (no Watermill/broker imports)
    │   └── outbound.go                  # Outbound ports for SDK-only cross-primitive calls
    ├── adapters/
    │   ├── postgres/
    │   │   └── outbox.go                # PostgresEventOutbox implementation
    │   │   └── outbox_integration_test.go # Integration tests (Phase 2; `//go:build integration`)
    │   └── sdk/
    │       └── clients.go               # Example SDK-backed outbound adapter stubs
    ├── dispatcher/
    │   └── dispatcher.go                # Outbox → broker dispatcher skeleton (polls DB, calls OutboxPublisher)
    │   └── dispatcher_test.go           # Unit tests (Phase 1)
    └── ...
```

GitOps wiring is authored in the `ameide-gitops` repo via the CI-owned scaffolding workflow (670):

```text
environments/_shared/components/apps/primitives/domain-<name>-<version>/component.yaml
sources/values/_shared/apps/domain-<name>-<version>.yaml
environments/_shared/components/apps/primitives/domain-<name>-<version>-smoke/component.yaml   # optional
sources/values/_shared/apps/domain-<name>-<version>-smoke.yaml                                 # optional
```

---

## 3. Opinionated EDA pattern (Domain)

Domain scaffolds must always follow the **outbox → dispatcher** pattern from `496-eda-principles-v6.md`:

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
  - Implements `EventOutbox` by writing JSON/bytes into an outbox table (see `496-eda-principles-v6.md` for schema guidance).
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

Topic naming and semantic identity conventions follow `backlog/509-proto-naming-conventions-v6.md` and the relevant domain proto.

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
3. Keep the outbox usage aligned with the EDA rules from `496-eda-principles-v6.md`.

---

## 5. Verification expectations

Phase 0 of `ameide test` is expected to enforce:

- Presence of `internal/ports/outbox.go` and `internal/adapters/postgres/outbox.go`.  
- Presence of dispatcher code under `internal/dispatcher/` and `cmd/dispatcher/`.  
- Flyway migrations exist under `migrations/` (starting at `migrations/V1__domain_outbox.sql`) and a per-domain migration image exists (`migrations/Dockerfile.*`).  
- For Scrum‑like domains, events/facts follow the proto contracts (`ScrumDomainFact`, etc.).
- An **Imports** check passes only when runtime Go code under `primitives/domain/<name>`:
  - Does **not** import `packages/ameide_core_proto` or `buf.build/gen/go/**` directly (SDK-only rule). Unit tests may import proto types directly when needed, but Scenario Slice / capability E2E runners must call primitives via wrapper SDK clients only (per `backlog/715-v6-contract-spine-doctrine.md`).  
  - Does **not** import other primitives’ modules under `github.com/ameideio/ameide/primitives/...` (cross-primitive imports must flow through SDK adapters).  
  - Ensures `internal/handlers/**` contains **no** Watermill/broker imports; broker wiring belongs in dispatcher/adapters.

In addition:

- The `EDA` check for Domain primitives should:
  - **FAIL** when outbox/dispatcher scaffolding files are missing.
  - **FAIL** when Flyway migrations / migration image are missing (domains own their DB schema).
  - **PASS** when outbox port, Postgres adapter, dispatcher, Flyway migrations, and migration image are all present.

Vertical slices (e.g., `502-domain-vertical-slice.md`, `506-scrum-vertical-v2.md`) remain the source of truth for **business semantics**; this backlog only defines the **scaffold shape and patterns** for Domain primitives.
