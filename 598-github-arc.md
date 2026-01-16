# 598: GitHub Actions Runner Controller (ARC) — Cluster Runner Scale Sets (local + AKS)

**Status:** Implemented (local + AKS)  
**Created:** 2025-12-24  
**Owner:** Platform SRE / GitOps  
**Scope:** Install GitHub’s Actions Runner Controller (ARC) **runner scale sets** in both the **local k3d cluster** and **AKS**, using **isolated namespaces** and the **official OCI Helm charts** (GHCR).

---

## Historical context

- **2025-12**: implemented **local-only** (`k3d`) ARC to unblock GitOps-first inner-loop work without depending on GitHub-hosted runners.
- **2026-01**: extended the same model to **AKS** to make CI-owned cluster rollouts and verification deterministic across environments.

## Executive Summary

We add **ARC** to the **local** Kubernetes cluster and **AKS** so we can run **GitHub Actions** workloads on-cluster (ephemeral runner pods, autoscaled), while keeping the install:

- **Namespace-isolated** from Ameide environment namespaces (`ameide-local`, `ameide-dev`, `ameide-staging`, `ameide-prod`)
- **GitOps-native** (Argo CD driven; no drift)
- **Secret-safe** (no tokens committed; Secret comes from Vault via ExternalSecrets)

This installs:

1) ARC CRDs (cluster-owned)  
2) ARC controller (cluster-owned operator, `skipCrds: true`)  
3) One runner scale set per cluster (cluster-scoped app; deploys into `arc-runners`)

Pinned to chart version **`0.13.1`**.

## Current state (2026-01)

### Runner sets (one per cluster)

- Local runner scale set name: `arc-local`
- AKS runner scale set name: `arc-aks`

### GitHub routing contract (no workflow defaults)

This repo’s workflows route runners via a GitHub variable only:

```yaml
jobs:
  build:
    runs-on: ${{ vars.AMEIDE_RUNS_ON }}
```

Set `AMEIDE_RUNS_ON` to:
- `arc-local` to run on local k3d ARC, or
- `arc-aks` to run on AKS ARC.

### Org-scoped runner registration (shared across repos)

The runner sets are registered to the org via:
- `githubConfigUrl: https://github.com/ameideio`

This enables a shared runner substrate for multiple repos, but it makes **runner group governance** matter (which repos are allowed to use the group).

### Failure mode: “jobs never start” when runner groups restrict the repo

If the org runner group used by the scale set does not allow a repo, jobs for that repo can sit in `queued` with no runner assigned (even if `runs-on: arc-aks` matches).

- Symptom: jobs stay `queued` and ARC never scales up for them.
- Resolution: allow the repo in the runner group, or deploy a dedicated runner set for that repo.
- Note: GitHub does not queue literally forever; jobs will fail after GitHub’s max queue time (currently 24h).

### Runner image is multi-arch and digest pinned

- Runner image: `ghcr.io/ameideio/arc-local-runner`
- Must be a **multi-arch manifest list** (at least `linux/amd64` + `linux/arm64`).
- GitOps pins the manifest digest (see `sources/values/_shared/cluster/github-arc-runner-set.yaml`).

---

## Non-goals

- No org-wide runner pool / runner group policy design (repo-scoped by default).
- No “trusted vs untrusted” runner isolation model beyond the current cluster boundaries.
- No CI migration work; this backlog only installs the runner substrate + routing contract.

### Follow-on (separate backlog)

ARC alone does not guarantee Docker is available on runner pods. Any workflow that builds container images should adopt a **k8s-native** build path (BuildKit-in-cluster, Buildx Kubernetes driver, etc.) rather than assuming a local Docker daemon.

**Target model (explicit):** local ARC runners (`arc-local`) must not rely on Docker-in-Docker; image builds use in-cluster BuildKit (`buildctl` → `buildkitd`).

See `backlog/599-k8s-native-buildkit-builds-on-arc.md`.

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

### Scheduling (AKS workload-class pools)

- ARC controller stays on the `system` pool (cluster operator).
- Runner pods schedule onto the dedicated `arc-runners` pool (autoscale 0→N, taint-protected).

Cross-reference: `backlog/684-aks-node-pool-strategy-option-b.md`.

### Workflow routing (`runs-on` contract)

For runner scale sets, the workflow routing key is the **scale set name** (`runnerScaleSetName`). Treat it as a contract:

```yaml
jobs:
  build:
    runs-on: ${{ vars.AMEIDE_RUNS_ON }}
```

### Standard runner image (local)

`arc-local` uses an org-owned, pinned runner image to remove per-workflow “installer glue”:

- Image: `ghcr.io/ameideio/arc-local-runner`
- Baseline tools include `git/curl/tar/jq/yq/rg/skopeo` and `buildctl` (for BuildKit builds)
- GitOps pins the runner image digest in `sources/values/_shared/cluster/github-arc-runner-set.yaml`
  - Tooling includes `rsync` + `cosign` so workflows don’t carry installer glue.

Image policy note:

- Any GitOps-managed runner image reference MUST be digest-pinned per `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
- Rollouts should happen because Git changed (PR write-back), not because of `imagePullPolicy`.

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

### Overlay component discovery (local + Azure)

Cluster-scoped components are deployed by the cluster ApplicationSet (`argocd/applicationsets/cluster.yaml`).
Overlays extend cluster component discovery:

- Local overlay adds: `environments/local/components/cluster/**/component.yaml`
- Azure overlay adds: `environments/azure/components/cluster/**/component.yaml`

---

## Implementation (GitOps)

### WP-1: CRDs (cluster)

- Components:
  - Local: `environments/local/components/cluster/crds/github-arc/component.yaml`
  - Azure: `environments/azure/components/cluster/crds/github-arc/component.yaml`
- Values: `sources/values/_shared/cluster/crds-github-arc.yaml`
- CRD file (vendored): `sources/charts/foundation/common/raw-manifests/files/github-arc-crds-0.13.1.yaml`

### WP-2: Namespaces + defaults (cluster)

- Components:
  - Local: `environments/local/components/cluster/configs/github-arc-namespaces/component.yaml`
  - Azure: `environments/azure/components/cluster/configs/github-arc-namespaces/component.yaml`
- Values: `sources/values/_shared/cluster/github-arc-namespaces.yaml`

### WP-3: Controller (cluster)

- Components:
  - Local: `environments/local/components/cluster/operators/github-arc-controller/component.yaml`
  - Azure: `environments/azure/components/cluster/operators/github-arc-controller/component.yaml`
- Values: `sources/values/_shared/cluster/github-arc-controller.yaml`
- `serviceAccount.name` is fixed to `arc-gha-rs-controller` and is referenced by the runner scale set.

### WP-4: Auth secret (cluster)

- Components:
  - Local: `environments/local/components/cluster/secrets/github-arc-auth/component.yaml`
  - Azure: `environments/azure/components/cluster/secrets/github-arc-auth/component.yaml`
- Values: `sources/values/_shared/cluster/github-arc-auth.yaml`
- Materializes `Secret/arc-github-auth` in `arc-runners` via ExternalSecrets using the existing `ghcr-token` secret value (kept as-is for now).

### WP-5: Runner scale set (cluster)

Runner scale set is cluster-scoped (deployed once per cluster):

- Components:
  - Local: `environments/local/components/cluster/apps/github-arc-runner-set/component.yaml`
  - Azure: `environments/azure/components/cluster/apps/github-arc-runner-set/component.yaml`
- Values:
  - Shared: `sources/values/_shared/cluster/github-arc-runner-set.yaml`
  - Local override: `sources/values/cluster/local/github-arc-runner-set.yaml`
  - Azure override: `sources/values/cluster/azure/github-arc-runner-set.yaml`
- Argo sync posture: `SkipDryRunOnMissingResource=true`
- `controllerServiceAccount` is set explicitly (required for GitOps / cross-namespace installs)
- Local default container mode: `kubernetes-novolume`

### WP-6: No DinD runner sets

ARC runners should not require Docker-in-Docker. Use in-cluster BuildKit for image builds (`backlog/599-k8s-native-buildkit-builds-on-arc.md`), and keep workflows free of Docker daemon assumptions.

If a workflow truly requires Docker daemon semantics, run it on GitHub-hosted runners or introduce a separate, explicitly trusted runner set (out of scope for local ARC).

---

## Definition of Done (local + AKS)

- Argo CD Applications are `Synced` and `Healthy`:
  - `cluster-crds-github-arc`
  - `cluster-github-arc-namespaces`
  - `cluster-github-arc-controller`
  - `cluster-github-arc-runner-set`
- Namespaces exist: `arc-systems`, `arc-runners`
- A workflow in `ameideio/ameide-gitops` runs with `runs-on: ${{ vars.AMEIDE_RUNS_ON }}` and succeeds on both `arc-local` and `arc-aks` when the variable is flipped.

---

## Workflow Onboarding (Other Repos)

### Scope note: these runners are repo-scoped

Because the runner sets are registered to `https://github.com/ameideio/ameide-gitops`, they are not a shared org runner pool.

If another repo needs ARC runners, choose one:
- Deploy a dedicated runner set for that repo (recommended).
- Switch ARC to org-scoped registration (`githubConfigUrl: https://github.com/ameideio`) and manage runner group allowlists (requires org admin governance).

### 2) Runner group access (org-scoped mode only)

If/when ARC is switched back to org-scoped registration, ensure the target repo is allowed to use the configured `runnerGroup`.

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

## Recommended Next Step: k8s-native image builds

Once ARC is installed, the main operational pain point is container builds: self-hosted runners may not provide Docker, and k3d-only hostnames/service endpoints are often only reachable from inside the cluster.

Adopt a k8s-native build strategy:

- Run a cluster BuildKit daemon (cluster-scoped) and have workflows use `buildctl` to build/export (or push) images.
- Keep publish workflows **dry-run by default** on `workflow_dispatch` so ARC verification remains safe.
- Standardize on `AMEIDE_BUILDKIT_ADDR=tcp://buildkitd.buildkit.svc.cluster.local:1234` (org-level preferred; per-repo fallback is acceptable).
- Prefer a repo/org variable for the BuildKit endpoint (example): `AMEIDE_BUILDKIT_ADDR=tcp://buildkitd.buildkit.svc.cluster.local:1234`.

---

## Risks / Gotchas

- **CRD upgrades:** some ARC upgrades require remove/replace per release notes; treat upgrades as a change window.
- **Job container requirement:** ARC Kubernetes modes can enforce “job must define `container:`” via `ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER=true`. Local config overrides this to allow standard workflows without a job container.
- **Metrics footgun:** if enabling controller metrics, also configure listener metrics or listeners may fail to start.
- **Egress:** runners require outbound access to GitHub + DNS; if default-deny egress is introduced later, add explicit allowances for `arc-runners`.

---

## Open Questions

- Do we keep repo-scoped registration for deterministic access, or switch back to org-scoped (`githubConfigUrl: https://github.com/ameideio`) to serve multiple repos (requires runner group governance)?
- If org-scoped: which `runnerGroup`/allowlist model do we adopt for isolation (and how do we prevent untrusted fork PR code from reaching self-hosted runners)?
