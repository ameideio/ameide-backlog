# 590 — Capabilities (repo organization + composition boundary)

**Status:** Draft  
**Audience:** Engineers (all stacks), architecture owners  
**Scope:** Establish a repo-wide **capability layer** that composes primitives/services into “vertical slices” (features) and provides a stable home for cross-primitive documentation, fixtures, and tests.

---

## Why a capability layer (and why now)

Our architecture is intentionally decomposed into **primitives** (Domain / Process / Projection / Integration / Agent / UISurface). This is good for boundary clarity and governance, but it creates a practical gap:

- “Where does an end-to-end feature live?”
- “Where do we write tests that span multiple primitives?”
- “Where do we put fixtures that represent a feature, not a component?”

**Capabilities** solve this by providing a stable **composition boundary** that is:

- closer to how users talk about the system (“agentic coding”, “transformation”, “commerce”),
- a natural owner boundary for vertical slice tests and fixtures,
- a navigable front door for feature-level implementation without undoing primitive boundaries.

Capabilities do **not** change runtime architecture. They are a **repo organization + test composition** layer.

---

## Non-goals

- This does **not** introduce a new runtime primitive.
- This does **not** move primitives out of `primitives/…`.
- This does **not** redefine ownership of primitive code.
- This does **not** replace service-level integration tests required by 430.

---

## Canonical repo shape

Capabilities live at the repo root:

```
capabilities/
  <capability>/
    README.md                      # optional: capability overview & links
    fixtures/                      # optional: seeded domain facts / repo fixtures
    __tests__/
      integration/
        run_integration_tests.sh   # 430 runner contract
        ...                        # capability-owned integration tests
```

Examples:

- `capabilities/transformation/…`

---

## Relationship to primitives and services

Capabilities are a **composition index**, not a component:

- **Primitives** remain the canonical home for reusable building blocks and invariants.
- **Services** remain the canonical home for service-owned runtime behavior and their integration suites.
- **Capabilities** own “vertical slice” flows that necessarily span multiple primitives/services.

Rule of thumb:

- If it tests or documents a single component boundary → keep it in the primitive/service.
- If it tests or documents a user-visible feature spanning multiple boundaries → put it in the capability.

---

## Ownership model

Each capability has a named owner group (or codeowner path). Capability owners:

- define vertical slice acceptance criteria,
- own capability fixtures and integration packs,
- coordinate changes across primitives/services needed to keep the feature green.

Primitive owners remain responsible for:

- correctness and stability of the primitive’s API/contracts,
- primitive-level invariants and tests,
- backward-compatible evolution where required.

---

## Cross-references

- Unified integration test contract: `backlog/430-unified-test-infrastructure.md`
- Transformation architecture + substrate seam: `backlog/527-transformation-implementation-migration.md`
- Capability test discipline and catalog: `backlog/591-capabilities-tests.md`

