---
title: "523 Commerce — Process (v6: Git-backed ProcessDefinitions, Zeebe/Flowable)"
status: draft
owners:
  - commerce
  - platform
created: 2026-01-19
supersedes:
  - 523-commerce-process.md
related:
  - 523-commerce.md
  - 520-primitives-stack-v6.md
  - 511-process-primitive-scaffolding-v3.md
  - 496-eda-principles-v6.md
  - 694-elements-gitlab-v6.md
---

# 523 Commerce — Process component (v6: Git-backed ProcessDefinitions, Zeebe/Flowable)

## Functional storyline (v6)

Commerce processes exist to make commerce operations reliable at fleet scale:

- onboarding a storefront domain (claim/verify/issue/attach),
- provisioning and rolling out store-sites,
- backfills and recovery,
- explicit handling of “real-time required” operations.

## Runtime posture (v6)

- Business ProcessDefinitions are BPMN files stored in the tenant Enterprise Repository (`backlog/694-elements-gitlab-v6.md`), versioned under `processes/<module>/<process_key>/v<major>/process.bpmn`.
- Business BPMN executes on **Zeebe/Camunda 8** or **Flowable** (runtime profiles), per `backlog/520-primitives-stack-v6.md`.
- Temporal is platform-only (non-BPMN platform workflows), not a commerce process backend.

## What the Process primitive is (v6)

- **ProcessDefinition (design-time):** BPMN file in Git.
- **Process primitive (runtime):** worker microservice implementing BPMN task side effects, plus deployment/verification glue for the chosen runtime profile.

Promotion (conceptually):

1) validate the BPMN against the selected runtime profile (Zeebe/Flowable),
2) verify worker coverage for all side-effect tasks,
3) run worker tests and any scenario smoke,
4) deploy worker + deploy BPMN to the orchestration runtime.

## Integration semantics (v6)

Processes are saga/process-managers:

- consume owner facts (at-least-once),
- request changes from owners via commands (RPC) or intents,
- never mutate canonical stores directly,
- emit process progress facts for projections (boards/timelines) and audit.

See `backlog/496-eda-principles-v6.md`.

