# 420 – Temporal CNPG creds + dev registry rollout (runbook and incident notes)

> **Cross-References (Deployment Architecture Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | How data-temporal apps are generated |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Rollout phases: 265 (migrations), 450/455 (runtime/bootstrap) |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Temporal charts: `sources/charts/platform/temporal-cluster` (TemporalCluster/TemporalNamespace) and `sources/charts/third_party/alexandrevilain/temporal-operator` (operator) |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | CNPG credential ownership pattern |
>
> **Related**:
> - [423-temporal-argocd-recovery.md](423-temporal-argocd-recovery.md) – ArgoCD recovery procedures
> - [425-vendor-schema-ownership.md](425-vendor-schema-ownership.md) – Vendor schema patterns
> - [435-remote-first-development.md](435-remote-first-development.md) – Canonical DevContainer + Telepresence workflow
> - [456-ghcr-mirror.md](456-ghcr-mirror.md) – GHCR mirroring for Temporal images

> ⚠️ **Legacy workflow:** References to the k3d dev registry (`k3d-ameide.localhost:5001`) describe the previous local-cluster model. With the remote-first pivot (backlog/435), dev builds push directly to ACR and run on AKS; keep these notes only for historical incident context.
>
> **Bootstrap location:** When this runbook mentions `tools/bootstrap/bootstrap-v2.sh`, substitute the GitOps bootstrap now located at `ameide-gitops/bootstrap/bootstrap.sh`. Developer DevContainers only run `tools/dev/bootstrap-contexts.sh` to point kubectl/Telepresence/argocd at the shared AKS cluster.

> ✅ **Current (operator-based) deployment:** Temporal is now managed by the community Temporal operator (`cluster-temporal-operator`). Per-environment Temporal clusters and namespaces are defined declaratively via `TemporalCluster`/`TemporalNamespace` resources deployed by the `data-temporal` app (`sources/charts/platform/temporal-cluster`). The older Helm chart + `data-migrations-temporal` + `data-temporal-namespace-bootstrap` flow is deprecated and removed.

## What we configured
- **Temporal DB ownership:** CNPG (`platform-postgres-clusters`) owns the `temporal` and `temporal_visibility` roles/secrets. The TemporalCluster config uses the CNPG-managed Secrets `temporal-db-env` / `temporal-visibility-db-env` for Postgres passwords and connection URLs.
- **Schema ownership:** Temporal owns its schema (we do not manage Temporal tables via Flyway). For deterministic GitOps on fresh clusters, `data-temporal` runs a `temporal-db-schema` Argo `PreSync` Job that executes `temporal-sql-tool setup-schema` + `update-schema` using the vendor admin-tools image.
- **Namespace management:** Temporal namespaces are created declaratively via `TemporalNamespace` CRs (packaged with `data-temporal`).
- **DB preflight (self-healing):** `data-temporal` runs a `temporal-db-preflight` Argo `PreSync` hook to wait for Postgres and ensure the metadata partition row exists (`namespace_metadata.partition_id=54321`). This removes the need for manual SQL in the normal rollout path.
- **Webhook CA injection:** The Temporal operator uses admission webhooks and depends on cert-manager CA injection in its namespace (`ameide-system`). This is provided by the cluster-shared cert-manager install (`cluster-cert-manager` in `cert-manager`; CRDs via `cluster-crds-cert-manager`).
- **Digest-pinned Temporal images:** GitOps deploys Temporal server/UI/admin-tools by digest per `backlog/602-image-pull-policy.md`. This requires the operator to treat `repo@sha256:...` as complete (no `:version` appending).
- **Patched operator image + pull secret:** The Temporal operator is deployed from `ghcr.io/ameideio/temporal-operator` (patched to support digest refs) and therefore must have `imagePullSecrets: [{name: ghcr-pull}]` so it can pull from GHCR.
- **Dev registry wiring:** k3d local registry `k3d-ameide.localhost:5001/ameide` is the single endpoint for dev GitOps. Builds push via the host-published port `localhost:5001` (configurable via `AMEIDE_REGISTRY_PUSH_HOST`) to avoid DNS-to-container IP hangs.
- **Image naming:** Build script enforces hyphenated image tags (`agents-runtime`, `www-ameide`, `www-ameide-platform`, etc.) even if service directories use underscores. Filters accept either form.
- **Argo CD apps:** Temporal is deployed by:
  - `cluster-crds-temporal-operator` + `cluster-temporal-operator` (CRDs + controller), and
  - `{env}-data-temporal` (TemporalCluster + TemporalNamespace CRs per environment namespace).

In dev, the **normal path** is now: open the DevContainer (per `backlog/435-remote-first-development.md`), let the GitOps bootstrap (`ameide-gitops/bootstrap/bootstrap.sh`) converge the cluster, and rely on Argo’s automated sync + retry + RollingSync to bring Temporal apps healthy. The CLI `argocd app sync` steps below remain the manual recovery path when something is stuck (see also `backlog/423-temporal-argocd-recovery.md`).

## Issues we hit and fixes
- **Temporal CrashLoopBackOff (schema compatibility check / schema missing):**
  - **Cause:** Temporal schema setup/update Jobs (owned by the operator) failed or were blocked by Postgres schema ownership/permissions on a fresh cluster.
  - **Fix:** Inspect operator-owned schema pods (`temporal-setup-*`, `temporal-update-*`) and their logs; fix any DB ownership issues (CNPG) and let the operator rerun schema setup/update.
- **Temporal operator webhook failures (CR create/update blocked, x509/CA bundle errors):**
  - **Cause:** Missing/unsynced `cluster-cert-manager` (or missing cert-manager CRDs), preventing CA injection into the operator webhook configurations.
  - **Fix:** Sync `cluster-crds-cert-manager` + `cluster-cert-manager`, then `cluster-crds-temporal-operator` + `cluster-temporal-operator`; verify webhook `caBundle` is injected and the operator pods are Running.
- **Temporal operator `ImagePullBackOff` (401 Unauthorized from GHCR):**
  - **Cause:** Operator chart was pointed at `ghcr.io/ameideio/temporal-operator` but did not set `imagePullSecrets`, so nodes attempted anonymous pulls.
  - **Fix (GitOps):** Ensure `sources/values/_shared/cluster/temporal-operator.yaml` sets `imagePullSecrets: [{name: ghcr-pull}]`, and that `local-foundation-ghcr-pull-secret` is Healthy before the operator rollouts.
- **Dev registry push hangs / missing images:**
  - **Cause:** Devcontainer resolving `k3d-ameide.localhost` to the registry container IP (no host port), and underscore image names diverging from GitOps values.
  - **Fix:** Build script now defaults push endpoint to `localhost:5001` while tagging `k3d-ameide.localhost:5001/ameide/<image>:dev`; enforces hyphenated tags; adds service filters; documents usage in README/backlog/415.
- **Temporal server panic (`initial versions have duplicates`):**
  - **Cause:** Temporal DB has multiple cluster names in `cluster_metadata_info` (often leftover from a temporary rename experiment).
  - **Fix (GitOps-first):** In local/dev, enable `data-temporal.preflight.autoFix.clusterMetadataInfo` and allowlist the stale name (e.g. `active`) so the `temporal-db-preflight` hook can self-heal. In staging/prod, keep fail-fast by default; treat this as an incident and apply a temporary allowlist+autofix change via PR if appropriate.

## Reproducible rollout (dev)
1) Pull repo + submodule: `git pull --recurse-submodules && git submodule update --init --recursive`.
2) Bootstrap dev cluster: `ameide-gitops/bootstrap/bootstrap.sh --config bootstrap/configs/dev.yaml` (builds + pushes dev images to the designated registry).
3) Port-forward and log into ArgoCD:
   ```
   kubectl -n argocd port-forward svc/argocd-server 8443:443 >/tmp/argocd-pf-8443.log 2>&1 &
   ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
   argocd login localhost:8443 --username admin --password "$ARGOCD_PASSWORD" --plaintext --grpc-web
   ```
4) If Temporal remains degraded after bootstrap, or you want to force a clean recovery, sync Temporal schema + apps and wait for health:
   ```
   argocd app sync cluster-crds-cert-manager cluster-cert-manager cluster-crds-temporal-operator cluster-temporal-operator --grpc-web --server localhost:8443 --plaintext
   argocd app sync dev-data-temporal --grpc-web --server localhost:8443 --plaintext
   argocd app wait dev-data-temporal --health --timeout 300 --grpc-web --server localhost:8443 --plaintext
   ```
5) Verify pods: `kubectl get pods -n <env-namespace> | grep temporal` (all Running). DB check (optional): `psql -U temporal -d temporal -c "select * from schema_version;"` on a CNPG pod.

## Follow-ups / hardening
- Auto-sync + retry for `data-temporal*` in dev is now configured via the shared ApplicationSet template. Manual `argocd app sync`/`app wait` is reserved for explicit recovery (especially after schema or CNPG incidents).
- Add a guardrail smoke to fail fast if `schema_version` is missing or if Temporal roles lack table privileges (e.g., simple readiness check pod).
- Keep dev Tilt registry config separate; do not point Tilt at `k3d-ameide.localhost` unless intentionally sharing artifacts with Argo.

For related design and runbooks, see:

- `backlog/425-vendor-schema-ownership.md` – vendor vs Flyway schema ownership (Temporal pattern).
- `backlog/423-temporal-argocd-recovery.md` – detailed recovery flows when Temporal or namespace bootstrap fail in Argo.
- `backlog/429-devcontainer-bootstrap.md` – legacy local k3d bootstrap context (`bootstrap-v2.sh`) retained for historical incident notes.
- `backlog/456-ghcr-mirror.md` – GHCR mirroring for Docker Hub rate limiting (Temporal uses GHCR mirrors for server, admin-tools, and UI images).
