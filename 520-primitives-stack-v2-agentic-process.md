# 520 — Primitives Stack v2: Agentic Scaffold → Implement → Publish → Preview Deploy

This document defines the **minimum-guesswork**, agent-friendly workflow for taking a primitive from “new scaffold” to a **PR-scoped preview environment** deployed by ArgoCD, while remaining compatible with the v2 stack principles in `backlog/520-primitives-stack-v2.md`.

The goal is to make the “happy path” **mechanical and reviewable**: each step produces deterministic artifacts, and every configuration value has exactly one authority (cluster-derived vs GitOps-provisioned vs request-provisioned).

Crossreference (ArgoCD wiring details): `backlog/465-applicationset-architecture-preview-envs.md`.

## Goals

- A single, repeatable workflow an agent can execute end-to-end with minimal interpretation.
- **No GitOps repo mutation during scaffold**; any GitOps changes (when needed) happen only as an explicit publish step (PR-based).
- Digest-pinned images everywhere (see `backlog/602-image-pull-policy.md`).
- Preview environments per PR (namespace + ArgoCD apps) to validate real deploy behavior early.

## Non-goals

- Replacing ArgoCD/GitOps with “push to cluster” workflows.
- Encoding environment endpoints/credentials inside protos or inside scaffolded runtime defaults.
- Introducing bespoke flags for per-primitive behavior; prefer checked-in config files.

## Principle recap (normative)

1. **Single authority config**: a runtime value MUST be either cluster-derived, GitOps/operator-provisioned, or request-provisioned; no fallback/override chains. (`backlog/520-primitives-stack-v2.md`)
2. **No local registries**: all images are pulled from GHCR and referenced by digest. (`backlog/602-image-pull-policy.md`)
3. **Generated vs implementation-owned boundary**: generators clobber only generated outputs; humans/agents own the implementation extension points. (`backlog/520-primitives-stack-v2.md`)

## The workflow (agentic outer loop)

### 1) Scaffold (code + deploy bundle; no GitOps PR)

Command:

- `ameide primitive scaffold --kind <kind> --name <name> [--proto-path <path>] [--include-test-harness]`

Scaffold MUST generate:

- Runtime skeleton under `primitives/<kind>/<name>/` (implementation-owned extension points included).
- Deterministic generated artifacts (if applicable: e.g. Process BPMN compile outputs).
- A checked-in **deploy bundle** that describes how this primitive is intended to be deployed by ArgoCD, but **does not require touching the GitOps repo**.

**Deploy bundle location (required):**

- `primitives/<kind>/<name>/deploy/`

**Deploy bundle contents (required, minimal):**

- A workload definition template (CRD instance or Helm values snippet) with:
  - `spec.image` sourced from an **image ref input** (to be filled with a digest during publish).
  - required runtime env/secret refs (placeholders allowed; names must be explicit).
  - stable labels/annotations for discovery (primitive kind/name/version).
- A single, well-known **image lock input file** that publish can update deterministically (no templating guesswork), for example:
  - `primitives/<kind>/<name>/deploy/image.ref.yaml` with `image.ref: ghcr.io/...@sha256:...` (empty during scaffold).
- A “preview overlay” template that can be parameterized by a PR number (namespace name, application name suffix).

Notes:

- “Scaffold also GitOps concerns” means: scaffold the *intended ArgoCD deploy shape* into the repo (deploy bundle), not mutate `ameide-gitops`.
- The deploy bundle MUST NOT contain environment-specific endpoints/credentials; those belong to GitOps/operator-provisioned config in the environment overlay.

## Vendor-correct preview environment requirements (ArgoCD/Kubernetes)

Preview environments are only “minimum guesswork” if we specify the operational semantics that ArgoCD/Kubernetes enforce.

### A) Low-latency PR updates (webhooks, not polling)

If preview environments are driven by an ApplicationSet PR generator, the system MUST be configured for webhook-driven refresh (not “wait for the next poll”, whose defaults are slow). Otherwise updates will be delayed and the loop feels flaky.

Required:

- ApplicationSet PR generator webhook server enabled for the SCM provider used.
- Templates rendered with strict missing-key behavior (fail hard if a template expects a value that is absent).

### B) PR generator security (admin-owned, constrained)

ApplicationSets that generate Applications from PRs MUST be treated as privileged and MUST be admin-owned.

Required:

- Generated Applications MUST be constrained to a safe ArgoCD Project (do not template `project` from PR inputs).
- Destination namespaces MUST be constrained (e.g., a fixed prefix like `pr-<num>-...`) and must not be attacker-controlled strings.
- PRs from forks/untrusted sources MUST NOT create preview environments unless explicitly allowlisted.

### C) Cleanup semantics (finalizers + pruning)

Preview environments MUST be deleted automatically when the PR is closed/merged and MUST not leak resources.

Required:

- Preview Applications MUST have pruning enabled and must delete managed resources on Application deletion.
- Preview Applications MUST use cascading deletion semantics (finalizer-based; include the ArgoCD resources finalizer), and ApplicationSet deletion policy MUST NOT preserve resources for previews.

Namespace deletion rule (choose one and standardize):

- Preferred: the preview namespace is managed as part of the preview environment and is deleted when the preview is deleted (requires intentional namespace management and avoids leaks).
- Alternative: namespace is pre-created/managed by separate cluster automation; preview Apps deploy only into an existing namespace (still requires a deterministic cleanup story).

### D) Namespace creation (no “namespace missing” failures)

The preview workflow MUST define namespace creation behavior; ArgoCD will not sync to a namespace that does not exist unless instructed.

Required (choose one and standardize):

- Preview Applications set `CreateNamespace=true`, and the environment includes baseline namespace policies (quota/network policy/limits), OR
- The preview overlay includes an explicit `Namespace` manifest and manages it intentionally.

### E) Combining “deploy bundle in primitive repo” with “env bindings in GitOps”

To avoid copy/paste and drift, preview Applications SHOULD use ArgoCD’s multiple sources and standardize on exactly two sources:

1. **Primitive repo PR ref**: the deploy bundle under `primitives/<kind>/<name>/deploy/`.
2. **GitOps repo main ref**: environment bindings (endpoints/secrets refs/broker bindings) + preview namespace baselines.

If more than two sources are required, treat it as a design smell and consolidate.

### F) How digest pinning enters the PR preview environment (make it explicit)

Digest pinning MUST be injected deterministically; otherwise “preview deploy” becomes tribal glue.

Trust model note:

- PR branches are attacker-controlled inputs. Preview deploy MUST NOT consume arbitrary PR-branch “lock files” as a deploy input unless the cluster-side trust model explicitly allows it.

Preferred mechanism (closed trust model; no PR-branch deploy inputs):

- CI publish ensures that, for every previewed component image, `ghcr.io/<org>/<image>:<HEAD_SHA>` exists in the registry.
  - If the component changed in the PR: build + push that tag.
  - If the component did not change: copy/retag a trusted baseline digest (typically from `:main`) to that tag (no rebuild).
- Preview deploy selects images using only `HEAD_SHA` (and optional PR number for human tags), without reading PR-branch artifacts.

Alternative mechanism (no GitOps mutation per PR):

- Publish writes the resolved digest into the deploy bundle lock file (e.g., `primitives/<kind>/<name>/deploy/image.ref.yaml`) on the primitive repo PR branch, and the PR generator deploys that exact commit.

Fallback mechanism (GitOps PR per preview):

- Publish opens a GitOps PR adding a PR-scoped overlay containing the digest-pinned image ref, and the preview Application references that overlay.

Deprecated mechanism (monorepo / repo-wide PR-branch contract):

- CI (or publish) writes a single repo-wide lock file (e.g., `images.lock.yaml`) that maps logical image names to digest-pinned refs (`ghcr.io/...@sha256:...`) and commits it back to the PR branch.
- This is deterministic, but it violates the “PR_NUMBER + HEAD_SHA only” trust model unless the cluster explicitly treats PR branch files as trusted inputs.

Historical note:

- This doc started from a per-primitive deploy bundle model (`primitives/<kind>/<name>/deploy/image.ref.yaml`). A repo-wide lock file is the generalized form for repos that publish multiple images (services/operators/primitives) from one PR.
- In the Ameide monorepo, the evolved direction for PR previews is to avoid consuming PR-branch lockfiles and instead guarantee `:<HEAD_SHA>` tags exist for every image, with unchanged images retagged from a trusted baseline.

### G) Supply chain hardening for agentic publish

Digest pinning is necessary but not sufficient for agentic automation.

Recommended (v2 posture):

- Publish generates an SBOM and signs the image + attestations.
- Clusters MAY enforce signature verification via admission for preview/prod parity.

### 2) Implement + test (repo-only)

Agent/human work:

- Implement the implementation-owned surfaces (handlers/activities/router bindings, etc.).
- Run deterministic verification:
  - `ameide primitive verify --kind <kind> --name <name> --mode repo`
  - language tests (`go test`, `pytest`, `npm test`) as applicable

Minimum gate:

- Repo verifies cleanly (no drift between design inputs and generated outputs).
- Tests pass locally.

### 3) Publish (build + push + open PR(s) for preview deploy)

This step is responsible for connecting “repo artifacts” to “cluster reality” without guesswork.

Publish MUST:

1. Build the runtime image.
2. Push to GHCR.
3. Resolve the immutable image digest.
4. Prepare the ArgoCD preview deployment and open PR(s) that will cause ArgoCD to deploy it.

There are two acceptable PR-based preview patterns:

**Pattern A (preferred): ArgoCD ApplicationSet PR generator**

- GitOps repo contains an ApplicationSet that discovers PRs and creates:
  - a PR namespace (or enables namespace auto-creation), and
  - an app that deploys the primitive from the PR ref.
- Publish updates the PR branch so the deploy bundle references the **digest-pinned** image (source of truth is the PR commit ArgoCD deploys).
- The generated Application uses **two sources**: primitive repo PR ref (deploy bundle) + GitOps repo main (env bindings/baselines).

**Pattern B: GitOps PR adds a preview overlay**

- Publish opens a PR to the GitOps repo that adds:
  - namespace definition (or relies on an existing namespace generator)
  - application component/values for the primitive
  - the digest-pinned image ref

### Recommended CLI surface (minimum flags)

To minimize flags while keeping behavior explicit:

- `ameide primitive publish --kind <kind> --name <name>` builds/pushes.
- A follow-on command (or submode) owns preview wiring:
  - `ameide dev publish --kind <kind> --name <name>` (recommended name), which:
    - builds/pushes
    - resolves digest
    - updates the deploy bundle image lock (commit to the PR branch), and/or opens the GitOps PR for preview deploy using the repo’s `deploy/` bundle

Inputs SHOULD come from checked-in files under the primitive (not flags), for example:

- `primitives/<kind>/<name>/deploy/ameide.deploy.yaml`:
  - image repository (without tag)
  - which deploy templates to render/apply
  - required secret refs by name (declared, not defaulted)

This keeps the agent path deterministic:

- the primitive repo PR contains both code and the intended deploy shape,
- the GitOps PR contains only environment wiring + digest pinning.

## Responsibility split (what owns what)

- **Primitive repo (this repo)**:
  - runtime implementation
  - generated artifacts
  - deploy bundle templates under `primitives/<kind>/<name>/deploy/`
- **GitOps repo (`ameide-gitops`)**:
  - per-environment bindings (addresses, secrets, topic names)
  - preview environment definitions (namespace/apps), via PR
- **Operators**:
  - reconcile CRDs and inject config
  - never embed business logic

## Why this is “standard practice”

- PR-scoped preview environments are common in GitOps: they surface integration issues early and provide a reproducible review artifact (“this PR deploys X into namespace Y”).
- Keeping deploy intent next to code (deploy bundle in the primitive repo) is a common technique to avoid “tribal knowledge” and reduce drift between implementation and deployment.
