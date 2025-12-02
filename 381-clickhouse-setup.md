# backlog/381 – ClickHouse setup (Argo + Altinity chart)

## Components & sources
- Operator: `foundation-clickhouse-operator` (Altinity operator chart 0.25.5, vendored under `gitops/ameide-gitops/sources/charts/third_party/altinity/altinity-clickhouse-operator/0.25.5`).
- Cluster: `data-clickhouse` (Altinity `clickhouse` chart 0.3.6, vendored under `sources/charts/third_party/altinity/clickhouse/clickhouse`).
- Secrets: `data-clickhouse-secrets` (ExternalSecret bundle that materialises `secret/clickhouse-auth`).

## Argo wiring
- Project: `data-services`.
- Waves: secrets wave 21 (`data-clickhouse-secrets`), cluster wave 22 (`data-clickhouse`).
- Dependencies: `data-clickhouse` depends on `data-clickhouse-secrets`; operator lives in wave 15.
- Sync options: `CreateNamespace=true`, `RespectIgnoreDifferences=true` for both; CH app also uses `SkipDryRunOnMissingResource=true`.

## Secret contract (guardrails v2)
- Source: Vault via `ClusterSecretStore/ameide-vault`.
- ExternalSecret target: `Secret/clickhouse-auth` in `ameide`.
- Required keys in target Secret:
  - `password` (default user for Altinity chart `defaultUser.password_secret_name`).
  - `adminUsername`, `adminPassword` (legacy consumers/smoke).
  - `langfuseUsername`, `langfusePassword`.
  - `plausibleUsername`, `plausiblePassword`.
- Vault keys referenced (shared values): `clickhouse-admin-username`, `clickhouse-admin-password`, `langfuse-clickhouse-password`, `plausible-clickhouse-password`.

## Chart values (shared defaults)
- `clickhouse.replicasCount=1`, `shardsCount=1`.
- Persistence: enabled, `20Gi`, `ReadWriteOnce`, `storageClass=managed-premium` (dev overlay shrinks to 5Gi/local-path).
- Image: `clickhouse/clickhouse-server:head-alpine`.
- Resources: req `200m/1Gi`, limits `1000m/2Gi` (dev: `100m/1Gi` req, `300m/2Gi` limits).
- Operator subchart: disabled (`clickhouse.operator.enabled=false`; operator deployed separately).
- Users use `password_secret_name: clickhouse-auth`; default user password also from `clickhouse-auth`.

## Validation steps
1) Sync order: `foundation-clickhouse-operator` → `data-clickhouse-secrets` → `data-clickhouse`.
2) Verify Secret: `kubectl -n ameide get secret clickhouse-auth -o jsonpath='{.data}' | jq` contains `password`, `langfusePassword`, `plausiblePassword` (base64).
3) Wait for pods: `kubectl wait --for=condition=Ready pod -n ameide -l clickhouse.altinity.com/chi=clickhouse --timeout=600s`.
4) Auth probe: exec into a ClickHouse pod and run `clickhouse-client --host 127.0.0.1 --user langfuse --password "$PW" --query 'SELECT 1'` (repeat for plausible).
5) Run Argo smoke: `shared-values/tests/data-plane-core` ClickHouse section should pass.

## Issues encountered & resolutions
- SSA panic in Argo: enabling `ServerSideApply=true` caused `runtime error: invalid memory address or nil pointer dereference` during sync (Argo controller SSA/diff bug). Resolution: disable SSA for ClickHouse apps, rely on client-side apply + `SkipDryRunOnMissingResource`.
- Rolling tag pulled and failed: `clickhouse/clickhouse-server:head-alpine` hit `ErrImagePull` when upstream tag disappeared. Resolution: pin to `25.10.2-alpine` (available tag) and push the submodule update.
- Bundled operator and CHI in one app: this deviated from vendor guidance and complicated upgrades. Resolution: disable in-chart operator (`clickhouse.operator.enabled=false`), keep `foundation-clickhouse-operator` as separate Argo app, and leave `data-clickhouse` to manage only the CHI.
- Stale Argo operation error: old SSA panic message persisted even after healthy state. Resolution: run a fresh sync after removing SSA; health turned green and message cleared.

## Notes / follow-ups
- The Altinity chart requires the `password` key; keep the ExternalSecret template aligned if Vault key names change.
- If enabling replicas >1, enable keeper (`clickhouse-keeper`) or point `clickhouse.keeper.host` to an external keeper/zookeeper.
- Consider adding `PruneLast=true` on the Application to further protect PVC ordering if we expand to multi-node layouts.
