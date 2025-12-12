# 419 – ClickHouse + Argo CD resiliency and rollout contract

> **Cross-References (Deployment Architecture Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | How data-clickhouse app deploys |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Wave 210 (CRDs) → 220 (operator) → 250 (CHI) |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | CRDs at `sources/charts/foundation/common/raw-manifests/files/` |
> | [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md) | ClickHouse tolerations |

**Status:** Documented / stabilized
**Scope:** data-clickhouse CRDs/operator/CHI under Argo CD
**Date:** 2025-11-29

## Configuration snapshot
- **Components & waves (Argo ApplicationSet `ameide-dev`):**
  - Wave 210: `data-crds-clickhouse` (`environments/dev/components/data/crds/clickhouse/component.yaml`)
  - Wave 220: `foundation-clickhouse-operator`
  - Wave 250: `data-clickhouse` (`environments/dev/components/data/core/clickhouse/component.yaml`)
- **CRDs (vendor contract):** Four Altinity CRDs applied via `data-crds-clickhouse` (raw-manifests chart), each as a separate file vendored from the Altinity 0.25.5 release:
  - `clickhouseinstallations.clickhouse.altinity.com`
  - `clickhouseinstallationtemplates.clickhouse.altinity.com`
  - `clickhousekeeperinstallations.clickhouse-keeper.altinity.com`
  - `clickhouseoperatorconfigurations.clickhouse.altinity.com`
  Value file: `sources/values/_shared/data/data-crds-clickhouse.yaml` lists all four `manifestFiles`.
- **Operator:** Deployed by `foundation-clickhouse-operator` (Altinity operator chart). The data ClickHouse chart has `operator.enabled: false` to avoid double-installation.
- **CHI runtime:** `data-clickhouse` uses Altinity `clickhouse` chart with pinned image `clickhouse/clickhouse-server:25.10.2-alpine`, 1 shard/replica, PVC 20Gi, secrets via ExternalSecret `clickhouse-auth`.
- **Health:** Custom Argo Lua health for `ClickHouseInstallation` (reads `spec.stop/suspend`, `status.status`, `status.error/errors`; Healthy when `status.status == "Completed"`). OpenLibs enabled.
- **Orphans:** Project-level ignores for operator-emitted CHI children (Service/StatefulSet/PDB/ConfigMap with `chi-*`) to prevent orphan warnings; warn=false in `project-ameide`.
- **Sync options:** SSA disabled on CRD app and CHI app (client-side apply). `SkipDryRunOnMissingResource=true` remains on CHI app.

## Issues observed
1) **Incomplete CRD bundle**  
   - Only `clickhouseoperatorconfigurations` existed on fresh clusters; CHI CRD was missing. Argo sync failed with “ClickHouseInstallation not found”.
2) **Argo SSA/diff panics on CRDs**  
   - Enabling `ServerSideApply=true` on CHI/CRD apps triggered Argo’s known nil-pointer in server-side diff for CRDs; left `runtime error: invalid memory address` in `operationState.message`.
3) **Orphan warnings**  
   - Operator-owned CHI children flagged as orphans; warnings persisted.
4) **Stale operationState**  
   - After fixes, Argo still displayed old panic text until `operationState` was cleared and app refreshed.

## Remediations applied
- **CRDs:** Vendored all four Altinity CRDs individually (`clickhouse-crd-*.yaml`); `data-crds-clickhouse` now applies all four. Verified via `kubectl get crd | grep clickhouse`.
- **SSA:** Disabled SSA on `data-crds-clickhouse` and `data-clickhouse` to avoid the Argo CRD diff panic path (client-side apply only).
- **Orphans:** Added project-level ignore patterns for `chi-*` Service/StatefulSet/PDB/ConfigMap, `warn: false`, so operator-managed children no longer raise warnings.
- **Cleanup:** Cleared stale `operationState` once after fixes; subsequent syncs are clean.
- **Files touched (authoritative):**
  - CRDs: `sources/charts/foundation/common/raw-manifests/files/clickhouse-crd-{chi,chit,chk,chopcfg}.yaml`
  - CRD values: `sources/values/_shared/data/data-crds-clickhouse.yaml`
  - CHI component: `environments/dev/components/data/core/clickhouse/component.yaml`
  - CHI values: `sources/values/_shared/data/data-clickhouse.yaml` (+ `sources/values/env/dev/data/data-clickhouse.yaml`)
  - Health script & OpenLibs: `sources/values/common/argocd.yaml`, `environments/dev/argocd/ops/argocd-cm.yaml`
  - Orphan ignores: `environments/dev/argocd/project-ameide.yaml`

## Reproducible rollout (new cluster)
1. Argo syncs `data-crds-clickhouse` → all four CRDs present.
2. Argo syncs `foundation-clickhouse-operator` → operator ready.
3. Argo syncs `data-clickhouse` → CHI created; health goes Healthy when `status=Completed`.
4. Downstream apps (e.g., Langfuse) proceed; DNS `clickhouse-data-clickhouse.ameide.svc.cluster.local` resolves.

## Current state
- `data-crds-clickhouse`: Synced/Healthy.  
- `data-clickhouse`: Synced/Healthy; CHI `STATUS: Completed`.  
- No orphan warnings; SSA off for CRD/CHI apps; health script active.

## Follow-ups (if desired)
- When Argo upgrades to a version that fixes SSA/CRD diff nil-pointer, re-test SSA on CHI/CRD apps.
- Keep CRD files in sync with Altinity releases when upgrading operator.

## Implementation details (deep)
- **CRD app (`data-crds-clickhouse`):**
  - Chart: `sources/charts/foundation/common/raw-manifests`
  - Values: `sources/values/_shared/data/data-crds-clickhouse.yaml` with four `manifestFiles` pointing to the vendored CRDs above.
  - SSA: disabled (client-side apply). SyncOptions: `CreateNamespace`, `RespectIgnoreDifferences`.
- **CHI app (`data-clickhouse`):**
  - Chart: `sources/charts/third_party/altinity/clickhouse/0.3.6`
  - Values: `_shared` defines storage/resources/users/image; `dev` overlay tweaks storage/resources.
  - SyncOptions: `CreateNamespace`, `RespectIgnoreDifferences`, `SkipDryRunOnMissingResource` (no SSA).
  - Operator subchart: disabled (`operator.enabled: false`).
  - ExternalSecret: `clickhouse-auth` (from `data-clickhouse-secrets` / Vault).
- **Health script (ClickHouseInstallation):**
  - Location: `sources/values/common/argocd.yaml` (mirrored to `environments/dev/argocd/ops/argocd-cm.yaml`).
  - Logic: Suspended on `spec.stop/suspend`; Degraded on `status.error/errors`; Healthy on `status.status == "Completed"`; Progressing otherwise. OpenLibs enabled to use Lua stdlib safely.
- **Orphan policy:**
  - Project `ameide`: `warn: false`, ignore chi-* Service/StatefulSet/PDB/ConfigMap to acknowledge operator ownership.
- **Wave ordering:**
  - CRDs (wave ~210) → operator (wave ~220) → CHI (wave ~250) → dependents.

## Troubleshooting notes
- **CRD presence check:** `kubectl get crd | grep clickhouse` must show all four; absence of `clickhouseinstallations` causes CHI sync failures.
- **Argo panic “invalid memory address”:** the Argo CD application controller occasionally panics while diffing the `ClickHouseInstallation` resource, leaving `status.operationState.phase=Error` and `message="runtime error: invalid memory address or nil pointer dereference"` even though the CHI is `Synced/Healthy`. This is a bug in Kubernetes apimachinery’s `strategicpatch` code path, not in ClickHouse or our manifests.
  - **Observed stack (v3.2.0, k8s 1.30):** controller logs show `Recovered from panic: runtime error: invalid memory address or nil pointer dereference` in `k8s.io/apimachinery/pkg/util/strategicpatch.handleMapDiff` → `CreateThreeWayMergePatch`, called from `github.com/argoproj/argo-cd/v3/controller.getMergePatch/normalizeTargetResources`. The panic is in the diff/patch layer, not during actual CHI reconciliation.
  - **Argo / apimachinery versions:** both Argo CD `v3.2.0+66b2f30` and `v3.2.1` pin `k8s.io/apimachinery v0.34.0` in `go.mod`, and the 3.2.1 changelog does not mention a fix for this panic. As of 2025‑12‑01, upgrading from 3.2.0 to 3.2.1 does *not* eliminate the issue.
  - **Current mitigation:** scope `ServerSideApply=true` and `SkipDryRunOnMissingResource=true` to the ClickHouse CRD/CHI Applications so normal syncs succeed and the CHI reaches `status.status=Completed`. When the only symptom is `operationState.phase=Error` with the panic message—but `status.sync.status=Synced` and `status.health.status=Healthy`—treat it as cosmetic noise and clear `status.operationState` once via `kubectl patch` if the UI needs to be clean.
  - **Tracking:** when Argo CD upgrades to a release that bumps `k8s.io/apimachinery` beyond `v0.34.0` and explicitly calls out a fix for the strategicpatch/CRD panic, re-test ClickHouse syncs and remove this workaround if the panic no longer reproduces.
- **Health stuck Unknown/Progressing:** ensure CM with Lua health is applied (argocd-cm) and OpenLibs enabled for `clickhouse.altinity.com_ClickHouseInstallation`.
