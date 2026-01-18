---
title: "527 Transformation — Domain Specification (v3: Git-backed Enterprise Repository owner)"
status: draft
owners:
  - transformation
  - platform
created: 2026-01-18
supersedes:
  - 527-transformation-domain-v2.md
  - 527-transformation-domain.md
related:
  - 694-elements-gitlab-v6.md
  - 496-eda-principles-v6.md
  - 527-transformation-e2e-sequence-v6.md
  - 660-transformation-primitive-v4-goals-invariants.md
---

# 527 Transformation — Domain Specification (v3: Git-backed Enterprise Repository owner)

This v3 reframes the Transformation Domain as the **owner of canonical design-time truth stored in Git**.

It supersedes the “elements/relationships are canonical Postgres rows” posture in `backlog/527-transformation-domain-v2.md` while preserving the same platform invariants:

- Domain is the single writer for canonical truth.
- Facts are emitted only after commit.
- Protos govern inter-primitive contracts; idempotency is mandatory.

## Canonical decisions (normative)

- Git-first repository substrate: `backlog/694-elements-gitlab-v6.md`
- Integration/EDA posture (hybrid, owner-writes): `backlog/496-eda-principles-v6.md`

## What the Domain owns (v3)

### 1) Canonical repository state (Git)

The Domain is the only primitive allowed to perform canonical Git operations against the tenant repository:

- create/ensure the GitLab project (internal-only),
- create/update branches and commits,
- open/update/merge merge requests,
- tag published snapshots when required.

### 2) Governance state (Postgres, minimal and auditable)

Postgres remains canonical for what is required to enforce policy and traceability, for example:

- tenant/repository registry and mapping to GitLab projects,
- transformation objects (identity, state machine, links to repo refs),
- governance decisions (DoR/accept/release gates), policies, and evidence references,
- durable work tracking (WorkRequests) and executor outcomes,
- audit pointers (MR id, commit SHA, pipeline id) for every emitted fact.

### 3) Facts (event plane)

The Domain emits facts after commit (where “commit” is a Git commit/merge/tag + durable audit pointer recording).

These facts are used to:

- start/correlate Zeebe processes (via Kafka→Zeebe ingress),
- drive projections (indexes/graphs/timelines),
- provide an audit spine for “why did this change happen”.

## What the Domain does not own (v3)

- Orchestration control flow (owned by Process primitives / Zeebe).
- Derived read models (owned by Projection primitives).
- Executor runtime state (owned by Integration/Agent primitives), except for the WorkRequest record and outcome pointers.

## Relationship to existing proto surfaces (migration posture)

Older Transformation Domain APIs and schemas (knowledge/governance/work) were designed around a Postgres element substrate.

Under v3, there are two valid migration paths:

1) **Keep the proto APIs but reinterpret semantics as Git-backed operations** (preferred when minimizing client churn).
2) **Introduce a new Git-backed command surface** for repository writes and mark the DB-element APIs as superseded for Transformation.

Either way, the owner-only write rule must hold: non-domain primitives must not mutate the repo directly and must request changes via Domain APIs.

## Projections (derived, rebuildable)

Because canonical truth is now files:

- the “graph” is computed by indexing repository content (inline links + relationship files),
- the projection database is rebuildable from Git history + Domain audit pointers.

## Superseded guidance (what to stop treating as current)

The following older postures remain important history but are no longer the current standard for Transformation’s repository substrate:

- `backlog/527-transformation-domain-v2.md` (assumes Elements/Relationships in Postgres are canonical)
- `backlog/527-transformation-domain.md` (older compiled-workflow assumptions + DB substrate posture)
