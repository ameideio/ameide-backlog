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

Keep linking any follow-up issues or incidents referencing vendored charts to this backlog until all acceptance criteria are met.
