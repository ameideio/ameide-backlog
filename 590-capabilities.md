# 590 — Capabilities: Repo Hierarchy + Composition Boundary

**Status:** Draft (target-state direction)  
**Audience:** Architecture, platform engineering, capability owners, test-infra  
**Scope:** Define a **repo-level “capability” folder hierarchy** that complements primitive kinds (Domain/Process/Projection/Integration/UISurface/Agent) and provides a clean home for **capability-owned tests**.

---

## 1) Problem

We already model Ameide as:

- **Capabilities** (features / business value streams) realized via
- **Primitives** (reusable runtime kinds and their operators)

But the repo is currently organized primarily by **primitive kind** (e.g. `primitives/domain/<capability>`, `primitives/process/<capability>`), which makes it hard to answer basic “capability owner” questions:

- Where is the **single place** that defines a capability’s composition (which primitive instances + services belong to it)?
- Where are the **capability’s tests** (end-to-end flows spanning primitives)?
- Where do capability fixtures / reference contracts live (without scattering them across primitive folders)?

---

## 2) Decision (normative)

Introduce a **top-level `capabilities/` directory** as the canonical “capability composition boundary”.

**Capabilities are composition + definition artifacts + tests.**  
**Primitives remain the runtime substrate.**

**Ownership rule:** capabilities own **vertical slice tests** (cross-primitive flows); primitives own **kind invariants** (per-kind correctness and operator readiness). See `backlog/591-capabilities-tests.md` and `backlog/537-primitive-testing-discipline.md`.

### 2.1 Non-goals (explicit)

This backlog does **not** propose moving:

- primitive runtime implementations (`primitives/**`)
- operator implementations (`operators/**`)
- shared libraries (`packages/**`, `sdk/**`, `tools/**`)

…under `capabilities/`.

That would be a large, high-risk repo rewrite (imports, build tooling, Helm/operator packaging, CI, ownership boundaries). Capabilities should **compose** primitives, not fork them.

---

## 3) Proposed `capabilities/` folder shape

This is a target shape; exact file names can evolve, but the intent should not.

```text
capabilities/
  <capability_name>/
    README.md                      # one-page definition + ownership + links
    capability.yaml                # machine-readable composition index (optional, recommended)
    contracts/                     # contract index + links (proto packages, topic families, services)
      contracts.md
    definitions/                   # authored “design-time” assets (not runtime code)
      archimate/
      bpmn/
      policy/
    fixtures/                      # test fixtures and seed data (capability-owned)
    __tests__/                     # capability-owned tests (see 591)
      integration/
        run_integration_tests.sh
        src/ | cmd/ | tests/
      e2e/                         # optional: browser e2e (Playwright), not required for headless flows
```

### 3.1 Naming conventions

- `capability_name` should match proto topic prefixes and primitive folder names where possible (`sales`, `transformation`, `sre`, etc.).
- Prefer **lowercase + hyphen** only if needed; avoid creating multiple names for the same capability.
- Topic/package naming rules remain governed by `backlog/509-proto-naming-conventions.md`; capability folder names should not introduce new prefixes.

---

## 4) Capability composition index (recommended)

Add a small machine-readable index per capability (example: `capabilities/sales/capability.yaml`) that lists **where the runtime components live** today.

Example intent (schema TBD):

- Domain: `primitives/domain/sales`
- Process: `primitives/process/sales`
- Projection: `primitives/projection/sales`
- Integration: `primitives/integration/sales`
- UISurface: `primitives/uisurface/sales`
- Agent: `primitives/agent/sales`
- Any capability-specific services/packages: `services/<...>`, `packages/<...>`

This gives tooling a stable entry point for:

- “run all tests for capability X”
- “verify capability X contracts”
- “generate capability X docs / diagrams”

---

## 5) Relationship to existing backlog items

- Capability semantics and primitive decomposition already exist in capability docs (example: `backlog/540-sales-capability.md`).
- Primitives stack invariants (guardrails plane, operators posture, proto discipline) are defined in `backlog/520-primitives-stack-v2.md`.
- Repo-wide test target state is defined by `backlog/430-unified-test-infrastructure.md`.
- Primitive-level testing discipline is defined by `backlog/537-primitive-testing-discipline.md`.
- “Front door” test entrypoint is `backlog/468-testing-front-door.md`.
- Capability-owned test packs are defined in `backlog/591-capabilities-tests.md`.

This backlog only adds the missing piece: **repo hierarchy for capabilities** so capability owners have a first-class home for **composition + tests** without reorganizing the primitive runtime substrate.
