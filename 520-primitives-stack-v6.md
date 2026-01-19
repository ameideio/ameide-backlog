---
title: "520 – Primitives Stack (v6: BPMN orchestration on Zeebe/Flowable, `io.ameide.*`, hybrid integration)"
status: draft
owners:
  - platform
created: 2026-01-18
supersedes:
  - 520-primitives-stack-v3.md
  - 520-primitives-stack-v2.md
related:
  - 496-eda-principles-v6.md
  - 694-elements-gitlab-v6.md
---

# Primitives Stack v6 (Backlog 520) — BPMN orchestration on Zeebe/Flowable, `io.ameide.*`, hybrid integration

This v6 is the current runtime posture “spine”:

- BPMN executes on **Camunda 8 / Zeebe** (default) or **Flowable** (supported profile) for business Process primitives.
- Temporal remains platform-only (not business BPMN orchestration).
- Integration is governed by owner-only writes + proto contracts, using a hybrid model (RPC commands + event facts).
- Inter-primitive semantic identities use `io.ameide.*`.

> **Terminology:** the term “capability” is deprecated in posture docs. Use **module** to mean a business domain grouping (e.g., `transformation`, `sales`). Use **primitive** to mean the deployable microservice unit (Domain/Process/Projection/Integration/Agent/UISurface).
>
> “Module” is an organizational label; it is not, by itself, a hard integration boundary. Integration rules are defined in `backlog/496-eda-principles-v6.md`.

## Summary (the decision that changes everything)

We adopt a BPMN orchestration posture for Ameide business domains:

1. **BPMN-authored business processes execute on Zeebe (default) or Flowable (supported profile).**
   - BPMN is the runnable program (engine semantics).
   - A Process primitive is “BPMN definition + worker implementations + verification”, not “Temporal workflow service”.
2. **Temporal is platform-only** (internal platform concerns, e.g. platform login/onboarding orchestration) and is not used as a BPMN execution target.
3. The BPMN→Temporal transpilation/compile-to-IR effort is discontinued (no backward compatibility layers).

The non-negotiable rule still holds: **the diagram must not lie**.

## What stays the same (principles that still apply)

- Operators are operational only; they never encode business logic.
- Domains are systems of record; processes/orchestration never become canonical truth.
- Protos define contracts; generation is deterministic; CI gates enforce drift discipline.
- Integration rules remain (owner-only writes, facts-after-commit, idempotency, at-least-once consumers).
- Configuration has a single authority (cluster conventions vs GitOps-injected vs request payloads).

## What changes (older posture → v6)

### A) Process runtime posture

- BPMN Process primitives = BPMN deployments + worker microservices implementing task side effects.
- Long work is modeled as explicit BPMN wait states and resumed by message correlation (request→wait→resume).
- **ProcessDefinitions are design-time governed artifacts stored in the tenant Enterprise Repository.**
  - BPMN is authored and versioned as files in the tenant repository and published by advancing the baseline (`main`).
  - Repository layout (recommended):
    - `processes/<module>/<process_key>/v<major>/process.bpmn`
    - optional: `processes/<module>/<process_key>/v<major>/bindings.yaml` (portable binding metadata / policy hooks)
    - optional: `processes/<module>/<process_key>/v<major>/README.md` (functional intent + gates)
  - Vendor-correct naming:
    - `process_key` SHOULD equal the BPMN `<process id="...">` (runtime key semantics across Zeebe/Flowable).
  - Versioning rules:
    - **Major** is encoded in the path (`v<major>`).
    - **Minor/patch** are Git commits on `main` (optionally tagged), anchored by commit SHA for auditability.
    - Recommended major policy: a major bump SHOULD introduce a new BPMN process id (new `process_key`) to avoid breaking changes in one runtime key’s version chain; instance migration across majors is explicitly TBD.
  - A Process primitive is the runtime implementation (worker + deploy logic) that executes the **published** process definition version.
  - This makes process orchestration “definition-driven”: agents/humans change the governed artifact; the Process primitive executes it.
  - **Gap (TBD):** multi-tenant mapping rules for deploying tenant-specific process versions without collisions (flagged; not addressed here).

#### Supported BPMN runtime profiles (v6)

The same ProcessDefinition artifact can be validated and deployed against different BPMN runtimes by selecting a profile:

- **Zeebe (Camunda 8)** — default runtime for business BPMN process primitives.
- **Flowable** — supported alternative runtime under an explicit profile.

Promotion of a ProcessDefinition MUST include runtime-profile verification (deployability + semantics) against the deployed target runtime. Temporal is not a BPMN runtime target in this posture.

### B) Integration posture (hybrid)

See `backlog/496-eda-principles-v6.md` for the normative rules:

- Commands (often RPC) request changes from an owner.
- Facts (often events) propagate committed outcomes for fan-out, replay, and derived read models.

### C) Naming posture (`io.ameide.*`)

All inter-primitive semantic identities (CloudEvents `type`, and Zeebe message names when mapped from facts) use `io.ameide.*`.

Do not introduce `com.ameide.*` identities in new work.

### D) Canonical store is not always a DB

Domains may be Git-backed owners (canonical content as files in a tenant repository). In that case:

- “Commit” means Git commit/merge/tag + durable audit pointers.
- Only the owning Domain performs canonical Git writes.
- Projection read models may index Git content but remain derived and rebuildable.

See `backlog/694-elements-gitlab-v6.md`.

## Tooling contract (CLI + CI)

The Ameide CLI is the guardrails tool:

- Verify:
  - validates BPMN against selected profiles,
  - validates Zeebe deployability constraints,
  - validates worker coverage for all side-effect steps.
- Scaffold:
  - produces worker stubs/tests in the owning primitive repos,
  - does not generate Temporal workflow code from BPMN.

## Backlog mapping / replacements

- `backlog/511-*` Process primitive scaffolding/compiler docs that assume Temporal-backed business orchestration are superseded by the Zeebe-based direction.
- EDA posture docs are superseded by `backlog/496-eda-principles-v6.md` (hybrid + `io.ameide.*`).
