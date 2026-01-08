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

## Relationship to existing ApplicationSets (clarification)

This repo already has an “apps-only layer”, but it is **not a separate ApplicationSet**:

- The environment-scoped ApplicationSet (`argocd/applicationsets/ameide.yaml`) generates Applications for *all* domains, including first-party apps (`environments/_shared/components/apps/**/component.yaml`).
- “Apps-only” exists today as a **rollout-phase slice** (apps phases are `610–699` in `backlog/465-applicationset-architecture.md`), which ArgoCD uses for RollingSync ordering.

That “apps-only layer” is useful for:

- reasoning about ordering (infra first, then apps),
- dashboards/queries (`rollout-phase` labels), and
- “deploy only apps” as an *operational choice* (e.g. sync only the apps Applications).

However, PR preview environments still need their own wiring because they require:

- a **different generator** (PR-driven vs environment matrix),
- a **different destination model** (`pr-*` namespaces),
- different **security constraints** (tight AppProject, safe namespace derivation), and
- deterministic **cleanup semantics** on PR close (finalizers + prune + namespace teardown).

## Preview environment delivery patterns

### Pattern A (preferred): ApplicationSet Pull Request generator

Use an ApplicationSet PR generator to create:

- a preview ArgoCD Application per PR, and
- a deterministic destination namespace per PR (created explicitly or via `CreateNamespace=true`).

Requirements:

- Webhook-driven refresh for **managed clusters** (do not rely on slow polling defaults).
- For **local clusters** (developer workstations), polling is typically the only viable trigger because the cluster is not publicly reachable.
- Strict template rendering (missing keys fail).
- Security constraints (admin-owned ApplicationSet; constrained destination/project).

### Pattern B: GitOps PR creates a preview overlay

Publish opens a GitOps PR that adds a PR-scoped overlay (namespace + app values).

This is more explicit and auditable, but has more moving parts and higher operational overhead.

## Mandatory preview environment requirements

## Argo CD semantics (normative; common footguns)

This section makes explicit a few Argo CD behaviors that routinely surprise teams implementing PR previews.

### A) Namespacing contract for deploy bundles (required)

Argo CD does **not** “force” a manifest into `spec.destination.namespace` if the manifest already declares its own namespace.

Normative rules:

- All namespaced Kubernetes resources in the primitive deploy bundle **MUST omit** `.metadata.namespace`.
- The destination namespace is set only via:
  - `Application.spec.destination.namespace`, and/or
  - the deploy tool’s namespace injection (Helm release namespace, Kustomize `namespace:`), **never** via hardcoded `.metadata.namespace` in the bundle.

Rationale: hardcoded namespaces cause preview collisions and can accidentally deploy into stable namespaces.

### A.1) Deploy bundle contract (required)

Preview environments rely on a single, explicit deploy-bundle contract in the PR repo. Make it boring and machine-checkable.

Normative rules:

- The PR repo MUST provide a **Kustomize** bundle at `deploy/preview/` containing `deploy/preview/kustomization.yaml`.
- The preview ApplicationSet MUST set `source.path: deploy/preview` and `targetRevision: {{ .head_sha }}` so the preview deploys exactly what was reviewed.
- The bundle MUST include only namespaced resources and MUST comply with the namespacing contract above (no `.metadata.namespace` hardcoding).
- The bundle MUST NOT include cluster-scoped resources (CRDs, ClusterRoles, ValidatingWebhookConfiguration, etc.). Those are infra.
- The bundle MUST NOT assume a webhook-driven trigger for local clusters; local uses polling.

### B) Namespace creation vs namespace ownership/deletion (required)

Creating a namespace and *owning it for cleanup* are not the same thing.

Normative choice (pick one and standardize):

- **Option 1 (recommended for previews): explicit Namespace manifest**
  - The preview baselines (GitOps source) include an explicit `Namespace` manifest for the PR namespace.
  - The preview Application prunes it on deletion, so the namespace is deleted (and all namespaced objects are collected).
  - The previews AppProject must allow `Namespace` creation only for the preview naming pattern.

- **Option 2: `CreateNamespace=true` + `managedNamespaceMetadata`**
  - The preview Application uses `syncOptions: ["CreateNamespace=true"]` and sets `managedNamespaceMetadata` to enforce labels/annotations.
  - If you want Argo CD to treat the Namespace as managed, you must also ensure tracking is configured safely (this can accidentally “adopt” shared namespaces; use only if you understand the risks).

**Repo constraint (ameide-gitops):** our AppProject CRD currently supports `clusterResourceWhitelist` by `{group, kind}` only (no name globs). That makes “previews own Namespace resources” unsafe, so the implementation uses **Option 2** (namespace created by Argo CD + labels via `managedNamespaceMetadata`) and treats namespace deletion as an operational concern for now.

**Implementation closure (namespace deletion):** because previews do not own `Namespace` resources, we delete leaked/orphaned preview namespaces using a janitor job:

- A cluster-scoped `CronJob` deletes namespaces that:
  - have label `ameide.io/managed-by=preview`
  - match the `pr-` prefix
  - are older than a retention window, and
  - are not referenced by any `Application.spec.destination.namespace` in the `argocd` namespace.

This provides deterministic cleanup even when the namespace was created via `CreateNamespace=true`.

### C) AppProject restrictions (required)

Preview apps must be constrained at the Argo CD Project layer.

Normative rules:

- `AppProject.spec.destinations` MUST restrict destination namespaces to a preview glob (e.g. `pr-*`) and MUST NOT allow `argocd` or other control-plane namespaces.
- `AppProject.spec.clusterResourceWhitelist` MUST be empty **unless** you intentionally choose to manage the preview Namespace.
  - If your AppProject CRD supports name-globs, allow only: group `""`, kind `Namespace`, name `pr-*`.
  - If it does **not** (current state in `ameide-gitops`), keep it empty and use `CreateNamespace=true` + `managedNamespaceMetadata`.

### D) Sync option hardening (required)

Preview Applications must fail fast when they accidentally contain infra resources or collide with stable apps.

Normative rules:

- Preview Applications MUST set the following sync options:
  - `FailOnSharedResource=true` (do not “share” resources with stable Applications)
  - `PruneLast=true` (reduce transient breakage during sync)
  - `PrunePropagationPolicy=background` (or `foreground`; pick one and standardize)

### E) Multiple sources: precedence and collision policy (required)

Argo CD applies multiple sources in order. If two sources render the *same resource identity* (group/kind/name/namespace), the later source effectively “wins” and Argo CD will warn.

Normative rules:

- The GitOps “bindings/baselines” source MUST be ordered **after** the primitive deploy bundle source.
- Overlapping resources across sources are either:
  - prohibited, or
  - allowed only for an explicit allowlist of kinds (e.g. ConfigMaps) with an explicit rationale.

### F) PR generator webhook behavior (required)

PR generators poll by default. Webhooks are the vendor-supported way to make previews feel fast.

Normative rules:

- Expose the ApplicationSet controller webhook endpoint (`/api/webhook`) as a first-class ingress/route.
- Keep an explicit `requeueAfterSeconds` configured even when webhooks are enabled (webhooks fail; polling is the safety net).
- Document and test the exact PR actions you rely on (opened/synchronize/closed/…); treat this as version-sensitive behavior.

**Local dev note:** local clusters run on developer workstations and are not publicly reachable. Do not require webhooks for local; rely on polling (short `requeueAfterSeconds`) and/or manual refresh for inner-loop work.

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

## Feasibility findings (current state of `ameide-gitops`)

This section summarizes what is already in place vs what must be added to implement Pattern A safely.

### What is already feasible / exists

- **ArgoCD + ApplicationSet controller are installed and used** (stable env matrix + cluster appset already exist).
- **PR generator capability is available** in the installed ApplicationSet CRD (supports GitHub PR generator with `tokenRef` / `appSecretName`).
- **Wildcard DNS is already provisioned per environment** (e.g. `*.dev.ameide.io`), so preview hostnames under an environment zone are viable without per-PR DNS automation.
- **RollingSync separation already exists**: infra vs apps is modeled via `rollout-phase`, and primitives are already part of the “apps” domain in stable environments.

### Gaps / required changes

- **Webhook-driven refresh**: required for low-latency previews on managed clusters; local clusters will generally rely on polling.
- **Preview AppProject**: required to constrain destinations and resource kinds.
- **Preview baselines**: required for namespace-scoped defaults (NetworkPolicies, limits/quotas, etc.).

### High-risk area: secrets for dynamic namespaces

The current secrets model is environment-scoped and intentionally conservative:

- Vault Kubernetes auth roles are bound to an explicit namespace allowlist (see the Vault bootstrap role bindings).
- As a result, a brand-new `pr-*` namespace will not automatically be able to use the same SecretStore/ExternalSecret pattern unless we introduce a preview-safe auth/policy model.

If previews must deploy primitives that need secrets (GHCR pulls, app config secrets, etc.), preview secrets handling is the main design decision and the main blocker.

## Implementation progress (2026-01)

This section records what is implemented in `ameide-gitops` as of 2026-01.

### Landed in `main`

- Merged via `ameide-gitops#98` and `ameide-gitops#99`.
- **Preview AppProject**: `argocd/projects/previews.yaml` (restricted to `pr-*` namespaces, no cluster-scoped resources).
- **PR generator ApplicationSet**: `argocd/applicationsets/preview.yaml` (GitHub PR generator gated by `preview` label, `requeueAfterSeconds` set, two-source Applications).
- **Preview baselines**: `argocd/previews/baseline/` (initial baseline: cross-environment ingress isolation `NetworkPolicy`).
- **Preview namespace janitor**: `environments/_shared/components/cluster/configs/preview-janitor/component.yaml` + `sources/values/_shared/cluster/preview-janitor.yaml`
  - Deletes orphaned `pr-*` namespaces labeled `ameide.io/managed-by=preview` after a retention window.
- **Local overlay behavior**: `argocd/overlays/local/kustomization.yaml` patches preview namespaces to `ameide.io/environment=local` (the baseline policy allows `dev` and `local`).
- **Cluster-gateway webhook route (managed clusters)**: `sources/charts/cluster/gateway/templates/httproute-applicationset-webhook.yaml`
  - Exposes a stable external path (`/api/applicationset-webhook`) and rewrites to the controller’s `/api/webhook` endpoint.
  - Local clusters typically cannot use this because they are not publicly reachable.
- **Secrets wiring**
  - `sources/values/_shared/foundation/foundation-vault-bootstrap.yaml` seeds `argocd-webhook-github-secret` (stable value generated if absent).
  - `sources/values/env/{dev,local}/foundation/foundation-vault-bootstrap.yaml` enables a preview-safe Vault role `external-secrets-previews` bound to all namespaces, with a narrowly scoped read policy (default: `secret/data/ghcr-*`).
  - `sources/values/_shared/cluster/vault-secrets-argocd.yaml`:
    - syncs `webhook.github.secret` into `argocd-secret`
    - syncs `Secret/argocd-pr-generator-github-token` for the PR generator (default Vault key `ghcr-token`)

### Still required to complete end-to-end

- **Managed-cluster connectivity + public webhook**: ensure the managed cluster’s Argo CD hostname is reachable from GitHub and register the webhook using the `webhook.github.secret` value.
- **Deploy bundle contract**: the PR repo must include a deploy bundle at the path used by the preview ApplicationSet (currently `deploy/preview/`).

### Important constraint: NetworkPolicy + “reuse infra”

This repo enforces cross-environment isolation via namespace labels and NetworkPolicies. In practice:

- If a preview namespace is meant to reuse “dev infra” services (databases, brokers, etc.), it must be able to talk to dev namespaces.
- With strict namespaceSelector-based policies, that typically means preview namespaces must carry the same environment label as the infra they consume (e.g. `ameide.io/environment=dev`) or provide an explicit allowlist/exception.

## Proposed implementation (Pattern A: PR generator)

This is a concrete plan for implementing previews in a vendor-correct way while keeping the “two-source” contract from this doc.

### 0) Decisions (must be explicit)

- **Base environment** for previews (recommended: `dev`): previews are “PR namespaces that reuse dev infra”.
- **Namespace naming** (deterministic + safe): e.g. `pr-<num>-<repo>` (current implementation uses `pr-ameide-<pr-number>`).
- **Namespace ownership**: current implementation uses `CreateNamespace=true` + `managedNamespaceMetadata` (see the AppProject constraint above).
- **Secrets posture**:
  - “No-secrets previews” (only for a limited subset of primitives), or
  - a preview-safe Vault/ESO strategy (recommended; see below).

### 1) Add a dedicated AppProject for previews

Create a new AppProject (admin-owned) with:

- source repo allowlist: **only** the primitive repo + this GitOps repo
- cluster resource whitelist: empty / minimal (no CRDs, no RBAC cluster roles)
- namespace-scoped resource whitelist: restricted to the primitive CRDs + basic K8s objects needed for previews
- destination server locked to `https://kubernetes.default.svc`
- destination namespaces restricted as tightly as ArgoCD supports; where prefix constraints are not expressible, enforce namespace determinism in the ApplicationSet template and rely on resource whitelisting to limit blast radius

### 2) Expose the ApplicationSet webhook endpoint (Gateway API)

To avoid slow polling and make PR previews feel responsive:

- expose the `argocd-applicationset-controller` service over HTTPS at a dedicated hostname (e.g. `argocd-appset-webhook.<env-zone>`)
- route `/api/webhook` to the controller service port (default `7000`)
- configure the SCM (GitHub App / webhook) to call this endpoint on PR events

**Local dev note:** local clusters run on developer workstations and are not publicly reachable. Prefer polling (`requeueAfterSeconds`) for local rather than requiring an external webhook.

### 3) Create a `preview` ApplicationSet using the PR generator

Add a new ApplicationSet dedicated to previews:

- generator: `pullRequest.github` (admin-owned)
- template:
  - `metadata.name`: deterministic and bounded (do not include attacker-controlled free text)
  - `spec.project`: fixed to the previews AppProject (not templated from PR data)
  - `destination.namespace`: derived from PR number using a fixed prefix (e.g. `pr-{{ .number }}`)
  - `finalizers`: `resources-finalizer.argocd.argoproj.io`
  - `syncPolicy.automated`: prune + selfHeal
  - `syncOptions`: include `FailOnSharedResource=true`, `PruneLast=true`, and a standardized `PrunePropagationPolicy`

### 4) Standardize on exactly two sources per preview Application

Each generated preview Application should have exactly two sources:

1) **Primitive repo PR ref**: a fixed deploy-bundle path in the PR branch/commit (for example, `primitives/<kind>/<name>/deploy/` or a single umbrella `deploy/preview/` that the publish step maintains).
2) **GitOps repo main ref**: preview baselines + environment bindings (endpoints, topic names, SecretStores/ExternalSecrets, etc.).

The primitive repo “publish” step is responsible for making the deploy bundle deterministic (including digest pinning).

### 5) Digest pinning mechanism (preferred)

Follow the preferred mechanism in this doc:

- publish writes the resolved image digest into a lock file in the primitive repo PR branch (e.g. `primitives/<kind>/<name>/deploy/image.ref.yaml`)
- the preview Application deploys that PR commit, so the deployed image is immutable and reviewable

### 6) Preview-safe secrets strategy (recommended approach)

To support secrets in `pr-*` namespaces without expanding the Vault auth allowlist indefinitely:

- introduce a **dedicated preview credential path** in Vault (e.g. `secret/data/previews/dev/pr-<num>/*`)
- introduce a **dedicated Vault Kubernetes auth role** bound to a stable ServiceAccount in a stable namespace (e.g. `external-secrets`), rather than binding to every `pr-*` namespace
- use an **(optional) ClusterSecretStore** for preview namespaces so ExternalSecrets in `pr-*` can reference it without requiring per-namespace Vault auth bindings

At minimum, previews usually need:

- `ghcr-pull` (image pull secret), and
- any per-primitive runtime secrets required by the deploy bundle.

### 7) Baselines for preview namespaces

Create a GitOps-owned “preview baselines” package that includes:

- `Namespace` manifest with required labels (including the chosen “base environment” label to satisfy NetworkPolicy constraints)
- minimal NetworkPolicies/quotas/limit ranges for isolation and cost control
- (optionally) `gateway-access: allowed` labeling if the preview needs HTTPRoute/GRPCRoute attachment
- pull secret plumbing (e.g., ExternalSecret that materializes `ghcr-pull`)

## Verification plan (implementation gate)

### Static verification (repo-only)

- Render the GitOps overlay (kustomize) and ensure the new resources are syntactically valid and included in the ArgoCD self-managed set.
- Validate the preview ApplicationSet template renders deterministically (missing keys fail, names are bounded, no attacker-controlled strings in destinations).

### End-to-end verification (cluster)

1) **Webhook refresh**
   - create/update a PR in the primitive repo and confirm the preview Application appears quickly (no long polling delay)
2) **Namespace + baselines**
   - confirm the preview namespace exists, has the expected labels/annotations, and baseline policies are present
3) **Scope correctness**
   - confirm preview Application deploys only primitive-layer resources (no infra charts/operators)
4) **Connectivity to shared infra**
   - confirm preview workloads can reach the intended shared infra endpoints (e.g. dev database/broker) under the enforced NetworkPolicy posture
5) **Secrets**
   - confirm required secrets (at least `ghcr-pull`) exist in the preview namespace and pods can pull images
6) **Cleanup**
   - close the PR and confirm the preview Application is deleted, resources are pruned, and the namespace is removed (no leaks)
7) **Security regression**
   - verify the previews AppProject prevents cluster-scoped resources and prevents source repos outside the allowlist

## What to track in this backlog (checklist)

- [ ] Add Argo CD semantics section (namespacing, namespace ownership, sync options, multi-source precedence).
- [ ] Define the preview namespace naming scheme (deterministic and safe).
- [ ] Choose namespace creation strategy (`CreateNamespace=true` vs explicit Namespace manifest).
- [ ] Decide where preview baselines live (GitOps repo path + ownership).
- [ ] Specify the ArgoCD Project restrictions for preview apps.
- [ ] Define deletion/prune/finalizer policy so previews never leak.
- [ ] Decide the digest injection mechanism (primitive lock file vs GitOps overlay).
- [ ] Standardize on 2 sources for preview Applications.
- [ ] Standardize multi-source collision/override policy.
- [ ] Specify PR webhook refresh details (`/api/webhook`, polling safety net).
- [ ] Document stateful dependency strategy (shared DB + schema, etc.).
