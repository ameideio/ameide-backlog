# 527 Transformation — Integration Primitive Specification

**Status:** Draft (MCP adapter scaffold implemented; connectors pending)  
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

## 1.1) Implementation progress (repo snapshot)

Delivered (scaffold + tests):

- MCP adapter scaffold exists at `primitives/integration/transformation-mcp-adapter` (transport skeleton + tests + non-root container).
- Scaffold tests run: `cd primitives/integration/transformation-mcp-adapter && go test ./...`.
- Note: `ameide primitive verify` does not currently support `--kind integration` (repo guardrails are not yet enforced via CLI for Integration).

Not yet delivered (integration meaning):

- Git/CI/ticketing connectors and evidence bundle production are not implemented.
- MCP tool/resource exposure derived from proto annotations (default deny; generated schemas) is not implemented end-to-end.

## 1.2) Clarification requests (next steps)

Confirm/decide:

- Which external connectors are in the v1 acceptance slice (Git provider, CI, ticketing) and what “evidence bundle” minimally contains (logs, links, artifacts, attestations).
- Whether integration-side facts (`transformation.integration.facts.v1`) are needed vs relying solely on domain/process facts for outcomes.
- The required MCP adapter posture for v1 (which tools/resources are exposed, what auth scopes are required, and what is query-vs-command split).

## 2) Acceptance criteria

1. Integrations are idempotent and do not emit domain facts.
2. External side effects are bounded and audit-friendly (evidence bundles).
