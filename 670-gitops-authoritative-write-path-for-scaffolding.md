---
title: 670 – GitOps Authoritative Write-Path for Primitive Scaffolding
status: active
owners:
  - gitops
  - platform
  - devx
created: 2026-01-13
updated: 2026-01-13
---

## Summary

Define the **single authoritative write-path** that turns “I want a new primitive/service wired into GitOps” into “a PR in `ameide-gitops`”, with guardrails and deterministic output.

This backlog exists because ArgoCD already **auto-discovers** desired state from repo conventions, but we still lack a single, CI-owned path that reliably **creates** the required GitOps files and lands them on `main`.

## Non-negotiable decision

**Desired state is changed only via Git PRs in `ameide-gitops`.**  
No manual `kubectl apply`, no Argo UI overrides, no in-cluster controllers committing to Git for image promotion or scaffolding.

ArgoCD remains the reconciler; CI remains the writer.

## Current state (what already works)

1. **ArgoCD auto-wires components by discovery**  
   The root `ApplicationSet` scans `environments/_shared/components/**/component.yaml` and generates Applications automatically.

2. **Values layering is already standardized**  
   `argocd/applicationsets/ameide.yaml` composes base/cluster/env/shared/env-specific values in a consistent order.

3. **Promotion via PRs already exists**  
   Image digest promotion is already implemented as a PR-based workflow in this repo (CI opens PRs; merge drives Argo).

## Problem

We do not have a single, deterministic, CI-owned path for **scaffolding GitOps wiring** itself. Today it is easy to end up with:

- local CLI-generated YAML that drifts from the GitOps repo’s real conventions
- manual edits and copy/paste wiring
- inconsistent enablement defaults across envs
- missing smokes or miswired smokes

## Target model (one consistent way)

### A) GitOps repo owns the generator and the workflow

Add a workflow in `ameide-gitops`:

- `workflow_dispatch`: `.github/workflows/scaffold-primitive-gitops.yaml`
- inputs: `kind`, `name`, `version`, `with_smoke`, optional `env`

Workflow behavior:

1. Generate GitOps wiring files **matching this repo’s conventions**, including:
   - `environments/_shared/components/apps/primitives/<kind>-<name>-<version>/component.yaml`
   - `sources/values/_shared/apps/<kind>-<name>-<version>.yaml` (default `enabled: false`)
   - `environments/_shared/components/apps/primitives/<kind>-<name>-<version>-smoke/component.yaml` (optional)
   - `sources/values/_shared/apps/<kind>-<name>-<version>-smoke.yaml` (optional)
   - `sources/values/env/<env>/apps/<kind>-<name>-<version>.yaml` (optional; default `enabled: false`)
2. Run guardrails:
   - `helm template` for the generated workload values (raw manifests chart)
   - `helm template` for the generated smoke values (helm-test-jobs chart, when enabled)
3. Open a PR (branch-named deterministically).

### B) CLI becomes a trigger (not a writer)

In the `ameide` repo CLI:

- Deprecate local “write GitOps files” behavior (`--include-gitops` as an authoring mechanism).
- Replace with “trigger GitOps scaffolding PR”:
  - `ameide primitive gitops scaffold ...` or `ameide primitive scaffold --gitops-pr ...`
  - Implementation calls GitHub `workflow_dispatch` for `ameideio/ameide-gitops`.

This prevents the CLI from drifting from GitOps conventions: the generator lives where the conventions live.

## Acceptance criteria (definition of done)

1. **Single entry point**: the documented/scoped path to add GitOps wiring is: “trigger workflow → PR → merge”.
2. **Deterministic output**: generator produces stable paths and stable YAML for the same inputs.
3. **Safe defaults**:
   - `_shared` values default to `enabled: false`
   - no env is enabled unless explicitly requested (dev only by default)
   - no secret values are ever written to Git
4. **Guardrails**: workflow refuses invalid names/kinds/versions and refuses overwriting existing wiring unless explicitly forced.
5. **Argo convergence** (dev): after merge, the generated Applications appear and sync; any smokes run as Argo hook Jobs.

## How we test (evidence)

### 1) Workflow evidence (GitHub Actions)

- Run ID + logs show:
  - generated file list
  - validation steps executed
  - PR created successfully

### 2) Repo evidence (PR diff)

- PR contains only the expected files.
- YAML matches the repo conventions (enabled gating, `image.ref` patterns, smokes).

### 3) Cluster evidence (ArgoCD)

- After merge, Argo Application(s) created by discovery:
  - `{{ env }}-{{ componentName }}` exists
  - sync status `Synced`, health `Healthy`
- Smokes (if enabled) succeed as PostSync hook Jobs.

## Cross-references

This backlog supersedes and/or clarifies prior guidance:

- `backlog/648-config-secrets-taxonomy-ci-only-deploy.md` – CI-only desired state and “one way” principle.
- `backlog/444-terraform-v2.md` – CI-first provisioning and “no manual mutation”.
- `backlog/465-applicationset-architecture-preview-envs-v2.md` – ApplicationSet discovery model and security posture.
- `backlog/611-trunk-based-main-and-gitops-environment-promotion.md` – PR-required gate correctness and promotion mechanics.
- `backlog/484a-ameide-cli-primitive-workflows.md` – CLI workflow updates: CLI triggers GitOps PRs, does not write GitOps files.
- `backlog/484d-ameide-cli-migration.md` – Migration updates: GitOps scaffolding moved to GitOps repo workflow.
- `backlog/520-primitives-stack-v2-tdd.md` – end-to-end loop (update: GitOps wiring is PR/workflow owned, not local CLI writes).
- `backlog/521d-external-generation-improvements.md` – CLI orchestration boundaries (update: GitOps wiring is “trigger PR”, not “write files”).
- `backlog/484a-ameide-cli-primitive-workflows.md` / `backlog/484d-ameide-cli-migration.md` – CLI guidance (update: remove `--include-gitops` as the canonical GitOps write-path).

## Deprecations / superseded notes (scope)

As of this backlog, the following are considered **historical** where they imply “CLI writes GitOps wiring directly”:

- `ameide primitive scaffold --include-gitops` as the *primary* mechanism to create GitOps files.
- Any guidance that implies developers/agents should manually add/patch GitOps wiring in-place without using the GitOps repo’s generator workflow.

## Implementation (landed)

This backlog is implemented in `ameide-gitops`:

- Generator: `scripts/scaffold-primitive-gitops.py`
  - Validates inputs and refuses overwrites unless `--force`.
  - Defaults to safe, inert desired state: workloads are scaffolded as `enabled: false` with `image.ref: ""`.
  - Smoke scaffolding is supported; smoke tests are **opt-in** (to avoid failing in envs where the workload remains disabled).
- CI workflow: `.github/workflows/scaffold-primitive-gitops.yaml`
  - Runs the generator on `workflow_dispatch`.
  - Validates by running `helm template` with the generated values.
  - Opens a PR via `peter-evans/create-pull-request`.
