# 421 – Argo CD + Strimzi KafkaNodePool health alignment

**Status:** Documented / fixed / refined  
**Scope:** `data-kafka-cluster` Application (Strimzi Kafka) under Argo CD RollingSync  
**Date:** 2025-12-01

## Configuration snapshot
- **Argo version:** v3.2.0 (OpenLibs enabled; Lua health scripts in `sources/values/common/argocd.yaml` → rendered into `argocd-cm`).  
- **RollingSync steps (ApplicationSet `ameide-dev`):**
  - Wave 210: `data-crds-strimzi` (`environments/dev/components/data/crds/strimzi/component.yaml`, raw-manifests chart with vendored Strimzi 0.48.0 CRDs `files/strimzi/040-049`, plus `ignoreDifferences` on `.spec.versions` for those 10 CRDs to mute API-server schema/defaulting noise).
  - Wave 220: `foundation-strimzi-operator` (`environments/dev/components/foundation/operators/strimzi/component.yaml`, Strimzi operator 0.48.0, CRDs skipped).
  - Wave 250: `data-kafka-cluster` (`environments/dev/components/data/core/kafka-cluster/component.yaml`, operators-config/kafka_cluster chart creating `Kafka` + `KafkaNodePool` in ns `ameide`).
- **Chart/values:** `sources/charts/foundation/operators-config/kafka_cluster` with `_shared` values (`namespace: ameide`, single nodepool replica, Kafka 4.0.0, KRaft).
- **Health customizations (before fix):**
  - `kafka.strimzi.io/Kafka`: Ready-based Lua (kept).
  - `kafka.strimzi.io/KafkaNodePool`: custom Lua that required `status.conditions[type=Ready]` (now removed).
  - `kafka.strimzi.io/KafkaTopic`: Ready-based Lua (kept).
- **Health customizations (after refinements):**
  - `kafka.strimzi.io/Kafka`: still gates **only** on `status.conditions[type=Ready]` with `observedGeneration >= metadata.generation`, but now also surfaces any `Warning` conditions in the message (without changing status) and avoids `string.*` / `table.concat` by using a small local `join()` helper.
  - `kafka.strimzi.io/KafkaTopic`: same pattern as Kafka (Ready-based gating, `observedGeneration` check, warnings included in the message only).
  - `kafka.strimzi.io/KafkaNodePool`: **no** health override – node pools no longer gate Argo health; Kafka remains the authoritative readiness signal.
- **Sync options:** `CreateNamespace`, `RespectIgnoreDifferences`, `SkipDryRunOnMissingResource` on data-kafka-cluster.

## Issues observed
1) **App stuck Progressing:** `data-kafka-cluster` never turned Healthy because the custom KafkaNodePool health script waited for `conditions[type=Ready]`. Strimzi 0.48.0 does not emit a Ready condition for KafkaNodePool (by design; readiness is reported on the Kafka CR). NodePool `status.conditions` was empty, so health stayed Progressing.
2) **Orphan warnings:** Application reported 11 orphaned Secrets (envoy, ghcr pull, tls). This is existing noise unrelated to Kafka health; not addressed in this fix.
3) **Controller reload needed:** After removing the health key from `argocd-cm`, the application controller required a restart to pick up the new Lua set; otherwise, stale health logic persisted.

## Remediations applied
- **Removed KafkaNodePool health override** to stop gating on a nonexistent Ready condition. Path: `sources/values/common/argocd.yaml` → deleted `resource.customizations.health.kafka.strimzi.io_KafkaNodePool`. Committed in gitops repo `ameide-gitops` main at `a00036a`.
- **Refined Kafka/KafkaTopic health scripts** to (a) gate strictly on Ready with `observedGeneration` checks, (b) include Warning conditions in the health message, and (c) avoid Lua stdlib dependencies (`string.*`, `table.concat`). Paths: `sources/values/common/argocd.yaml` and `environments/dev/argocd/ops/argocd-cm.yaml`. Committed in gitops repo at `06d22eb` / `929cb57`.
- **Propagated to cluster:** Patched `argocd-cm` to pick up the new scripts and restarted `argocd-application-controller` StatefulSet. Performed hard refreshes on `data-kafka-cluster` and related apps to recalc health.
- **Aligned Strimzi CRDs with vendored 0.48.0 definitions** by replacing the live `kafkas.kafka.strimzi.io` CRD (and siblings) with the manifests from `files/strimzi/040-049` using `kubectl replace` (preferred over `apply` for large CRDs), so the API-server state matches Git aside from canonicalization.
- **Scoped CRD diff noise** by adding `ignoreDifferences` entries for the Strimzi CRDs in `environments/dev/components/data/crds/strimzi/component.yaml`, targeting `.spec.versions`. This keeps Argo from reporting perpetual OutOfSync on CRDs caused by API-server rewriting of version blocks, while still enforcing presence and non-CRD fields via the app and the upgrade runbook.
- **Result:** `data-kafka-cluster` now Healthy; Kafka CR Healthy and gates the wave; KafkaNodePool no longer evaluated for health. `data-crds-strimzi` converges to Synced/Healthy once the CRD schemas are aligned and the ignore rules are in place. Orphan warnings remain (unchanged).

## Operational checklist (repeatable)
1. **Config change:** Ensure `sources/values/common/argocd.yaml` no longer defines `resource.customizations.health.kafka.strimzi.io_KafkaNodePool`. Deploy Argo CD so `argocd-cm` matches.
2. **Reload controller:** `kubectl -n argocd rollout restart sts argocd-application-controller` and wait for readiness.
3. **Refresh app:** Hard refresh `data-kafka-cluster` (annotation `argocd.argoproj.io/refresh: hard` or CLI `argocd app get` followed by refresh) to clear cached health.
4. **Verify:** `kubectl get app data-kafka-cluster -n argocd -o jsonpath='{.status.health.status}'` should read `Healthy`; `argocd app resources data-kafka-cluster` shows Kafka/KafkaNodePool Synced and non-orphaned. Orphaned Secrets remain unless separately ignored.

## Troubleshooting notes
- If health stays Progressing after removal, confirm the CM no longer contains the key: `kubectl -n argocd get cm argocd-cm -o json | jq -r '.data | keys[]' | grep KafkaNodePool` (should return nothing). Then restart the controller and hard refresh the app.
- Kafka cluster readiness is tracked on the `Kafka` CR (`status.conditions[type=Ready]==True`); do not add Ready-based checks to KafkaNodePool—Strimzi does not publish them.
- Orphan warnings can be silenced via project-level ignores if desired; they are unrelated to Strimzi health.

## Current state
- `data-kafka-cluster`: Synced/Healthy.  
- `foundation-strimzi-operator`: Synced/Healthy.  
- `data-crds-strimzi`: Synced/Healthy (Strimzi CRDs match vendored 0.48.0; Argo ignores server-side changes under `.spec.versions` for the Strimzi CRDs via targeted `ignoreDifferences`).  
- Orphan warnings: still present (11 Secrets); unchanged by this work.

## Follow-ups (optional)
- Add a replicas-based KafkaNodePool health script only if needed (check `status.replicas` vs `spec.replicas`), but keep Kafka Ready as the authoritative signal.
- Keep Strimzi CRDs in sync with the operator version by updating `files/strimzi/040-049` from the corresponding tag in the Strimzi repo and re-running `kubectl replace` plus an Argo sync on `data-crds-strimzi` during each upgrade.
- Consider project-level orphan ignores for the repeating Secrets if they are expected operator outputs.
- Document the vendor stance: KafkaNodePool has status but no Ready condition; Kafka CR carries the Ready condition. Keep Argo customizations aligned with that contract when upgrading Strimzi.
