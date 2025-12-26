---
title: 610 – CI Rationalization & Cost Controls
status: active
owners:
  - platform
  - test-infra
created: 2025-12-26
updated: 2025-12-26
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

- `430-unified-test-infrastructure.md` — integration packs contract and structure
- `468-testing-front-door.md` — “run what CI runs” local front door
- `537-primitive-testing-discipline.md` — normative requirements for primitives/operators
- `521h-external-verification-improvements.md` — log for changes to external gates

However, CI can still be “correct but too heavy” if:

- heavyweight jobs trigger on unrelated changes
- CD/publish jobs run on every `dev` push
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

## Target metrics (initial)

Track and publish these as a weekly snapshot:

- `p50/p90` **PR Time-to-first-signal**: target `p50 <= 5m`, `p90 <= 15m`
- `p50/p90` **PR Time-to-green**: target `p50 <= 15m`, `p90 <= 45m`
- `p95` **queue time** for PR-required workflows: target `<= 2m`
- “wasted runs”: count of cancelled/superseded runs per week, and the delta after cancellation changes

## Work items

### A) Cancellation and concurrency policy

- Ensure all PR workflows use `concurrency` with `cancel-in-progress: true` (grouped by PR number).
- Ensure CD/publish workflows cancel superseded runs on `dev` pushes (grouped by branch/ref).
- Make queue amplification visible (metrics + dashboards if available).

### B) Scope heavy checks to relevant changes

Split/gate heavy work by change category:

- UI/Playwright: only when platform UI or its test deps change
- Operator suites: only when `operators/**` or acceptance harness changes
- Integration packs: already path-filtered; ensure the filters stay accurate as packs expand
- SDK regeneration/drift: only when proto or SDK surfaces change

Prefer explicit `paths-filter`-style gates over “always run and hope cache saves you”.

### C) Stop “always-on” CD on `dev` (unless relevant)

Publishing images/packages on every `dev` push is expensive and creates long queues.

Target behavior:

- **Manual dispatch** always runs (for smoke/publish)
- **Tags** always run (release)
- **Branch pushes** run only if relevant directories changed (diff gate)

### D) Cache and dependency hygiene

- Replace apt installs for common deps with either preinstalled tool usage or service containers where possible.
- Cache expensive tool downloads (e.g., Playwright browsers) with a stable key.
- Make “fast paths” explicit and documented (keep `468` aligned with CI reality).

### E) Required-check policy for promotion PRs (`dev → main`)

Avoid duplicating full PR CI on promotion PRs:

- Preferred: only run the full CI suite on PRs targeting `dev`, and a minimal “promotion smoke” on PRs targeting `main`.
- If branch protection requires specific checks: create “proxy” checks for `main` that are quick but still meaningful.

## Change log discipline

Any change to:

- workflow triggers (`on:`), concurrency, or scoping rules
- what is required for merge (required checks, job gating)
- “fast path” semantics

…must add an entry to `521h-external-verification-improvements.md`.

