# 523 Commerce — Agent component (optional)

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
