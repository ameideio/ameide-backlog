# 527 Transformation — Integration Primitive Specification

**Status:** Draft  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Integration primitives** — adapters for external systems (git/CI/scaffolding/ticketing) used to realize and govern transformation initiatives.

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Integration), Application Services/Interfaces (external ports), Data Objects (integration payloads).
- **Out-of-scope layers:** Canonical domain state and portal UX.

## 1) Integration responsibilities

- External connectors:
  - Backstage scaffolder / scaffolding runners,
  - GitHub/GitLab, CI systems, container registries,
  - ticketing/work item systems where required.
- Strict idempotency for inbound webhooks and retries.
- Emit intents/commands into domains (never facts); domains remain the only fact emitters.

## 2) Acceptance criteria

1. Integrations are idempotent and do not emit domain facts.
2. External side effects are bounded and audit-friendly (evidence bundles).

