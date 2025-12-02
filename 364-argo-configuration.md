# 364 – Argo Configuration Hardening

## Summary

- Argo CD now sources every environment directly from the GitOps tree (`infra/kubernetes/gitops/argocd/parent/apps-root-<env>.yaml`), with GitHub for staging/production and the local workspace (`file:///workspace`) for devcontainers, fulfilling the single-source-of-truth story and eliminating the old Gitea dependency.
- Each Helmfile slice (e.g., `bootstrap-namespace`, `crds-gateway`, `data-plane`) is rendered via the CMP plugin with environment-specific values, and the operational runbook lives in `infra/kubernetes/gitops/argocd/README.md`.
- Remaining work for this backlog centers on automation (CI validation/diffing) and service-team onboarding guidance so changes are enforced before merge and new workloads have a checklist to follow.

## Goals

1. **Single Source of Truth (Complete)** – Application manifests live under `infra/kubernetes/gitops/argocd/{foundation,platform,apps}`, with environment selection handled by the `parent/apps-root-<env>.yaml` and matching ApplicationSets.
2. **Deterministic Repos (Complete)** – All environments authenticate to GitHub; there are no default local-only fallbacks.
3. **Layered Helmfile Support (Complete)** – Every Helmfile stage has a CMP-driven Argo Application with env-specific values and sync waves.
4. **Operational Clarity (Complete)** – Runbooks/CLI helpers in `infra/kubernetes/gitops/argocd/README.md` cover bootstrap, sync, rollback, and troubleshooting.
5. **Automated Validation (In Progress)** – Add CI steps (`argocd app diff --dry-run`, `helmfile template`, etc.) and migration guidance for service teams onboarding via Argo.

## Work Breakdown

| ID | Task | Owners | Notes |
|----|------|--------|-------|
| 364.1 | Clean up the GitOps overlays (`parent/apps-root-<env>.yaml` + `applicationsets/<env>.yaml`) so only the intended repo URLs are referenced; add repo secrets where needed | Platform Infra | Status: Complete – staging/production pin to GitHub, local points at `file:///workspace`. |
| 364.2 | Finalize the per-layer Helmfile Applications (`infra-*`): verify env selection, sync waves, and retry policies | Platform Infra + SRE | Status: Complete – CMP plugin definitions live under `gitops/argocd/foundation`/`platform`/`apps`, sharing waves/retries. |
| 364.3 | Document Argo operational runbook (bootstrap, sync, rollback) inside `infra/kubernetes/gitops/argocd/README.md` | DX Team | Status: Complete – README now covers login, sync, diff, wait, and rollback flows. |
| 364.4 | Add CI/automation step that validates Argo manifests (lint + `argocd app diff --dry-run` or `helmfile template`) | CI Platform | Status: Pending – still need pre-merge validation workflow. |
| 364.5 | Provide migration guidance for service teams: how to onboard new Helm charts via Argo vs Helmfile | Platform PM | Status: Pending – checklist/readme addendum not yet drafted. |

## Risk & Open Questions

- **Repo Secrets:** What’s the canonical method for injecting GitHub PAT / deploy key into Argo? (Options: HashiCorp Vault sync, Kubernetes secret, SOPS.)
- **Environment Promotion:** Do we eventually want stage-specific branches/tags, or will overlays remain sufficient?
- **Testing:** Need lightweight validation (kind/k3d) to ensure each `infra-*` Helmfile Application renders quickly; CMP template could be heavy.

## Definition of Done

- `helmfile -f helmfiles/00-argocd.yaml -e <env> sync` installs Argo + the env-specific root Application without any local-only dependencies.
- `kubectl get applications -n argocd` shows healthy `infra-*` layer apps across all environments.
- Documentation clarifies how to add/modify Argo Applications, secrets, and Helmfile layers.
