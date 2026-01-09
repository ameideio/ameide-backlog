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

### 1) Scaffold (code only; no deploy manifests)

Command:

- `ameide primitive scaffold --kind <kind> --name <name>`

Scaffold MUST generate:

- Runtime skeleton under `primitives/<kind>/<name>/` (implementation-owned extension points included).
- Deterministic generated artifacts (if applicable: e.g. Process BPMN compile outputs).
- No Kubernetes manifests, Helm charts, or deploy bundles. All deployment intent lives in GitOps.

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

Preview environments MUST not leak resources when a PR closes.

- Preview Applications MUST have pruning enabled and MUST delete managed resources on Application deletion (finalizer-based cascading deletion).
- Preview namespaces MUST NOT be deleted automatically by ArgoCD. Namespace deletion MUST be performed by a cluster-owned janitor that deletes only preview-managed namespaces after verifying no ArgoCD Application still targets them.

### D) Namespace creation (no “namespace missing” failures)

The preview workflow MUST define namespace creation behavior; ArgoCD will not sync to a namespace that does not exist unless instructed.

Preview namespaces MUST be created by a GitOps-owned system lane that applies an explicit `Namespace` manifest with deterministic labels.

### E) Preview trust model (PR inputs are not deploy inputs)

Preview deploy MUST be defined entirely by GitOps. PR branches MUST NOT provide Kubernetes manifests, Helm charts, or values files that render cluster objects.

The only PR-derived values used by preview deploy are:

- `PR_NUMBER` (namespace/app suffix)
- `HEAD_SHA` (image selection)

### F) How digest pinning enters the PR preview environment (make it explicit)

Digest pinning MUST be injected deterministically; otherwise “preview deploy” becomes tribal glue.

- CI publish MUST ensure that, for every previewed component image, `ghcr.io/<org>/<image>:<HEAD_SHA>` exists in the registry.
  - If the component changed in the PR: build + push `:<HEAD_SHA>`.
  - If the component did not change: copy/retag a trusted baseline digest (from `:main`) to `:<HEAD_SHA>` (no rebuild).
- Preview deploy MUST select images using only `HEAD_SHA`. It MUST NOT read PR-branch artifacts such as lock files.

### G) Supply chain hardening for agentic publish

Digest pinning is necessary but not sufficient for agentic automation.

- Publish MUST produce signed images and attach provenance and an SBOM to the OCI artifact.

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

1. Build images for changed components.
2. Push images to GHCR.
3. Ensure `:<HEAD_SHA>` exists for all previewed images by retagging unchanged components from `:main`.
4. Sign images and attach provenance + SBOM.

## Responsibility split (what owns what)

- **Code repo (`ameide`)**: runtime implementation, generated artifacts, and container images.
- **GitOps repo (`ameide-gitops`)**: Kubernetes manifests/charts/values, per-environment bindings, preview environment definitions, and namespace lifecycle wiring.
- **Operators**: reconcile CRDs and inject config.
