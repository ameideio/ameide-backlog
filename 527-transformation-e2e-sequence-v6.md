---
title: "527 Transformation — E2E Sequence (v6: Git-backed Enterprise Repository + Zeebe request→wait→resume)"
status: draft
owners:
  - transformation
  - platform
created: 2026-01-18
supersedes:
  - 527-transformation-e2e-sequence-v5.md
  - 527-transformation-e2e-sequence-v4.md
related:
  - 694-elements-gitlab-v6.md
  - 496-eda-principles-v6.md
  - 527-transformation-process-v4.md
  - 527-transformation-domain-v3.md
  - 660-transformation-primitive-v4-goals-invariants.md
---

# 527 Transformation — E2E Sequence (v6: Git-backed Enterprise Repository + Zeebe request→wait→resume)

This v6 updates the v4/v5 sequence narratives by making the **repository substrate Git-first**:

- Canonical design-time truth lives as **files in a tenant Git repository** (GitLab project).
- Postgres is used for **tenancy/governance/work orchestration** and **audit pointers**, not for “elements/relationships” as canonical content.

See `backlog/694-elements-gitlab-v6.md` for the normative platform decisions behind this posture.

## Canonical assets (v6)

- Platform decisions: `backlog/694-elements-gitlab-v6.md`
- Integration posture: `backlog/496-eda-principles-v6.md`
- Process packaging + semantics: `backlog/527-transformation-process-v4.md`
- Domain ownership posture (Git-backed): `backlog/527-transformation-domain-v3.md`
- Executable BPMN (current): `primitives/process/transformation_v4/bpmn/process.bpmn`
- Worker implementation (current): `primitives/process/transformation_v4/`

## The two invariants (v6)

### 1) Owner-only writes still holds — the owner just writes to Git now

- The Transformation **Domain** is the owner of canonical repository state.
- “Commit” for owner facts means “the Git operation succeeded” (commit/merge/tag) and the Domain durably recorded the audit pointer(s) it will rely on.
- No other primitive writes to the tenant repo directly (Process/Agent/Integration/UISurface must route via the owner).

This is the same invariant as 496, applied to a Git-backed store: `backlog/496-eda-principles-v6.md`.

### 2) Long work is still request→wait→resume — but `work_id` must be a business id

The request→wait→resume rule from v5 remains correct, but **`work_id` must be stable and owner-issued**:

- Request step asks the Domain to create a durable WorkRequest / external action id.
- Wait step correlates on that Domain-issued `work_id`.
- Resume happens via message publication using `messageId = work_id` (buffer-level idempotency), plus Domain/process idempotency for end-to-end correctness.

## How “elements” and “relationships” appear in v6

Per `backlog/694-elements-gitlab-v6.md`:

- “Elements” are files (Markdown/YAML/JSON/etc.) in the repo.
- “Relationships” are both:
  - inline links inside files, and
  - standalone relationship files (e.g., `relationships/<id>.yaml`),
  with a derived graph/index computed by projections.

## End-to-end scenario (minimal happy path)

This is the smallest scenario that proves the v6 posture end-to-end while reusing existing primitive roles.

### Stage 0 — Tenant repository exists (platform-owned)

1) Platform creates/ensures a tenant “Enterprise Repository” GitLab Project (internal bot identity).
2) Domain stores only the mapping and audit anchors (project id, default branch, etc.).

### Stage 1 — Transformation requested → Requirements batch starts

1) UISurface requests a new transformation (Domain command).
2) Domain creates (or selects) the working branch(es)/MR(s) for the transformation and records the audit pointers.
3) Domain emits `io.ameide.transformation.fact.transformation.requested.v1` (after Git commit/branch/MR creation as applicable).
4) Kafka→Zeebe ingress publishes the message to Zeebe and starts `transformation_requirements_v4`.

### Stage 2 — Requirements analysis (agent) → human DoR → publish requirements

1) Process executes `transformation.requirements.agent.request.v1`.
2) Worker requests the Domain to create a WorkRequest for the Requirements agent (Domain is the authority that binds:
   - the repo ref (branch/MR),
   - allowed paths,
   - and the idempotency key).
3) Agent primitive runs and emits one of:
   - `io.ameide.agent.fact.run.completed.v1`
   - `io.ameide.agent.fact.run.needs_input.v1`
   - `io.ameide.agent.fact.run.failed.v1`
   correlated by the Domain-issued `work_id`.
4) Process reaches user tasks (review + DoR).
5) Process executes `transformation.requirements.publish.request.v1`.
6) Worker calls Domain to “publish requirements”, which in v6 is typically:
   - apply accepted changes to the working branch, and
   - merge to `main` (baseline) via MR (policy gated),
   - record `main` commit SHA / MR id / pipeline id as audit pointers.
7) Domain emits:
   - `io.ameide.transformation.fact.requirements.published.v1`
   - `io.ameide.transformation.fact.requirement.ready_for_delivery.v1` (one per requirement)
8) Ingress correlates those facts into the Delivery batch process.

### Stage 3 — Delivery/Acceptance/Release (same pattern, different executors)

For each long-running step (coder/tests/preview validate/build/publish/gitops promote/rollout verify):

1) Process requests work (short job) and gets a Domain-issued `work_id`.
2) Process waits on the completion message correlated by `work_id`.
3) The external executor records outcomes/evidence back to the Domain (owner), and the Domain decides what becomes canonical Git state (commit/MR update/merge/tag).
4) Domain emits facts that advance the next batch stage(s); projections update derived views/graphs.

## What this supersedes (and what it preserves)

This v6 sequence supersedes the “elements/relationships are canonical rows” reading of older Transformation docs (`backlog/527-transformation-domain-v2.md`, and the 303-era element substrate), but preserves:

- Process semantics (Zeebe request→wait→resume),
- Owner-write + facts-after-commit integration posture (496),
- Projection-as-derived-view posture (read models are rebuildable).
