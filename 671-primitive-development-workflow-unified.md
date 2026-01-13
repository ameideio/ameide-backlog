---
title: 671 – Primitive Development Workflow (Unified)
status: active
owners:
  - platform
  - devx
  - gitops
created: 2026-01-13
updated: 2026-01-13
---

## Summary

Define one **end-to-end, enforceable** workflow for implementing and deploying primitives, with a clear split of responsibilities between:

- the **core repo** (code/proto/SDKs/CLI/operators), and
- the **GitOps repo** (`ameide-gitops`: desired state, values, components, smokes).

This document is the canonical “how we build primitives” playbook and cross-references the detailed backlogs that define individual policies.

## Non-negotiables (single way)

1. **Desired state is changed only via Git PRs.** No manual `kubectl apply`, no Argo UI overrides for Argo-managed workloads (break-glass only, then revert via Git).
2. **CI is the only writer for shared cloud environments.** CI opens PRs; merges land on `main`; ArgoCD reconciles.
3. **GitOps wiring is CI-owned.** Scaffolding GitOps wiring happens via `ameide-gitops` workflow → PR → merge (see 670). Local “write GitOps files” (e.g. `--include-gitops`) is historical and not the canonical path.
4. **Secrets are KV-only; settings are Git-only.** (648)
5. **Smokes run in ArgoCD.** CI validates renders and policy; cluster-level smokes are Argo hook jobs.

## Repos and responsibilities

### Core repo (e.g. `ameide`)

Owns:
- Protos and Buf module(s)
- SDKs and generated-only artifacts
- Primitive runtime code (`primitives/<kind>/<name>/...`)
- Operators and CRD schemas
- `ameide` CLI (scaffold/verify/guardrails)

Produces:
- Published images (digest-pinned) via CI

Enforces (via `ameide primitive verify` + CI gates):
- deterministic generation drift checks
- import policy (SDK-only, no cross-primitive runtime imports)
- tests and “no scaffold markers” discipline

### GitOps repo (`ameide-gitops`)

Owns:
- ApplicationSet discovery inputs (`environments/_shared/components/**/component.yaml`)
- Values layering (`sources/values/**`)
- Argo-level smokes (hook Jobs)
- GitOps gate validations (kustomize/helm/policy)

Produces:
- Cluster desired state, reconciled by ArgoCD

Enforces (via GitOps Gate + Argo smokes):
- charts render and validate for supported envs
- image policy (digest pinning rules)
- “front door” / service smokes as Argo hook Jobs

## Unified primitive workflow (end-to-end)

### Phase A — Contract and generation (core repo)

1. Update `.proto` (when needed).
2. Run `buf generate` (SDKs + generated-only glue).
3. Ensure regen-diff is clean (no uncommitted drift).

**Policy references:** `backlog/520-primitives-stack-v2-tdd.md`, `backlog/521d-external-generation-improvements.md`

### Phase B — Scaffold runtime (core repo)

4. Run `ameide primitive scaffold --kind <kind> --name <name> ...` to create the implementation skeleton.
5. Implement the primitive until tests go GREEN.
6. Run `ameide primitive verify --kind <kind> --name <name> --json` and fix all failures (imports, tests, codegen, etc.).

**Policy references:** `backlog/514-primitive-sdk-isolation.md`, `backlog/537-primitive-testing-discipline.md`, `backlog/430-unified-test-infrastructure-v2-target.md`

### Phase C — Publish image (core repo CI)

7. CI builds and publishes the runtime image to `ghcr.io` as an immutable digest.

**Policy references:** `backlog/602-image-pull-policy.md`, `backlog/603-image-pull-policy.md`, `backlog/611-trunk-based-main-and-gitops-environment-promotion.md`

### Phase D — Scaffold GitOps wiring (GitOps repo, CI-owned)

8. Trigger the GitOps wiring PR in `ameide-gitops` using the workflow:

- Workflow: `.github/workflows/scaffold-primitive-gitops.yaml`
- Behavior: generate component + values scaffolding, validate with `helm template`, open PR, merge.

**Policy references:** `backlog/670-gitops-authoritative-write-path-for-scaffolding.md`

### Phase E — Enable in an environment (GitOps repo)

9. In `ameide-gitops`, enable the primitive in the target env and set `image.ref` to the published digest (dev first).
10. Merge PR → ArgoCD reconciles → Argo smokes run (PostSync hook jobs).

**Policy references:** `backlog/648-config-secrets-taxonomy-ci-only-deploy.md`

### Phase F — Promote (GitOps repo, CI-owned)

11. Promote digests forward (dev → staging → production) via the existing promotion workflow.

**Policy references:** `backlog/611-trunk-based-main-and-gitops-environment-promotion.md`

## What is enforced where

### Enforced in `ameide primitive verify` (core repo)

Strongly enforceable (deterministic):
- Codegen drift (regen-diff semantics)
- Import policy (SDK-only; no proto-source imports; no cross-primitive runtime imports)
- Test presence and test execution (430v2); no `AMEIDE_SCAFFOLD` markers
- Primitive shape checks (expected files for each kind)
- Secret hygiene scans (no literals)

Not enforceable as “truth” in core repo:
- GitOps desired state correctness and Argo routing/smokes (those are GitOps repo + cluster concerns)

### Enforced in GitOps Gate (GitOps repo CI)

- `kustomize build` for overlays
- `helm template` + schema validation
- image policy checks (digest pinning)

### Enforced in ArgoCD (cluster)

- Sync + health
- Smokes via Argo hook jobs (HTTP/GRPC checks, secret materialization checks where applicable)

## Evidence / Definition of Done

For a primitive change to be “done”:

- Core repo CI passes (generation + tests + verify gates)
- Image digest is published
- GitOps wiring PR merged (or already existed)
- Env enablement PR merged (dev)
- Argo app is `Synced/Healthy` and smokes succeed

## Cross-references

- CI-only desired-state + config/secrets taxonomy: `backlog/648-config-secrets-taxonomy-ci-only-deploy.md`
- GitOps authoritative write-path (wiring scaffolding): `backlog/670-gitops-authoritative-write-path-for-scaffolding.md`
- TDD outer loop and repo-aligned primitive work: `backlog/520-primitives-stack-v2-tdd.md`
- CLI orchestration boundaries: `backlog/521d-external-generation-improvements.md`
- SDK-only and isolation rules: `backlog/514-primitive-sdk-isolation.md`
- Testing discipline and verify expectations: `backlog/537-primitive-testing-discipline.md`
- Capability DAG (how to parallelize work): `backlog/533-capability-implementation-playbook.md`
- Promotion mechanics: `backlog/611-trunk-based-main-and-gitops-environment-promotion.md`
- Image policy: `backlog/602-image-pull-policy.md`, `backlog/603-image-pull-policy.md`

