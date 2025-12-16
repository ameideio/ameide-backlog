# 523 Commerce — Agent component (optional)

**Status:** Scaffolded (read-only diagnostics surface + tests; tool design pending)

Implementation (repo): `primitives/agent/commerce-assistant`

## Layer header (Application: governed assistants)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Agent primitives), Application Services (tool calls/requests), Application Events (facts consumed for diagnostics), Data Objects.
- **Out-of-scope layers:** Strategy/Business definition (capability/value streams); Technology/runtime model hosting (see `473-ameide-technology.md`).

## Intent

Use agents as support accelerators for commerce setup and operations (read-only diagnostics + guided steps), not as mutation engines.

Primary use-cases:

- storefront domain setup assistant (DNS CNAME/TXT instructions, verification diagnostics, cert/gateway status explanation)
- store site provisioning assistant (edge bootstrap guidance, connectivity checks, sync diagnostics)
- POS operations assistant (device/register/shift issue triage and connectivity mode explanation)

## Hard rule

- Agents MUST NOT directly mutate routing/TLS, payments, replication, or store configuration; they can propose actions, not apply them.

## Stack alignment (proto / generation / operator)

- Proto: tool I/O schemas + minimal graph hints only; prompts/policy stay in code.
- Generation: typed state/tool adapters + compile/test harness for deterministic execution.
- Operator: injects model/provider configuration and secrets via Kubernetes; no secrets in proto or generated outputs.

## EDA responsibilities (496-native)

Agents follow the same EDA invariants as other primitives:

- Consume domain/process facts (at-least-once; idempotent handlers).
- Propose or emit commands/intents only through the standard Domain/Process surfaces; no side-channel state writes.
- Preserve tenant isolation and propagate correlation/causation metadata for auditability.

See `523-commerce-proto.md` for message families and envelopes.

## Implementation progress (current)

Implemented (scaffold):

- [x] Minimal agent gRPC surface that is read-only and passes guardrails (tests + non-root container).

Not yet implemented:

- [ ] Actual diagnostic tools/resources for BYOD onboarding and store health.
- [ ] MCP protocol adapter surface (if in scope) and its allowlist/guardrails.

Build-out checklist (Agent v1):

- [ ] Define the minimal read-only tool set: domain claims/mappings, cert/gateway status, projection lag/health, store-site connectivity checks.
- [ ] Define “guided steps” response format (next actions, copy-paste DNS records, safe retry suggestions) without mutating state.
- [ ] Add auditability requirements (correlation/causation propagation; tool call logs).

## Clarification requests (next steps)

Confirm/decide:

- [ ] Whether Commerce will ship a `commerce-mcp-adapter` Integration primitive in v1 (see `backlog/534-mcp-protocol-adapter.md`).
- [ ] Which read-only tools/resources the assistant must expose first (domains, cert status, gateway routes, projection lag).
