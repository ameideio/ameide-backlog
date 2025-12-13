# 420 – Temporal CNPG creds + dev registry rollout (runbook and incident notes)

> **Cross-References (Deployment Architecture Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | How data-temporal apps are generated |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Rollout phases: 265 (migrations), 450/455 (runtime/bootstrap) |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Chart at `sources/charts/third_party/temporal/` |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | CNPG credential ownership pattern |
>
> **Related**:
> - [423-temporal-argocd-recovery.md](423-temporal-argocd-recovery.md) – ArgoCD recovery procedures
> - [425-vendor-schema-ownership.md](425-vendor-schema-ownership.md) – Vendor schema patterns
> - [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md) – DevContainer bootstrap
> - [456-ghcr-mirror.md](456-ghcr-mirror.md) – GHCR mirroring for Temporal images

> ⚠️ **Legacy workflow:** References to the k3d dev registry (`k3d-ameide.localhost:5001`) describe the previous local-cluster model. With the remote-first pivot (backlog/435), dev builds push directly to ACR and run on AKS; keep these notes only for historical incident context.
>
> **Bootstrap location:** When this runbook mentions `tools/bootstrap/bootstrap-v2.sh`, substitute the GitOps bootstrap now located at `ameide-gitops/bootstrap/bootstrap.sh`. Developer DevContainers only run `tools/dev/bootstrap-contexts.sh` to point kubectl/Telepresence/argocd at the shared AKS cluster.

> ✅ **Current (operator-based) deployment:** Temporal is now managed by the community Temporal operator (`cluster-temporal-operator`). Per-environment Temporal clusters and namespaces are defined declaratively via `TemporalCluster`/`TemporalNamespace` resources deployed by the `data-temporal` app (`sources/charts/platform/temporal-cluster`). The older Helm chart + `data-migrations-temporal` + `data-temporal-namespace-bootstrap` flow is deprecated and removed.

## What we configured
- **Temporal DB ownership:** CNPG (`platform-postgres-clusters`) owns the `temporal` and `temporal_visibility` roles/secrets. The TemporalCluster config uses the CNPG-managed Secrets `temporal-db-env` / `temporal-visibility-db-env` for Postgres passwords and connection URLs.
- **Schema ownership:** The Temporal operator owns schema init/updates via its reconcile loop (setup/update Jobs driven by `TemporalCluster.spec.version`).
- **Namespace management:** Temporal namespaces are created declaratively via `TemporalNamespace` CRs (packaged with `data-temporal`).
- **Dev registry wiring:** k3d local registry `k3d-ameide.localhost:5001/ameide` is the single endpoint for dev GitOps. Builds push via the host-published port `localhost:5001` (configurable via `AMEIDE_REGISTRY_PUSH_HOST`) to avoid DNS-to-container IP hangs.
- **Image naming:** Build script enforces hyphenated image tags (`agents-runtime`, `www-ameide`, `www-ameide-platform`, etc.) even if service directories use underscores. Filters accept either form.
- **Argo CD apps:** Temporal is deployed by:
  - `cluster-crds-temporal-operator` + `cluster-temporal-operator` (CRDs + controller), and
  - `{env}-data-temporal` (TemporalCluster + TemporalNamespace CRs per environment namespace).

In dev, the **normal path** is now: open the DevContainer, let the GitOps bootstrap (`ameide-gitops/bootstrap/bootstrap.sh`) converge the cluster (see `backlog/429-devcontainer-bootstrap.md` for legacy context), and rely on Argo’s automated sync + retry + RollingSync to bring Temporal apps healthy. The CLI `argocd app sync` steps below remain the manual recovery path when something is stuck (see also `backlog/423-temporal-argocd-recovery.md`).

## Issues we hit and fixes
- **Temporal CrashLoopBackOff (permission denied on `schema_version`):**
  - **Cause:** Temporal pods were using the generic `dbuser`, which lacked grants on `schema_version`; sometimes the schema table was absent after fresh cluster brings.
  - **Fix:** Switched chart values to the dedicated roles (`temporal`, `temporal_visibility`), removed Vault-driven secrets, and relied on CNPG-managed secrets. Ensured the Temporal schema hook (`data-migrations-temporal`) and namespace bootstrap run after sync (manual `argocd app sync data-migrations-temporal data-temporal{,-namespace-bootstrap}`).
- **Temporal CrashLoopBackOff (relation `schema_version` does not exist):**
  - **Cause:** Fresh cluster without Temporal schemas initialized (Temporal migrations/bootstrap not yet applied).
  - **Fix:** Sync `data-migrations-temporal`, then `data-temporal` and `data-temporal-namespace-bootstrap`; confirm `schema_version` exists via `psql` against CNPG. Post-sync, all Temporal deployments went Running.
- **Dev registry push hangs / missing images:**
  - **Cause:** Devcontainer resolving `k3d-ameide.localhost` to the registry container IP (no host port), and underscore image names diverging from GitOps values.
  - **Fix:** Build script now defaults push endpoint to `localhost:5001` while tagging `k3d-ameide.localhost:5001/ameide/<image>:dev`; enforces hyphenated tags; adds service filters; documents usage in README/backlog/415.
- **Temporal schema job reran endlessly (duplicate `namespace_metadata` PK):**
  - **Cause:** Because Argo self-healed deleted hook resources, the Helm hook job re-created itself after every sync. After the first successful `setup-schema`, the second rerun hit `duplicate key value violates unique constraint "namespace_metadata_pkey"` and exited before `update-schema` could bump the version, leaving Temporal stuck at `0.0`.
  - **Fix:** Disabled `prune`/`selfHeal` for the `data-migrations-temporal` Application so the hook only runs when we explicitly sync or bump the chart. Added the manual cleanup procedure below to clear any partially-initialized tables before the next intentional sync.

## Reproducible rollout (dev)
1) Pull repo + submodule: `git pull --recurse-submodules && git submodule update --init --recursive`.
2) Bootstrap dev cluster: `ameide-gitops/bootstrap/bootstrap.sh --config bootstrap/configs/dev.yaml` (builds + pushes dev images to the designated registry).
3) Port-forward and log into ArgoCD:
   ```
   kubectl -n argocd port-forward svc/argocd-server 8080:80 >/tmp/argocd-pf.log 2>&1 &
   ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
   argocd login localhost:8080 --username admin --password "$ARGOCD_PASSWORD" --insecure --grpc-web
   ```
4) If Temporal remains degraded after bootstrap, or you want to force a clean recovery, sync Temporal schema + apps and wait for health:
   ```
   argocd app sync cluster-crds-temporal-operator cluster-temporal-operator --grpc-web --server localhost:8080
   argocd app sync dev-data-temporal --grpc-web --server localhost:8080
   argocd app wait dev-data-temporal --health --timeout 300 --grpc-web --server localhost:8080
   ```
5) Verify pods: `kubectl get pods -n ameide | grep temporal` (all Running; namespace bootstrap Completed). DB check (optional): `psql -U temporal -d temporal -c "select * from schema_version;"` on a CNPG pod.

### Manual cleanup when `namespace_metadata_pkey` is duplicated

> Legacy only: this section applied to the removed Helm-hook migration job. With the operator-based deployment, prefer inspecting `TemporalCluster` status and operator logs.

If `data-migrations-temporal-temporal-migrations` fails with:

```
duplicate key value violates unique constraint "namespace_metadata_pkey"
```

the schema setup phase re-ran after partially initializing the default namespace. Clean up and rerun the hook once:

1. Delete the stuck hook job so we start from a clean slate:
   ```
   kubectl -n ameide delete job data-migrations-temporal-temporal-migrations
   ```
2. Remove the duplicate namespace row from CNPG (pick the primary pod for `platform-postgres-clusters`):
   ```
   CNPG=$(kubectl -n ameide get pods -l cnpg.io/cluster=platform-postgres-clusters,role=primary -o jsonpath='{.items[0].metadata.name}')
   kubectl -n ameide exec -it "${CNPG}" -- psql -U temporal -d temporal -c \
     "delete from namespace_metadata where namespace_id='namespace|default';"
   ```
3. Trigger the hook exactly once:
   ```
   argocd app sync data-migrations-temporal --grpc-web --server localhost:8080
   ```
   Wait for the hook pod to finish and confirm `temporal-sql-tool version` now matches the chart’s expected version (currently `1.18`).
4. Re-sync the runtime and namespace bootstrap apps as in step 4 above.

## Follow-ups / hardening
- Auto-sync + retry for `data-temporal*` in dev is now configured via the shared ApplicationSet template. Manual `argocd app sync`/`app wait` is reserved for explicit recovery (especially after schema or CNPG incidents).
- Add a guardrail smoke to fail fast if `schema_version` is missing or if Temporal roles lack table privileges (e.g., simple readiness check pod).
- Keep dev Tilt registry config separate; do not point Tilt at `k3d-ameide.localhost` unless intentionally sharing artifacts with Argo.

For related design and runbooks, see:

- `backlog/425-vendor-schema-ownership.md` – vendor vs Flyway schema ownership (Temporal pattern).
- `backlog/423-temporal-argocd-recovery.md` – detailed recovery flows when Temporal or namespace bootstrap fail in Argo.
- `backlog/429-devcontainer-bootstrap.md` – how the DevContainer + `bootstrap-v2.sh` orchestrate k3d, Argo, and GitOps end-to-end.
- `backlog/456-ghcr-mirror.md` – GHCR mirroring for Docker Hub rate limiting (Temporal uses GHCR mirrors for server, admin-tools, and UI images).
