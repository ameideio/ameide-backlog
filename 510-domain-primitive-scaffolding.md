# 510 – Domain Primitive Scaffolding (Go, opinionated)

**Status:** Draft  
**Audience:** AI agents (primary), Go developers (secondary), CLI implementers  
**Scope:** Exact scaffold shape and patterns for **Domain** primitives. One opinionated pattern, aligned with `514-primitive-sdk-isolation.md` (SDK-only, self-contained primitives), no extra Domain-specific CLI parameters beyond the canonical `ameide primitive scaffold` flags.

---

## General implementation guidelines (for CLI scaffolder)

- One fixed, opinionated scaffold per primitive kind; **no additional CLI flags** beyond the existing `ameide primitive scaffold` parameters.  
- All human‑facing instructions for the scaffold **must live in template files** (README/templates, code comment templates), not hard‑coded inline strings in the CLI implementation.  
- Scaffolded documentation (README + comments) must be **self‑contained and exhaustive** for implementers; it must **not reference external backlogs by ID** (this file and others are for design, not for runtime instructions).  

---

## Grounding & cross‑references

- **Primitive stack:** `477-primitive-stack.md` (Domain primitives in `primitives/domain/{name}` and GitOps under `gitops/primitives/domain/{name}`).  
- **EDA / outbox principles:** `470-ameide-vision.md` (§8–13), `472-ameide-information-application.md` (§3.3), `496-eda-principles.md`.  
- **CLI workflows & verification:** `484-ameide-cli-overview.md`, `484a-ameide-cli-primitive-workflows.md`, `484b-ameide-cli-proto-contract.md`, `484f-ameide-cli-scaffold-implementation.md`.  
- **Domain operator / vertical slice:** `498-domain-operator.md`, `502-domain-vertical-slice.md`.  
- **Scrum example:** `506-scrum-vertical-v2.md`, `508-scrum-protos.md` (Transformation Scrum domain).

---

## 1. Canonical scaffold command (Domain / Go)

For Domain primitives we use **one opinionated pattern**:

```bash
ameide primitive scaffold \
  --kind domain \
  --name <name> \
  --proto-path <path/to/service.proto> \
  --include-gitops \
  --include-test-harness
```

- `--kind`, `--name`, `--proto-path`, `--include-gitops`, `--include-test-harness` are the **canonical** CLI flags for Domain scaffolds.  
- The implementation must **implicitly choose Go** as the language for Domain scaffolds (`--lang` is effectively fixed to `go` and treated as a compatibility flag only).  
- After scaffolding, the CLI should **automatically add the new module to `go.work`** (via `go work use ./primitives/domain/<name>`) so `go build ./primitives/domain/<name>/...` works without additional wiring.  
- The expected location for the scaffold is:

```text
primitives/domain/<name>/
gitops/primitives/domain/<name>/
```

---

## 2. Generated structure (Domain / Go)

The Domain scaffold emits **shape‑only** code and tests, but with a **fixed EDA pattern** baked in.

```text
primitives/domain/<name>/
├── README.md                            # Scaffold command, domain checklist
├── catalog-info.yaml                    # Backstage component
├── go.mod                               # Go module with ameide-sdk-go
├── Dockerfile                           # Multi-stage build (domain service)
├── cmd/
│   └── main.go                          # Domain service entrypoint (injects handler/outbox)
└── internal/
    ├── handlers/
    │   └── handlers.go                  # RPC stubs, call into domain logic + EventOutbox
    ├── ports/
    │   └── outbox.go                    # EventOutbox interface (no Watermill/broker imports)
    ├── adapters/
    │   └── postgres/
    │       └── outbox.go                # PostgresEventOutbox implementation
    ├── dispatcher/
    │   └── dispatcher.go                # Outbox → broker dispatcher skeleton (Watermill)
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
        Insert(ctx context.Context, tx *sql.Tx, topic string, payload proto.Message,
            aggregateType, aggregateID string, version int64) error
    }
    ```

  - No broker, Watermill, or NATS/Kafka imports; pure port.

- **Postgres outbox adapter (`internal/adapters/postgres/outbox.go`)**
  - Implements `EventOutbox` by writing JSON/bytes into an outbox table (see `496-eda-principles.md` for schema guidance).
  - Runs in the **same transaction** as the aggregate update.

- **Dispatcher (`internal/dispatcher/dispatcher.go`, `cmd/dispatcher/main.go`)**
  - Separate process that:
    - reads pending outbox rows,
    - publishes them to the configured broker/topic using Watermill, and
    - marks them as published / increments retry counters.
  - This is the only place Watermill/broker clients appear.

Topic naming and envelope semantics follow `509-proto-naming-conventions.md` and the relevant domain proto (`events/v1` or `*DomainFact` packages).

---

## 4. Handler and test semantics (Domain / Go)

Scaffolded handlers:

- Live under `internal/handlers/handlers.go`.  
- Are generated from the proto service: one method per RPC, returning `codes.Unimplemented`.  
- Provide a simple `New()` constructor; concrete primitives are expected to extend the handler to accept dependencies (DB handle, outbox, SDK-backed adapters) as needed.

Scaffolded tests:

- Live under `internal/tests/<rpc>_test.go`.  
- Are intentionally RED:
  - they construct `handlers.New(mockOutbox, ...)`,
  - call the RPC with a minimal request, and
  - `t.Fatalf("scaffold test ... replace with real assertions (RED→GREEN→REFACTOR)")`.

Agents/humans are expected to:

1. Replace `codes.Unimplemented` with real domain logic + outbox calls.  
2. Replace `t.Fatalf` with real assertions (including checks that `mockOutbox` saw the right topic/event).  
3. Keep the outbox usage aligned with the EDA rules from `496-eda-principles.md`.

---

## 5. Verification expectations

`ameide primitive verify --kind domain --name <name>` is expected to enforce:

- Presence of `internal/ports/outbox.go` and `internal/adapters/postgres/outbox.go`.  
- No broker/Watermill imports in `internal/handlers/**`.  
- Dispatcher present under `internal/dispatcher/` / `cmd/dispatcher/`.  
- Outbox table schema exists in the Domain’s DB (or is referenced via migrations).  
- For Scrum‑like domains, events/facts follow the proto contracts (`ScrumDomainFact`, etc.).
- An **Imports** check passes only when runtime Go code under `primitives/domain/<name>` does **not** import `packages/ameide_core_proto` or `buf.build/gen/go/**` directly; outbound calls must go through SDK clients (tests are allowed to import proto types directly).

Vertical slices (e.g., `502-domain-vertical-slice.md`, `506-scrum-vertical-v2.md`) remain the source of truth for **business semantics**; this backlog only defines the **scaffold shape and patterns** for Domain primitives.
