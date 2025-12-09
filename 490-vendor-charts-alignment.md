# 490 – Third-Party Chart Vendoring Alignment

**Status**: Draft  
**Created**: 2025-12-09  
**Owner**: Platform / GitOps  
**Related**: [456-ghcr-mirror.md](456-ghcr-mirror.md) · [464-chart-folder-alignment.md](464-chart-folder-alignment.md) · [458-helm-chart-ci-testing.md](458-helm-chart-ci-testing.md) · [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md)

> **Repository scope**: Application source lives in `ameide-core`. This backlog applies to the `ameide-gitops` repo, which owns vendored Helm charts, Argo CD configs, and infra automation.

---

## 1. Background

This `ameide-gitops` repository is the platform’s infrastructure source of truth: it keeps all Helm charts under `sources/charts/`, values overlays under `sources/values/`, and the automation scripts (`scripts/vendor-charts.sh`, `scripts/validate-hardened-charts.sh`, etc.) that Argo CD relies on. Application services live in `ameide-core`; they only consume the outputs published here (cluster state, vendored CRDs, mirrored container images). Any “vendoring” work (pinning upstream charts, mirroring container images, recording supply-chain metadata) happens here; `ameide-core` only vendors its own application dependencies (e.g., SDKs) and has no third-party Helm charts.

We finished consolidating all Helm charts under `sources/charts/` (per backlog/464), but the operational tooling that refreshes vendored dependencies still points at the pre-refactor layout (`infra/kubernetes/charts/third_party`). As a result:

- `scripts/vendor-charts.sh` aborts with “Missing //infra/kubernetes/charts/third_party/charts.lock.yaml” even though the authoritative lock file now lives at `sources/charts/third_party/charts.lock.yaml`.
- Contributors have no single checklist that explains how to add/refresh a third-party chart, mirror its container images (backlog/456), and prove that Argo CD sees only vendored sources (backlog/458 test expectations).
- CI can drift silently: nothing enforces that the lock file matches the vendored directories, and ApplicationSets still rely on people remembering to run the script whenever the lock changes.

This backlog defines the desired end-state for vendoring: a single lock file, deterministic script, CI guardrails, and documentation that lives alongside the charts.

---

## 2. Objectives

1. **Update tooling** – Point `scripts/vendor-charts.sh` (and any callers) at `sources/charts/third_party/...`, handle OCI artifacts + raw manifests, and fail fast when `helm pull` exits non-zero.
2. **Add verification** – Extend `scripts/validate-hardened-charts.sh` or a new lint step to ensure `charts.lock.yaml` matches the vendored tree (no missing versions, no stale directories).
3. **Document workflow** – Update `sources/charts/README.md` (and link from the repo root README) with a “Vendoring process” section: edit lock → run script → run validator → commit artifacts.
4. **Wire CI** – Add a GitHub Action job (or extend an existing workflow) that runs the validator and fails if vendored state diverges from the lock, similar to how buf/SDK lockfiles are enforced.
5. **Ownership clarity** – Route all future chart bumps through PR templates: describe which alias changed, which upstream release notes were reviewed, and which mirrored images (backlog/456) need updates.

---

## 3. Current state (Dec 2025)

| Item | Current behavior | Gap |
|------|------------------|-----|
| `scripts/vendor-charts.sh` | ✅ (2025-12-09) now targets `sources/charts/third_party/...`, offers a `--lock` override flag, and surfaces `helm pull`/`curl` failures. | Validator + CI guardrails still pending to ensure the layout stays in sync with the lock file. |
| `charts.lock.yaml` | Lives at `sources/charts/third_party/charts.lock.yaml` and already drives `helm pull` manually. | No automation to keep vendored artifacts synchronized. |
| CI | `validate-hardened-charts.sh` only checks security knobs in Ameide-authored charts. | No coverage for third-party dependencies. |
| Documentation | `sources/charts/README.md` still shows `platform-layers/` and makes no mention of the vending script or lock process. | Contributors lack guidance and reach for upstream repos every time. |

### Deviations observed while wiring tooling (Dec 8)

- `scripts/check-vendored-charts.sh` immediately reports a **missing OCI directory** for `ghcr.io/argoproj/argo-helm/argo-cd/9.1.3`; the lock references this chart, but no unpacked tree exists under `sources/charts/third_party/oci/...`, so the validator cannot pass.
- The validator also surfaces **legacy versions that remain in Git** (`cnpg/cloudnative-pg/0.25.0`, `external-secrets/0.20.3`, `grafana/loki/6.45.2`, `grafana/loki/6.46.0`, `grafana/tempo/1.24.0`, `prometheus-community/prometheus-operator-crds/17.0.0`, `prometheus-community/kube-prometheus-stack/79.1.1`, `spotahome/redis-operator/3.3.0`, `strimzi/0.47.0`, etc.). These directories are intentional rollback anchors today, but the lock has no concept of “allowed historic versions”, so every CI run would fail until we either prune or model them.
- Some vendored charts do **not follow the alias/name/version layout** (e.g., `altinity/clickhouse/clickhouse` nests an additional folder, `hashicorp/vault` and `telepresence` keep legacy structures). The lock cannot describe these shapes today, so the validator flags them as unexpected directories.
- Extracted OCI charts include sub-chart directories containing `Chart.yaml`. The current drift check walks every `Chart.yaml` under `sources/charts/third_party/oci/**`, so it mislabels those nested charts (e.g., `envoyproxy/gateway-helm/charts/*`) as unexpected locations. We must restrict the scan to the top-level chart path or encode expectations for subcharts in the lock.

These deviations need to be resolved before the “verification” and “CI enforcement” objectives in section 4 can be marked complete.

---

## 4. Plan of record

### Phase A – Tooling parity

1. ✅ Update `scripts/vendor-charts.sh`
   - Added default `LOCK_FILE="${REPO_ROOT}/sources/charts/third_party/charts.lock.yaml"` with a `--lock` override.
   - Writes all repo/OCI/manifest artifacts below `sources/charts/third_party/...` and no longer touches `infra/kubernetes/...`.
   - `helm pull`/`curl` failures bubble up; Helm repo cache refresh uses `--force-update`.
2. ✅ Create `scripts/check-vendored-charts.sh`
   - Compares the lock to `sources/charts/third_party/**` (including OCI paths) and fails on missing directories/files or stray versions.
   - Current run highlights legacy artifacts (`hashicorp`, `telepresence`, older chart versions, unused OCI paths) that must be removed or re-locked before CI can enforce it.
3. ✅ Lock coverage for bespoke charts
   - Added lock entries for the Altinity ClickHouse runtime, HashiCorp Vault, Envoy Gateway CRDs, and Telepresence (marked as `source: local` because the vendor ships the chart inside the CLI release).
   - Rehydrated the tree so every entry follows the `<alias>/<name>/<version>` shape (`telepresence/telepresence/2.25.1`, `altinity/clickhouse/0.3.6`, `oci/.../gateway-crds-helm/v1.6.0`), letting the vendor script manage them going forward.

### Phase B – Documentation + CI

3. Update `sources/charts/README.md`
   - Add “Vendoring workflow” steps.
   - Cross-link backlog/456 for mirrored container images.
   - Mention the new validator script + how to run it locally.
4. Extend `.github/workflows/validate-charts.yaml` (or add a new workflow)
   - Run `scripts/vendor-charts.sh --dry-run` (no writes) to ensure lock is parseable.
   - Run the validator script and fail the build on drift.
   - Optional: install `helm`/`yq` caching for faster runs.

### Phase C – Guardrails

5. Update PR template / docs checklist
   - Add an item: “If `sources/charts/third_party/charts.lock.yaml` changed, run `scripts/vendor-charts.sh` and commit vendored outputs.”
6. Add a `make vendor-charts` target that wraps the script, so developer docs can reference `make vendor-charts`.
7. Investigate adding a Renovate (or Dependabot) datasource to propose chart bumps automatically, tagging the new backlog as the process owner.

---

## 5. Acceptance criteria

- Running `./scripts/vendor-charts.sh` in a clean workspace repopulates `sources/charts/third_party` identically to the committed state (no git diff).
- CI fails when the lock and vendored directories diverge.
- Documentation explains the vendoring workflow, and the repo root README links to it.
- PR template and/or CONTRIBUTING checklist remind authors to refresh vendored artifacts whenever they touch the lock.
- Future chart bumps (e.g., `helm pull` for GitLab, Temporal, CNPG) are self-service: the backlog can be closed once the first end-to-end chart update runs through the new flow.

---

## 6. File map

| File | Purpose |
|------|---------|
| `scripts/vendor-charts.sh` | Download and cache repo/OCI/manifest charts defined in the lock file. |
| `sources/charts/third_party/charts.lock.yaml` | Single source of truth for third-party chart versions. |
| `sources/charts/README.md` | Chart layout + vendoring instructions (to be expanded). |
| `.github/workflows/validate-charts.yaml` *(new or updated)* | CI enforcement for vendored charts. |
| `scripts/check-vendored-charts.sh` *(new)* | Local + CI validator for lock vs. filesystem state. |

---

## 7. Open questions

1. Should we unpack chart archives (`helm pull --untar`) or keep `.tgz` blobs to reduce git noise? (Current state stores the charts untarred; validator must respect whichever format we standardize on.)
2. Do we also vendor CRDs referenced outside Helm (e.g., Gateway API raw manifests) via the same script, or keep a separate raw manifest process?
3. How do we surface upstream release notes in PR descriptions? Possible approach: embed release URLs beside each lock entry and surface them during diffs.

Answering these will finalize the implementation scope for phase C.

---

## 8. Timeline (target)

| Date | Milestone |
|------|-----------|
| 2025-12-10 | Tooling parity PR merged (`vendor-charts.sh` + validator script). |
| 2025-12-12 | Documentation + CI updates live; backlog moves to “In Review.” |
| 2025-12-16 | First chart upgrade (e.g., `cert-manager v1.19.2`) exercised through the new path; backlog closed. |

---

## 9. Drift inventory (Dec 2025)

Output from `./scripts/check-vendored-charts.sh` on 2025-12-09. Each line captures where the chart is referenced today, what the script found in `sources/charts/third_party`, and what “latest stable” means so we can scope the follow-up stories. Obvious duplicates (old versions left behind) need to be deleted after the lock is re-rendered; divergences must be reconciled by bumping the lock + vendored contents.

### Observability stack

- **Grafana** – `environments/_shared/components/observability/grafana/component.yaml` still references `sources/charts/third_party/grafana/grafana/9.3.1`, while the tree also contains 10.1.4 bits. Latest upstream chart is **10.3.0** (`helm search repo grafana/grafana`). Action: plan the Grafana 10.x upgrade (dashboards, image renderer, oauth2 proxy values) and drop 9.x entirely once the Application rolls forward; pending work ties into backlog/436 (observability hardening).
- **Loki** – the component at `…/observability/loki/component.yaml` already points at `grafana/loki/6.46.0`, but the lock still says 6.35.0 and there is a stray 6.45.2 directory. Latest stable is **6.46.0**; update `charts.lock.yaml` + shared values to 6.46.0 and prune 6.35.0/6.45.2 once the validator passes (`platform-loki` depends on the newer chart because of WAL fixes called out in backlog/452).
- **Tempo** – component `…/observability/tempo/component.yaml` uses `grafana/tempo/1.24.0`, but the lock is pinned to 1.23.2. Latest stable is **1.24.1**. Action: bump lock + vendored bits to 1.24.1, re-run integration smoke in backlog/436, then drop 1.23.x.
- **Prometheus stack** – `platform-prometheus` continues to run `kube-prometheus-stack/75.17.0` (`…/observability/prometheus/component.yaml`), yet the tree also contains 79.1.1; upstream is already **80.0.0** (`v0.87.0`). We need a dedicated upgrade PR that bumps the lock, re-renders the CRDs, updates SSA ignores, and retires the 75.x + 79.x directories.
- **Prometheus operator CRDs** – `foundation-crds-prometheus` embeds YAML rendered from an ancient `prometheus-operator-crds/17.0.0` release, but we vendored 24.0.1 and upstream now ships **25.0.0** (`prometheus-operator v0.87.0`). Action: re-render the CRDs from 25.0.0, update `sources/values/_shared/foundation/foundation-crds-prometheus.yaml`, drop the 17.0.0 directory, and teach the lock about CRDs being supplied via raw manifest (ties back to backlog/364 for CRD ownership).

### Data-plane operators & runtimes

- **CloudNativePG** – `…/cluster/operators/cloudnative-pg/component.yaml` uses `cnpg/cloudnative-pg/0.26.1`, but `sources/charts/third_party/cnpg/cloudnative-pg/0.25.0` is still around. Latest stable is **0.26.1**; remove 0.25.0 once the lock diff is clean so that the validator stops flagging it (backlog/447).
- **External Secrets** – operator + CRD components (`…/cluster/operators/external-secrets` and `…/cluster/crds/external-secrets`) point to `external-secrets/0.20.4`, while 0.20.3 remains in the tree. Upstream has GA’d **1.1.1**. We need an upgrade story that covers CRD migration (new `SecretStore` fields), controller values, and Helm test jobs before we can cut 0.20.x entirely.
- **Spotahome Redis operator** – `…/cluster/operators/redis/component.yaml` references `spotahome/redis-operator/3.3.0` but the lock is stuck at 3.2.1. Latest stable is **3.3.0**; bump the lock and remove the 3.2.1 directory so the validator stops failing.
- **Strimzi** – components use `strimzi-kafka-operator/0.48.0`, yet 0.47.0 is still vendored. Latest release is **0.49.1**. Short-term fix: delete the 0.47.0 directory after the lock sync; medium-term plan is to schedule the 0.49.x upgrade per backlog/421.
- **Altinity ClickHouse runtime** – `data-clickhouse` (backlog/381) imports `sources/charts/third_party/altinity/clickhouse/clickhouse`, but the lock has no record of it. The Helm repo exposes **0.3.6**, which is what we run. Action: add `clickhouse` to `charts.lock.yaml` (source `altinity/clickhouse` @0.3.6) so `vendor-charts.sh` can manage it and so CI catches drift.

### Platform & control-plane

- **Argo CD chart** – bootstrap relies on `argo-cd@9.1.3` (per backlog/379), yet the validator fails because `sources/charts/third_party/oci/ghcr.io/argoproj/argo-helm/argo-cd/9.1.3` was never pulled into the new `sources/` tree. Upstream latest is **9.1.6** (AppVersion v3.2.1). We need to re-run `vendor-charts.sh`, commit the OCI tarball, and plan the 9.1.6 bump along with the bootstrap scripts.
- **Legacy `argocd-apps` OCI artifacts** – directories `oci/ghcr.io/argoproj/argo-helm/argocd-apps/{current,2.0.2}` are leftovers from the v2 GitOps layout (see backlog/364) and are unused by any component. Action: delete both directories once the lock no longer references them.
- **Telepresence traffic-manager** – ✅ `environments/_shared/components/platform/control-plane/telepresence/component.yaml` now targets `sources/charts/third_party/telepresence/telepresence/2.25.1`, the chart directory has been re-vendored under the `<alias>/<name>/<version>` layout, and `charts.lock.yaml` tracks it as a repo source. The traffic-manager RBAC fixes (grant `pods/eviction.create` and `networking.k8s.io/servicecidrs.{get,list,watch}` for Kubernetes v1.33+) are baked into that tree, so `vendor-charts.sh` + ArgoCD stay in sync (ties to backlogs 432/434/435).
- **HashiCorp Vault** – `…/foundation/operators/vault/component.yaml` pulls from `sources/charts/third_party/hashicorp/vault/0.28.0`, yet the lock is missing an entry and upstream is at **0.31.0**. Action: add Vault to the lock, schedule an upgrade (newer webhooks + CSI bits), and drop 0.28.x artifacts after validation (ties to backlog/452).
- **GitLab** – no direct drift was flagged, but note that `gitlab-operator/gitlab` both depend on the vendored tree. Ensure any chart bumps go through this backlog once GitLab toleration fixes (backlog/488) land.

### Envoy Gateway + Gateway API

- **Gateway CRDs chart** – `environments/_shared/components/platform/control-plane/envoy-crds/component.yaml` consumes `sources/charts/third_party/oci/docker.io/envoyproxy/gateway-crds-helm`, but the lock has no entry for it. Upstream release **v1.6.1** bundles both Gateway API + Envoy Gateway CRDs; add it to the lock (source: OCI) and pin the component to that version.
- **Gateway operator chart** – we currently keep three copies of Envoy Gateway: the vanilla `1.6.0` pull, a patched `1.6.0-nocreds` directory that strips the bundled certgen hooks per backlog/386, and a “current” symlink. `platform-envoy-gateway` only uses the patched tree. Action items: document the `-nocreds` fork, add a lock entry that records the upstream digest + our patch commit, drop the unused `current/` folder, and plan the upstream upgrade to **v1.6.1** once the certgen patch is rebased.

### Miscellaneous clean-up

- **Gateway API manifests** – `gateway-api/standard-install.yaml` entries already live in the lock; no drift reported.
- **GitOps CRD bundles outside the lock** – validate that `sources/charts/foundation/common/raw-manifests` references only vendored files so the validator doesn’t have to special-case them.

Once the items above are captured in follow-up stories (chart bumps or deletions), rerun `scripts/vendor-charts.sh && scripts/check-vendored-charts.sh` to verify the tree matches `charts.lock.yaml` and wire the validator into CI per Phase B.
Keep linking any follow-up issues or incidents referencing vendored charts to this backlog until all acceptance criteria are met.
