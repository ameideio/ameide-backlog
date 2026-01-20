---
title: 610 – CI Rationalization & Cost Controls
status: active
owners:
  - platform
  - test-infra
created: 2025-12-26
updated: 2026-01-20
---

## Summary

Reduce CI latency, queue pressure, and wasted compute **without weakening correctness** by making GitHub Actions workloads:

- Cancel superseded runs aggressively
- Run only the checks relevant to the change (path/scoped gates)
- Avoid always-on publish pipelines for non-release commits
- Track and enforce CI performance budgets (time-to-signal, time-to-green)

This backlog complements testing-contract backlogs by focusing on *operational economics* (minutes + queues), not on *test semantics*.

## Context

We already have strong testing/verification contracts:

- `backlog/430-unified-test-infrastructure-v2-target.md` — strict phases + JUnit evidence contract
- `468-testing-front-door.md` — “run what CI runs” local front door
- `537-primitive-testing-discipline.md` — normative requirements for primitives/operators
- `521h-external-verification-improvements.md` — log for changes to external gates

However, CI can still be “correct but too heavy” if:

- heavyweight jobs trigger on unrelated changes
- CD/publish jobs run on every branch push (e.g. `main`)
- repeated pushes/PR updates pile up queued runs that won’t ever merge
- a single monolithic job forces worst-case setup on every PR

## Goals

1. **Time-to-first-signal** (first failing check or first passing core gate) is consistently low.
2. **Time-to-green** on typical PRs is predictable and bounded.
3. **Queue time** stays low during normal development (no “hours queued” from always-on pipelines).
4. CI remains a **strict correctness gate** where it matters; we reduce work by *scoping*, not by “skipping correctness”.

## Non-goals

- Redesigning the testing contract (430/468/537) or watering down invariants.
- Replacing CI with local-only discipline.
- Changing release governance (promotion PR policy, GitOps release flow) beyond what is needed for cost controls.

## Vendor constraints (GitHub Actions semantics)

These are “gotchas” we must encode into docs and workflow design so we don’t accidentally fight GitHub’s behavior:

1. **Workflow skipped ⇒ required checks stay Pending ⇒ merge blocks**  
   If a workflow is skipped due to trigger-level filters (`paths`, `branches`) or skip markers in commit messages, its checks can remain **Pending**. If those checks are required, the PR merge is blocked.
2. **Job skipped via `if:` ⇒ reports Success ⇒ does not block merge (even if required)**  
   Job-level skipping is not a safe enforcement mechanism by itself.
3. **`needs:` + failure/skip ⇒ downstream skips unless `if: ${{ always() }}`**  
   A final “gate” job must be forced to run even when upstream jobs fail/skip.
4. **Concurrency: define groups carefully to avoid cross-cancel**  
   Prefer workflow-unique concurrency groups (commonly include `${{ github.workflow }}`) to avoid canceling other workflows unintentionally.
5. **Metrics endpoint stability: `/actions/runs/{run_id}/timing` is closing down**  
   “Get workflow usage” and “Get workflow run usage” endpoints (including `/timing`) are closing down as part of GitHub billing platform changes.

## Required-check-safe scoping pattern (recommended)

For any workflow whose checks are configured as **required** for merge, prefer this architecture:

1. Trigger the workflow on all PRs to `main` (no trigger-level `paths` filters).
2. Use a `changes` (diff) job to compute boolean outputs for “what changed”.
3. Run suite jobs conditionally based on those outputs (`if:`).
4. Have a single **required** `gate` job that:
   - has `needs: [...]` on all suite jobs
   - runs with `if: ${{ always() }}`
   - fails if any *relevant* suite did not succeed

This lets branch protection require **only the gate job’s check**, while the workflow still scopes expensive work to what changed.

## Measurement note

To measure cost/latency without relying on the closing `/timing` endpoint:

- **Queue time**: workflow run created → first job started
- **Runner occupancy**: sum over jobs of (`started_at` → `completed_at`)
- Prefer job timestamp–based measurement via workflow jobs APIs (or `gh run view --json jobs`) over run-level `/timing`.

## Target metrics (initial)

Track and publish these as a weekly snapshot:

- `p50/p90` **PR Time-to-first-signal**: target `p50 <= 5m`, `p90 <= 15m`
- `p50/p90` **PR Time-to-green**: target `p50 <= 15m`, `p90 <= 45m`
- `p95` **queue time** for PR-required workflows: target `<= 2m`
- “wasted runs”: count of cancelled/superseded runs per week, and the delta after cancellation changes

## Work items

### A) Cancellation and concurrency policy

- Ensure all PR workflows use `concurrency` with `cancel-in-progress: true` (grouped by PR number).
- Ensure CD/publish workflows cancel superseded runs on branch pushes (grouped by branch/ref).
- Make queue amplification visible (metrics + dashboards if available).

### B) Scope heavy checks to relevant changes

Split/gate heavy work by change category:

- UI/Playwright: only when platform UI or its test deps change
- Operator suites: only when `operators/**` or acceptance harness changes
- Integration packs: already path-filtered; ensure the filters stay accurate as packs expand
- SDK regeneration/drift: only when proto or SDK surfaces change

Prefer explicit `paths-filter`-style gates over “always run and hope cache saves you”.

### C) Stop “always-on” CD on branch pushes (unless relevant)

Publishing images/packages on every branch push is expensive and creates long queues.

Target behavior:

- **Manual dispatch** always runs (for smoke/publish)
- **Tags** always run (release)
- **Branch pushes** run only if relevant directories changed (`on.push.paths`, plus a defensive diff gate)

### D) Cache and dependency hygiene

- Replace apt installs for common deps with either preinstalled tool usage or service containers where possible.
- Cache expensive tool downloads (e.g., Playwright browsers) with a stable key.
- Make “fast paths” explicit and documented (keep `468` aligned with CI reality).

### E) Required-check policy under trunk-based `main`

Avoid duplicated “full CI twice” by aligning required checks with the trunk model:

- Required checks apply to **PRs into `main`** (the only integration path).
- “Promotion” is a **GitOps digest-copy PR** (dev → staging → prod) and should have its own minimal checks (lint/policy/manifest render), not the full application CI suite.

#### Historical note (pre-trunk `dev → main`)

Earlier, the repo used a `dev` branch and a `dev → main` promotion PR flow. Under that model, the correct optimization was:

- Preferred: only run the full CI suite on PRs targeting `dev`, and a minimal “promotion smoke” on PRs targeting `main`.
- If branch protection requires specific checks: create “proxy” checks for `main` that are quick but still meaningful.

## Change log discipline

Any change to:

- workflow triggers (`on:`), concurrency, or scoping rules
- what is required for merge (required checks, job gating)
- “fast path” semantics

…must add an entry to `521h-external-verification-improvements.md`.

## Repo alignment status (ameide-gitops)

As of `2026-01-11`, `ameide-gitops` largely follows the scoping + cancellation principles above **without relying on trigger-level skipping for PR-required gates**.

### Achievements

- **Update (2026-01-20):** `ameide test` is now the canonical Phase 0/1/2 gate (local-only) and Phase 3 is split into explicit `ameide test e2e` (PRs: `ameideio/ameide#582`, `ameideio/ameide#589`).
- **Update (2026-01-20):** Dev workspaces now ship the required toolchain (`go`, `pnpm`, `uv`, `buf`) so `ameide test` can run deterministically in Coder-based environments (PR: `ameideio/ameide#584`).

### New CI pressure source to manage: Coder workspaces E2E

The AmeideDevContainerService (Coder-based human workspaces; 626/628/629) introduces a natural “full E2E” that is:

- slow (workspace provision/build + app proxy verification)
- secret-heavy (Coder token, GitHub bot token, optional Codex auth)
- environment-scoped (AKS dev only)

Required-check-safe recommendation:

- Keep the Coder E2E as **workflow_dispatch** and/or scheduled (nightly), not as a required PR check.
- If we want PR signal, add a fast “Coder smoke” gate that proves `coder.dev.ameide.io` is healthy and past `/setup`,
  without creating PR spam (see 629).

- **Required-check-safe scoping**: `GitOps / Gate` is the single required check and scopes work via a diff-based `changes` job + an always-run gate.
- **Aggressive cancellation**: PR workflows use `concurrency` with `cancel-in-progress: true` (and workflow-unique groups to avoid cross-workflow cancellation).
- **Runner routing is GitHub-config-driven**: most CI workflows run on `runs-on: ${{ vars.AMEIDE_RUNS_ON }}` (no workflow defaults), allowing switching between `arc-local` and `arc-aks-v2`.
- **Metrics without `/timing`**: a weekly snapshot workflow exists (`CI Metrics (Weekly)`).

### Workflows updated to match this pattern

- `GitOps / Gate` (`.github/workflows/gitops-gate.yaml`)
- Render/validation workflows called by the gate (examples):
  - `.github/workflows/helm-test.yaml`
  - `.github/workflows/kustomize-build.yaml`
  - `.github/workflows/image-policy.yaml`
  - `.github/workflows/vendor-charts.yaml`
- `CI Metrics (Weekly)` (`.github/workflows/ci-metrics-weekly.yaml`)

### Branch protection note (required checks)

To preserve correctness while allowing scoping:

- Prefer requiring only `GitOps / Gate` in branch protection (and keep other workflows as internal implementation details).
- Do not require conditional suite jobs directly; the gate job is the enforcement surface.

### Known exceptions (intentional)

Some operational workflows are intentionally pinned to `runs-on: ubuntu-latest` (e.g., Azure Terraform and image mirroring). Treat these as “ops substrate” rather than CI substrate; if we want them switchable too, use a separate GitHub variable for ops workflows rather than reusing `AMEIDE_RUNS_ON`.
