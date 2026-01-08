# 465 – ApplicationSet Architecture: PR Preview Environments

**Created**: 2026-01-08
**Updated**: 2026-01-08

This document extends `backlog/465-applicationset-architecture.md` with a **vendor-correct, low-guesswork** pattern for **PR-scoped preview environments** (namespaces + Applications) that deploy **only primitive workloads** while reusing shared infrastructure (databases/Temporal/Kafka/observability/etc).

Crossreference: the agentic workflow that drives this from the developer/agent side is `backlog/520-primitives-stack-v2-agentic-process.md`.

## Problem statement

We want a PR to produce a preview deployment that is:

- **Fast** (no long polling delays).
- **Safe** (PR generator cannot escalate privileges).
- **Clean** (teardown on PR close; no leaked namespaces/resources).
- **Scoped** (deploys only primitive workloads; does not redeploy shared infrastructure).
- **Deterministic** (digest-pinned images; no “latest tag” ambiguity).

## Key concept: “infra apps” vs “preview apps”

ArgoCD reconciles **only what an Application declares**. Preview environments avoid “redeploy infra” by separating ownership:

- **Infrastructure Applications** (stable): cluster-scoped + environment-scoped infra (operators, CRDs, CNPG/Temporal/Kafka, observability stack, gateways). These remain on `main` and deploy to stable namespaces.
- **Preview Applications** (ephemeral): PR namespace + primitive CR instances + per-namespace bindings (ConfigMaps/ExternalSecrets/etc) that *reference* the existing infra.

Preview Apps must not include infra manifests. If they do, the preview loop becomes slow and dangerous.

## Preview environment delivery patterns

### Pattern A (preferred): ApplicationSet Pull Request generator

Use an ApplicationSet PR generator to create:

- a preview ArgoCD Application per PR, and
- a deterministic destination namespace per PR (created explicitly or via `CreateNamespace=true`).

Requirements:

- Webhook-driven refresh (do not rely on slow polling defaults).
- Strict template rendering (missing keys fail).
- Security constraints (admin-owned ApplicationSet; constrained destination/project).

### Pattern B: GitOps PR creates a preview overlay

Publish opens a GitOps PR that adds a PR-scoped overlay (namespace + app values).

This is more explicit and auditable, but has more moving parts and higher operational overhead.

## Mandatory preview environment requirements

### 1) Namespace creation strategy (choose one and standardize)

Preview envs must not fail sync because the namespace doesn’t exist.

- **Option 1**: Application `syncOptions: ["CreateNamespace=true"]` and include baseline namespace policies (quotas, network policies, limit ranges) in the preview stack.
- **Option 2**: Preview overlay includes a `Namespace` manifest and manages it intentionally.

### 2) Cleanup + deletion semantics (no leaks)

Preview environments must delete everything they own when a PR closes.

Required:

- Preview Applications enable **prune**.
- Preview Applications use ArgoCD’s **resources finalizer** so deletion cascades to managed resources.
- ApplicationSet preview policy must not preserve resources on deletion.

Namespace deletion rule:

- If the preview namespace is created by ArgoCD, it should also be deleted by ArgoCD (or by the preview teardown automation) to avoid leaks.

### 3) Security constraints (PR generators are privileged)

Required:

- ApplicationSets that generate Applications from PRs are **admin-owned** and reviewed like infra.
- Generated Applications are constrained to a safe ArgoCD **Project** (do not template `spec.project` from PR data).
- Destination namespace is constrained to a safe pattern (e.g. `pr-<num>-<repo>`), not attacker-controlled free text.
- Fork/untrusted PRs are not allowed to deploy previews unless explicitly allowlisted.

### 4) Avoid redeploying infrastructure (preview scope)

Required:

- Preview Applications deploy only primitive-layer resources (CR instances, per-namespace bindings, optional baseline policies).
- Any shared infra (Temporal/Kafka/Postgres/observability) is owned by separate Applications and is referenced only via stable in-cluster endpoints and secret refs.

### 5) Two-source Application composition (avoid copy/paste)

To keep “deploy intent next to code” without copying manifests into GitOps, standardize on **exactly two sources** for preview Applications:

1. **Primitive repo PR ref**: `primitives/<kind>/<name>/deploy/` (deploy bundle).
2. **GitOps repo main ref**: preview namespace baselines + environment bindings (endpoints/secrets refs/topic bindings).

If more than two sources are needed, consolidate (treat as a smell).

### 6) Digest pinning injection (no tribal glue)

Preview deployments must deploy immutable digests.

Preferred:

- Publish updates a deterministic lock file in the primitive repo PR branch (e.g., `primitives/<kind>/<name>/deploy/image.ref.yaml`), and the preview Application deploys that commit.

Fallback:

- Publish opens a GitOps PR that provides a PR-scoped overlay containing the digest-pinned image ref.

### 7) Stateful dependencies (DBs) for previews

Preview envs should reuse infra but isolate state:

- Preferred: shared database infrastructure with per-preview **schema/db/user** provisioning (declared and managed; no runtime “magic defaults”).
- Alternative: ephemeral DB per preview (costly; use only when required).

## What to track in this backlog (checklist)

- [ ] Define the preview namespace naming scheme (deterministic and safe).
- [ ] Choose namespace creation strategy (`CreateNamespace=true` vs explicit Namespace manifest).
- [ ] Decide where preview baselines live (GitOps repo path + ownership).
- [ ] Specify the ArgoCD Project restrictions for preview apps.
- [ ] Define deletion/prune/finalizer policy so previews never leak.
- [ ] Decide the digest injection mechanism (primitive lock file vs GitOps overlay).
- [ ] Standardize on 2 sources for preview Applications.
- [ ] Document stateful dependency strategy (shared DB + schema, etc.).

