---
title: 657 — Transformation Clean Target (v2: Git-backed repository, hybrid EDA, memory model TBD)
status: draft
owners:
  - transformation
  - platform
created: 2026-01-18
supersedes:
  - 657-transformation-domain-clean-target.md
related:
  - 694-elements-gitlab-v6.md
  - 496-eda-principles-v6.md
  - 509-proto-naming-conventions-v6.md
  - 527-transformation-v6-index.md
  - 656-agentic-memory-v6.md
---

# 657 — Transformation Clean Target (v2: Git-backed repository, hybrid EDA, memory model TBD)

This v2 clean-target backlog updates 657’s “target-state contract” to align with the v6 story:

- Git-backed Enterprise Repository (`backlog/694-elements-gitlab-v6.md`)
- Hybrid integration posture (`backlog/496-eda-principles-v6.md`)
- `io.ameide.*` semantic identity naming (`backlog/509-proto-naming-conventions-v6.md`)

## What stays the same

- Projection is the read path for humans and agents (browse/search/history/diff/context assembly).
- Domain is the owner for canonical truth (owner-only writes; facts after commit).
- Idempotency and traceability are mandatory.
- Authorization must be enforced at gRPC boundaries (not only at HTTP edges).

## What changes (v2 alignment)

### 1) Canonical repository substrate is Git-backed

Transformation’s canonical repository content lives as files in a tenant Git repository.
Postgres remains for tenancy/governance/work orchestration and audit pointers.

### 2) EDA posture is v6 (hybrid)

Kafka remains the standard event plane for facts/intents, but v6 explicitly allows RPC for commands where appropriate (owner-only writes still holds).

### 3) Semantic identity uses `io.ameide.*`

CloudEvents `type` and any stable semantic identities use `io.ameide.*`.

## TBD: Agent memory model under Git-first

Agent memory is still a projection concern, but the canonical “memory model” (IDs, citations, and how we represent element-version semantics over Git files) is **TBD**.

This backlog intentionally does not decide:

- whether we embed stable IDs in files (frontmatter),
- the final citation shape (commit SHA + path vs content hash vs owner-issued version ids),
- or how “baselines” are represented beyond “`main` plus commit anchors”.

Those decisions will be made in a dedicated memory v6 backlog once the approach is clear.
See `backlog/656-agentic-memory-v6.md`.
