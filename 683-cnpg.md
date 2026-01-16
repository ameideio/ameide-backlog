# 683 – CloudNativePG (CNPG) platform notes

**Status:** Living doc  
**Scope:** CNPG operator + `postgres-ameide` cluster (AKS)

This document centralizes how PostgreSQL is deployed/operated via CloudNativePG (CNPG) in this repo, with emphasis on GitOps constraints, Azure Disk RWO behavior, and the few sharp edges we’ve hit.

## CNPG suite (cross-references)

- `683-cnpg.md` (this doc)
- `412-cnpg-owned-postgres-greds.md` (credential ownership “north star”)
- `440-storage-concerns.md` (storage tiers + backups plan)
- `684-aks-node-pool-strategy-option-b.md` (workload-class pools; CNPG placement on `platform`)
- `420-temporal-cnpg-dev-registry-runbook.md` (Temporal usage of CNPG DBs/users + recovery)
- `489-pgadmin-keycloak-oidc.md` (pgAdmin + CNPG `servers.json` generation)
- `531-local-cnpg-placement-vs-control-plane-taints.md` (local CNPG scheduling/PV binding)

## Where configuration lives (GitOps source of truth)

- **CNPG operator config (cluster-scoped):** `sources/charts/shared-values/operators/cloudnative-pg.yaml`
- **CNPG CRDs (vendored):** `sources/charts/foundation/common/raw-manifests/files/cnpg-crds.yaml`
- **Postgres cluster + roles + DBs + (optional) poolers:** `sources/charts/foundation/operators-config/postgres_clusters`
- **Platform cluster values (shared across envs):** `sources/values/_shared/data/platform-postgres-clusters.yaml`
- **Env overrides (if present):** `sources/values/env/<env>/data/platform-postgres-clusters.yaml`

Operational constraint (cloud): changes must go via Git → CI → ArgoCD reconcile. Read-only inspection via `kubectl get/describe/logs` is OK.

## Azure Disk “Multi-Attach”: root cause

AKS “managed-csi” Azure Disks are **ReadWriteOnce (RWO)**: a disk can be attached to only one node at a time.

If Kubernetes tries to start a CNPG instance pod on a different node while the disk is still attached to the previous node (or detach is still in progress), you get **Multi-Attach** and the pod cannot start until detach completes.

## Update strategy: reduce primary disk moves

In `sources/values/_shared/data/platform-postgres-clusters.yaml` we set:

- `cluster.primaryUpdateMethod: switchover`

Rationale: for RWO disks, `switchover` promotes a replica that already has its own disk attached, rather than pushing more restarts/reschedules of the primary.

**Vendor caveat to remember:** with `primaryUpdateMethod: switchover`, CNPG may reject an update if you change **the Postgres image and Postgres parameters** in the same reconciliation. Prefer sequential rollouts when possible.

## Poolers (PgBouncer): naming rules + current state

CNPG reserves default service names:

- `<cluster>-rw`, `<cluster>-ro`, `<cluster>-r`

We previously had a Pooler name collide with `<cluster>-rw` (`postgres-ameide-rw`), causing repeated `InvalidOwnership` events and preventing the pooler from reconciling.

**Current state:** poolers are disabled in shared values:

- `sources/values/_shared/data/platform-postgres-clusters.yaml` sets `poolers: []`

**When reintroducing poolers:** choose a non-reserved name (e.g. `postgres-ameide-pooler-rw`) and explicitly migrate clients to the pooler service (do not overload `postgres-ameide-rw`).

## Readiness tuning (avoid flapping)

We keep CNPG readiness timeout at vendor-default-ish levels:

- `sources/values/_shared/data/platform-postgres-clusters.yaml`: `cluster.probes.readiness.timeoutSeconds: 5`

The 1s timeout was too aggressive under normal jitter (I/O, kube-proxy, DNS, etc.).

## Credentials / Secrets model (and drift)

The current shared values file describes an “operator-first” setup:

- Vault KV holds stable passwords (per environment)
- External Secrets Operator renders Kubernetes Secrets
- CNPG consumes those Secrets via `managed.roles[].passwordSecret`

Separately, `412-cnpg-owned-postgres-greds.md` documents the **north star**: CNPG should be authoritative for DB credentials; Vault can mirror but should not drive. Treat that as the direction of travel when changing credential flows.

The `postgres_clusters` chart also contains an optional `passwordReconcile` CronJob (`sources/charts/foundation/operators-config/postgres_clusters/templates/password-reconcile-job.yaml`) that can force DB role passwords to match Secrets. It should remain **off** unless explicitly required for a controlled migration.

## Backups vs “WAL archiving is working”

CNPG can report “continuous archiving is working” without you having a durable backup/restore story declared in Git. Do not treat “WAL archiving OK” as “we have backups”.

Intended (GitOps + IaC) path is in `440-storage-concerns.md`:

1. Provision Azure backup storage account + identity in IaC.
2. Enable CNPG backups in values (prefer vendor plugin approach).
3. Add scheduled backups + retention.
4. Verify backups exist and restores are viable.

## Quick read-only checks

- Cluster summary: `kubectl -n <ns> get cluster postgres-ameide`
- Pod roles: `kubectl -n <ns> get pods -l cnpg.io/cluster=postgres-ameide -L role`
- Events (Multi-Attach / ownership): `kubectl -n <ns> get events --sort-by=.lastTimestamp | tail -n 50`
- Volume attachments: `kubectl get volumeattachment | rg -i 'postgres-ameide|pvc-'`
- Services (reserved names): `kubectl -n <ns> get svc | rg postgres-ameide`
- Poolers (should be empty while disabled): `kubectl -n <ns> get pooler`
