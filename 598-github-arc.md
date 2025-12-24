# 598: GitHub Actions Runner Controller (ARC) — Local Cluster Install (Isolated Namespaces)

**Status:** Ready to implement  
**Created:** 2025-12-24  
**Owner:** Platform SRE / GitOps  
**Scope:** Install GitHub’s Actions Runner Controller (ARC) **runner scale sets** in the **local k3d cluster only**, using **isolated namespaces** and the **official OCI Helm charts** (GHCR), without impacting Azure.

---

## Executive Summary

We add **ARC** to the **local** Kubernetes cluster so we can run **GitHub Actions** workloads on-cluster (ephemeral runner pods, autoscaled), while keeping the install:

- **Local-only** (must not appear in Azure)
- **Namespace-isolated** from Ameide environment namespaces (`ameide-local`)
- **GitOps-native** (Argo CD driven; no drift)
- **Secret-safe** (no tokens committed; Secret comes from Vault via ExternalSecrets)

This installs:

1) ARC CRDs (cluster-owned)  
2) ARC controller (cluster-owned operator, `skipCrds: true`)  
3) One runner scale set (env-scoped app; deploys into `arc-runners`)

Pinned to chart version **`0.13.1`**.

---

## Non-goals

- No ARC deployment to Azure environments (dev/staging/production).
- No production-grade runner hardening beyond local needs.
- No CI migration work; this only installs the runner substrate.

---

## Related / Use With

- `backlog/446-namespace-isolation.md`
- `backlog/465-applicationset-architecture.md`
- `backlog/519-gitops-fleet-policy-hardening.md`

---

## Target Architecture

### Namespaces

- `arc-systems`: controller
- `arc-runners`: runner scale set + ephemeral runner pods

### Workflow routing (`runs-on` contract)

For runner scale sets, the workflow routing key is the **installation name** (`runnerScaleSetName`). Treat it as a contract:

```yaml
jobs:
  build:
    runs-on: arc-local
```

### Auth secret shape

ARC expects a Secret in the runner namespace with:

- PAT: `github_token`
- or GitHub App keys

Local GitOps uses **Vault KV → ExternalSecret → Secret** (`arc-github-auth`) in `arc-runners`.

---

## GitOps Surroundings (How this repo deploys things)

### Operators vs workloads (no exceptions on CRDs)

Repo conventions:

- **CRDs**: cluster ApplicationSet (`argocd/applicationsets/cluster.yaml`) at rollout phase `010`
- **Operators/controllers**: cluster ApplicationSet at rollout phase `020`, with `skipCrds: true`
- **Workloads**: env ApplicationSet (`argocd/applicationsets/ameide.yaml`) at rollout phases `100+`

ARC follows this exactly:

- CRDs are applied as `cluster-crds-github-arc`
- Controller is installed as `cluster-github-arc-controller` with `skipCrds: true`

### Local-only allowlist (keeps Azure clean)

Local env-scoped components are curated under:

- `environments/local/components/**/component.yaml`

Local also needs local-only cluster components, so we extend the local overlay so the **cluster** ApplicationSet additionally discovers:

- `environments/local/components/cluster/**/component.yaml`

Azure overlay remains unchanged (cluster appset only reads `_shared/components/cluster/**`).

---

## Implementation Plan (GitOps)

### WP-1: CRDs (cluster, local-only)

- Component: `environments/local/components/cluster/crds/github-arc/component.yaml`
- Values: `sources/values/_shared/cluster/crds-github-arc.yaml`
- CRD file (vendored): `sources/charts/foundation/common/raw-manifests/files/github-arc-crds-0.13.1.yaml`

### WP-2: Namespaces + defaults (cluster, local-only)

- Component: `environments/local/components/cluster/configs/github-arc-namespaces/component.yaml`
- Values: `sources/values/_shared/cluster/github-arc-namespaces.yaml`

### WP-3: Controller (cluster, local-only)

- Component: `environments/local/components/cluster/operators/github-arc-controller/component.yaml`
- Values: `sources/values/_shared/cluster/github-arc-controller.yaml`
- `serviceAccount.name` is fixed to `arc-gha-rs-controller` and is referenced by the runner scale set.

### WP-4: Auth secret (cluster, local-only)

- Component: `environments/local/components/cluster/secrets/github-arc-auth/component.yaml`
- Values: `sources/values/_shared/cluster/github-arc-auth.yaml`
- Materializes `Secret/arc-github-auth` in `arc-runners` from local Vault key `ghcr-token` (derived from `.env` `GHCR_TOKEN`; bootstrap convenience).
- Local: `.env` secrets are seeded into local Vault by `infra/scripts/seed-local-secrets.sh` (invoked by `infra/scripts/deploy.sh local`); `GHCR_TOKEN` becomes Vault key `ghcr-token`.
- Azure: Terraform can reconcile `.env` → Key Vault secrets when running `infra/scripts/deploy.sh azure` locally; GitHub Actions CI typically reads secrets from Key Vault (no `.env` in CI). ARC auth wiring remains local-only (revisit if promoting).

### WP-5: Runner scale set (env, local-only)

- Component: `environments/local/components/apps/runtime/github-arc-runner-set/component.yaml`
- Values:
  - `sources/values/_shared/apps/github-arc-runner-set.yaml`
  - `sources/values/env/local/apps/github-arc-runner-set.yaml`
- Argo sync posture: `SkipDryRunOnMissingResource=true`
- `controllerServiceAccount` is set explicitly (required for GitOps / cross-namespace installs)
- Local default container mode: `kubernetes-novolume`

---

## Definition of Done (local)

- Argo CD Applications are `Synced` and `Healthy`:
  - `cluster-crds-github-arc`
  - `cluster-github-arc-namespaces`
  - `cluster-github-arc-controller`
  - `local-github-arc-runner-set`
- Namespaces exist: `arc-systems`, `arc-runners`
- A workflow in `ameideio/ameide` or `ameideio/ameide-gitops` runs with `runs-on: arc-local`

---

## Workflow Onboarding (Other Repos)

### 1) Route workflows to ARC

Set the job `runs-on` to the ARC scale set name:

```yaml
runs-on: arc-local
```

Recommended pattern (allows overriding per-repo with a GitHub **Variable**):

```yaml
runs-on: ${{ vars.AMEIDE_RUNS_ON || 'arc-local' }}
```

Then, in each repo:

- Create GitHub variable `AMEIDE_RUNS_ON=arc-local`
- Ensure the repo is in the `ameideio` org (this local ARC install is org-attached via `githubConfigUrl: https://github.com/ameideio`)

### 2) Runner group access (optional but recommended)

If you restrict runner groups, ensure the target repo is allowed to use the configured `runnerGroup` (default is `default`).

### 3) Security guardrail for PRs from forks

Self-hosted runners should not execute untrusted fork PR code. If you keep `pull_request` triggers, add a job-level guard:

```yaml
if: ${{ github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository }}
```

### 4) Smoke test

Add a `workflow_dispatch` smoke workflow and trigger it:

```bash
gh workflow run <workflow-file>.yaml -R ameideio/<repo> --ref main
```

While it runs:

```bash
kubectl -n arc-runners get pods -w
```

---

## Risks / Gotchas

- **CRD upgrades:** some ARC upgrades require remove/replace per release notes; treat upgrades as a change window.
- **Job container requirement:** ARC Kubernetes modes can enforce “job must define `container:`” via `ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER=true`. Local config overrides this to allow standard workflows without a job container.
- **Metrics footgun:** if enabling controller metrics, also configure listener metrics or listeners may fail to start.
- **Egress:** runners require outbound access to GitHub + DNS; if default-deny egress is introduced later, add explicit allowances for `arc-runners`.

---

## Open Questions

- Keep org-level attachment (`githubConfigUrl: https://github.com/ameideio`) or scope to a repo?
- Do we need a non-default `runnerGroup`, or stay in `default`?
