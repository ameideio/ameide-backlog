> Note: Chart and values paths are now under `gitops/ameide-gitops/sources` (charts/values); any `infra/kubernetes/charts` references in child docs are historical.

# backlog/367 – Elements + Transformation + Automation (Parent)

**Status:** Parent backlog (navigation + invariants)  
**Audience:** Platform, Transformation, Process, Agents  
**Scope:** Defines how Stage 0 intake, Stage 1 methodology profiles (Scrum/SAFe/TOGAF), Stage 2 Process orchestration, and Stage 3 agent execution compose into a single, event-driven vertical.

This file exists because multiple 367-* backlogs reference it as the parent; it is intentionally short and points to the canonical contracts.

## Canonical contracts (no exceptions)

- **Scrum profile (Stage 1):** `backlog/300-400/367-1-scrum-transformation.md` (methodology semantics + UX) and the repo proto sources under `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/**`.
- **Scrum runtime seam (Stage 2):** `backlog/506-scrum-vertical-v2.md` (topics, envelopes, Temporal workflow structure) and `packages/ameide_core_proto/src/ameide_core_proto/process/scrum/v1/**` (process facts).
- **EDA invariants:** `backlog/496-eda-principles.md` (outbox, idempotency, commands vs facts).
- **Agent execution (Stage 3):** `backlog/505-agent-developer-v2.md` and `backlog/505-agent-developer-v2-implementation.md`.
- **Tool split:** `backlog/520-primitives-stack-v2.md`, `backlog/521c-internal-generation-improvements.md`, `backlog/521d-external-generation-improvements.md` (Buf generates; CLI orchestrates; CI gates).

## Stage map

- **Stage 0 – Intake:** `backlog/300-400/367-0-feedback-intake.md` (capture + element wiring + methodology_profile_id tagging).
- **Stage 1 – Methodology profile:** one of:
  - Scrum: `backlog/300-400/367-1-scrum-transformation.md`
  - SAFe: `backlog/300-400/367-1-safe-transformation.md`
  - TOGAF: `backlog/300-400/367-1-togaf-transformation.md`
- **Stage 2 – Process orchestration:** Temporal Process primitives + operator (`backlog/499-process-operator.md`) consuming domain facts and emitting process facts (never direct domain RPC on the hot path).
- **Stage 3 – Agents:** AmeidePO/AmeideSA/AmeideCoder consume process facts and publish domain intents (no cross-layer shortcuts).

## Bright-line invariants

- Methodology semantics live in Stage 1; orchestration/timeboxes live in Stage 2; agent autonomy lives in Stage 3.
- Runtime integration is event-driven only (bus topics + SDKs); synchronous calls are control-plane only (operators resolving static config).
