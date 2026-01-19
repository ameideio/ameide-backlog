---
title: "506 — Scrum Vertical (v6: Git-backed ProcessDefinitions, Zeebe/Flowable, no Definition Registry)"
status: draft
owners:
  - platform
  - transformation
created: 2026-01-19
supersedes:
  - 506-scrum-vertical-v2.md
related:
  - 508-scrum-protos.md
  - 520-primitives-stack-v6.md
  - 511-process-primitive-scaffolding-v3.md
  - 496-eda-principles-v6.md
  - 509-proto-naming-conventions-v6.md
  - 694-elements-gitlab-v6.md
---

# 506 — Scrum Vertical (v6: Git-backed ProcessDefinitions, Zeebe/Flowable, no Definition Registry)

## Functional storyline (what a user experiences)

1) A product team runs Scrum to deliver a deployable change
- They maintain a Product Backlog, commit a Sprint Backlog, and drive work to Done.
- “Done” includes evidence (tests, build artifacts, deploy outcome) required by policy.

2) The platform shows progress and enforces gates
- People see a Kanban-style progress view derived from process progress facts and domain facts.
- When a gate is required (DoR/DoD/approval), it is enforced by the owning Domain policy and the process flow.

3) The platform publishes and audits outcomes
- Publication is a governed change to the tenant Enterprise Repository (Git), advancing `main` (baseline commit SHA, optionally tagged).
- Audit pointers (MR id, commit SHA, pipeline ids) are recorded and facts are emitted after commit.

## Big changes vs v2 (normative)

### 1) ProcessDefinitions are Git-backed files (no Definition Registry)

Scrum ProcessDefinitions (BPMN) live in the tenant Enterprise Repository as versioned files, for example:

- `processes/<module>/<process_key>/v<major>/process.bpmn`

The “Definition Registry” as a separate canonical store is deprecated under the Git-backed Enterprise Repository posture (`backlog/694-elements-gitlab-v6.md`).

### 2) BPMN runtimes: Zeebe and Flowable only (business)

- Business BPMN executes on **Zeebe/Camunda 8** or **Flowable**, per `backlog/520-primitives-stack-v6.md`.
- Temporal is platform-only (non-BPMN platform workflows such as login/onboarding).

### 3) Process primitive definition (v6)

A Process primitive is the deployable worker implementation for a ProcessDefinition:

- **ProcessDefinition (design-time):** `process.bpmn` in Git (governed artifact).
- **Process primitive (runtime):** worker microservice implementing BPMN task side effects, verified for coverage, deployed alongside (or ahead of) the BPMN.
- **Promotion:** verify the BPMN against the target runtime profile (Zeebe/Flowable), run the worker test suites, then roll out the worker and deploy the BPMN to the runtime.

## Contract surfaces (Scrum)

The Scrum domain/process seam remains proto-governed:

- `backlog/508-scrum-protos.md` (domain intents/facts + process facts)
- Integration posture: `backlog/496-eda-principles-v6.md` (hybrid commands + owner facts after commit)
- Semantic identity: `backlog/509-proto-naming-conventions-v6.md` (`io.ameide.*`)

## Minimal end-to-end slice (v6)

1) Domain records Scrum truth
- Transformation Domain (Scrum profile) validates and persists Scrum aggregates (PBIs, Sprint, commitments).
- Domain emits facts via outbox (DB-backed) after persistence.

2) Process orchestrates long-running coordination
- A Scrum ProcessDefinition (BPMN) models timeboxes/gates and emits process progress facts.
- Worker implementations call owners for commands; they do not mutate canonical stores.

3) Projection builds read models (boards/timelines)
- Kanban/timeline views are derived from facts (projection-owned, rebuildable).

## Known gaps (explicit; out of scope here)

- Multi-tenant runtime mapping (how a single Process primitive instance is scoped across tenants) is TBD.
- The final “memory model” abstraction over Git files (IDs/citations/baselines) is TBD; treat memory as projection-owned retrieval over Git plus audit pointers for now.

