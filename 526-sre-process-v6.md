---
title: "526 SRE — Process Primitive (v6: Git-backed ProcessDefinitions, Zeebe/Flowable)"
status: draft
owners:
  - sre
  - platform
created: 2026-01-19
supersedes:
  - 526-sre-process.md
related:
  - 526-sre-capability.md
  - 520-primitives-stack-v6.md
  - 511-process-primitive-scaffolding-v3.md
  - 496-eda-principles-v6.md
  - 694-elements-gitlab-v6.md
---

# 526 SRE — Process Primitive (v6: Git-backed ProcessDefinitions, Zeebe/Flowable)

## Functional storyline (v6)

SRE processes orchestrate cross-domain work during operations:

- incident triage (pattern lookup → triage → remediation → verification → documentation),
- change verification after deployment,
- SLO burn monitoring and escalation,
- alert correlation into incidents.

## Runtime posture (v6)

- SRE ProcessDefinitions are BPMN files stored in the tenant Enterprise Repository (`backlog/694-elements-gitlab-v6.md`).
- Business BPMN executes on **Zeebe/Camunda 8** or **Flowable**, per `backlog/520-primitives-stack-v6.md`.
- Temporal is platform-only (non-BPMN platform workflows), not an SRE process backend.

## Process primitive definition (v6)

- **ProcessDefinition (design-time):** BPMN file in Git (versioned).
- **Process primitive (runtime):** worker microservice implementing BPMN task side effects and emitting progress facts.

The process primitive:

- consumes owner facts (at-least-once),
- requests changes from owners via commands/intents,
- emits process progress facts for projections (boards/timelines),
- never becomes canonical truth for domain state.

See `backlog/496-eda-principles-v6.md`.

