# 373 – Argo + Tilt Helm adoption (logging & pruning) **DEPRECATED** (see backlog/424-tilt-www-ameide-separate-release.md)

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

Details for the Tilt helm-adoption helper referenced in backlog/373-argo-tilt-helm-north-start-v4.md.

## What the helper does

- Annotates/labels all namespaced resources matching `app.kubernetes.io/instance=<release>` so Helm owns them (`meta.helm.sh/*`, `app.kubernetes.io/managed-by=Helm`), and strips `managedFields` to avoid SSA conflicts.
- Logs **pre- and post-adoption** workload snapshots (Deployments/StatefulSets/DaemonSets) showing names, images, and ready/total replicas so Tilt UI output makes the takeover obvious.
- Prunes duplicate workloads and services so only the canonical Tilt deployment/service remain.

## Pruning behavior

- Pruning is on by default: for each release the script lists workloads with selector `app.kubernetes.io/instance=<release>`, prefers the workload named `<release>`, otherwise keeps the first and deletes the rest. If the kept workload image differs from the expected image (`TILT_EXPECTED_IMAGE` from `TILT_IMAGE_0`), the script exits non-zero after logging the mismatch.
- Services are also deduped: if multiple Services share the release selector, the Service named `<release>` is kept (or the first if none match) and the rest are deleted.
- Adoption runs twice per Helm apply: once **before** install to annotate existing resources so Helm can take ownership without conflicts, and once **after** install to prune duplicates and enforce the expected image.

## Runbook

1) Let Tilt trigger the helper automatically (wired into `helm_app` extra env). No flags required.  
2) Watch the Tilt log stream for `[helm-adopt]` entries:
   - Pre-adoption snapshot
   - “Adopting …” lines per resource
   - Post-adoption snapshot + prune logs + optional image mismatch fail-fast  
3) To rerun manually: `NS=ameide tilt/tilt-helm-adopt.sh <apps>` (uses `TILT_EXPECTED_IMAGE` from the current Tilt image when invoked via Tilt).
