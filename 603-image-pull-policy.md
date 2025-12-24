# 603: Image Pull Policy — Refactoring Plan (GitOps + Operators)

**Status:** Ready to implement  
**Owner:** Platform SRE / GitOps  
**Scope:** Implement the target state in `backlog/602-image-pull-policy.md`  
**Related:** `backlog/602-image-pull-policy.md`, `backlog/495-ameide-operators.md`, `backlog/503-operators-helm-chart.md`, `backlog/450-argocd-service-issues-inventory.md`, `backlog/519-gitops-fleet-policy-hardening.md`, `backlog/456-ghcr-mirror.md`

## Goal

Make “Git commit → deployed artifact” deterministic in GitOps-managed environments by:

- eliminating reliance on mutable tags for rollouts
- pinning by digest (or immutable SHA tag + digest)
- making `imagePullPolicy` usage consistent and intentional (not accidental drift)

## Current state (known reality)

- Some workloads still use floating tags (especially during rapid iteration).
- `imagePullPolicy` is currently used to force refreshes when tags move, but it does not trigger rollouts on its own.
- Argo can surface drift from Kubernetes-defaulted fields when diff mode changes.

## Refactoring work packages

### WP-1: Primitive CRD contract completeness (repo: `ameide`)

**Target:** all primitive CRDs support `spec.imagePullPolicy`, and operators apply it to generated Deployments/Jobs.

**DoD**
- `spec.imagePullPolicy` exists on Domain/Process/Agent/UISurface/Projection/Integration CRDs.
- Operators copy it into all generated pod templates (Deployments + Jobs, including migrations).
- Tests: at least one controller test per kind asserts `imagePullPolicy` propagation.

### WP-2: GitOps values contract for primitives (repo: `ameide-gitops`)

**Target:** primitives are configured via a single predictable value key, and rendering never relies on brittle template helpers inside `tpl`.

**DoD**
- A single “fleet contract” value exists for primitives (example: `global.ameide.primitives.imagePullPolicy`).
- All primitive CR manifests render `spec.imagePullPolicy` from that value.
- No `dig`/`get` usage that fails when `.Values` is not a plain map in `tpl` contexts.
- Docs updated: `backlog/495-ameide-operators.md` and `backlog/520-primitives-stack-v2-tdd.md` reference 602/603.

### WP-3: Digest pinning support in GitOps values (repo: `ameide-gitops`)

**Target:** every GitOps-managed environment can deploy primitives by digest.

**Approach**
- Prefer setting the primitive CR’s `spec.image` to `repo@sha256:<digest>` directly.
- Values should support either:
  - `image.digest` (preferred), or
  - `image.tag` + `image.digest` (readability), or
  - `image.tag` only (explicitly transitional and tracked).

**DoD**
- Primitive value templates support digests cleanly.
- Staging/prod overlays require digests (policy enforcement in CI; see WP-5).

### WP-4: Continuous update mechanism for the “fast-moving” environment

We need one environment that moves quickly without humans manually restarting pods.

Pick one:

- **Option A (PR-based):** CI opens PRs that update digests in GitOps.
- **Option B (Image Updater):** Argo CD Image Updater with Git write-back.
- **Option C (Kargo):** promotion controller committing refs per stage.

**DoD**
- A new build results in a Git change (digest update) and an Argo-driven rollout without manual intervention.
- Rollout events are auditable (commit shows old/new digest).

### WP-5: Policy enforcement (CI + optional admission control)

**CI enforcement (required)**
- Reject floating tags in GitOps-managed environment directories (except explicit allowlisted local-only transitional cases).
- Require digest pinning for staging/prod (and for cluster-scoped operators).

**Admission control (optional)**
- Gatekeeper/Kyverno policy to require digests in target namespaces.

**DoD**
- CI fails fast when a disallowed tag policy is violated.
- The allowlist is explicit and time-boxed.

### WP-6: Reduce Argo drift/noise related to defaulted image fields

**Target:** Argo stays boring: no spurious OutOfSync from defaulted `imagePullPolicy`.

**DoD**
- Workloads either set `imagePullPolicy` explicitly or `ignoreDifferences` is applied narrowly where defaulting is unavoidable.
- Drift guidance is documented in `backlog/519-gitops-fleet-policy-hardening.md` and referenced from 602/603.

## Verification checklist

- Argo applications converge without manual restarts after a new image build.
- `kubectl get deploy -A -ojsonpath='{..image}{\"\\n\"}'` shows digests (or SHA tags) in GitOps-managed environments.
- Primitive CRs show `spec.imagePullPolicy` only when needed (transitional), not as a permanent crutch.

## Risks / gotchas (track explicitly)

- GHCR mirror tags must remain multi-arch for local `arm64` (see `backlog/456-ghcr-mirror.md`).
- Pull secrets must exist in every namespace that runs private images (see `backlog/450-argocd-service-issues-inventory.md`).
- “Fast-moving” tags (`:dev`) must not bleed into staging/prod.
