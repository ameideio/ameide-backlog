# 588 — Smoke Tests Inventory (ArgoCD PostSync hook Jobs)

**Status:** Draft (inventory)  
**Audience:** Platform/GitOps operators, runtime owners  
**Scope:** Inventory of GitOps **smoke-test components** (ArgoCD PostSync hook Jobs) that validate “cluster truth” after reconciliation.

These smokes are **not** 430 `INTEGRATION_MODE` suites; they are Argo/cluster reconciliation probes.

Related:
- 430 test infra status: `backlog/430-unified-test-infrastructure-status.md`
- Kafka/topic inventory: `backlog/587-kafka-topics-and-queues-inventory.md`

## Inventory (current)

| Smoke | Component | Values | Chart |
|---|---|---|---|
| `keda-smoke` | `environments/_shared/components/cluster/smokes/keda-smoke/component.yaml` | `sources/values/_shared/cluster/keda-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `agent-echo-v0-smoke` | `environments/_shared/components/apps/primitives/agent-echo-v0-smoke/component.yaml` | `sources/values/_shared/apps/agent-echo-v0-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `domain-transformation-v0-smoke` | `environments/_shared/components/apps/primitives/domain-transformation-v0-smoke/component.yaml` | `sources/values/_shared/apps/domain-transformation-v0-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `integration-echo-v0-smoke` | `environments/_shared/components/apps/primitives/integration-echo-v0-smoke/component.yaml` | `sources/values/_shared/apps/integration-echo-v0-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `process-ping-v0-smoke` | `environments/_shared/components/apps/primitives/process-ping-v0-smoke/component.yaml` | `sources/values/_shared/apps/process-ping-v0-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `projection-foo-v0-smoke` | `environments/_shared/components/apps/primitives/projection-foo-v0-smoke/component.yaml` | `sources/values/_shared/apps/projection-foo-v0-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `uisurface-hello-v0-smoke` | `environments/_shared/components/apps/primitives/uisurface-hello-v0-smoke/component.yaml` | `sources/values/_shared/apps/uisurface-hello-v0-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `platform-smoke` | `environments/_shared/components/apps/qa/platform-smoke/component.yaml` | `sources/values/_shared/apps/platform-smoke.yaml` | `sources/charts/foundation/platform-smoke` |
| `data-data-plane-smoke` | `environments/_shared/components/data/core/data-plane-smoke/component.yaml` | `sources/values/_shared/data/data-data-plane-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `data-data-plane-ext-smoke` | `environments/_shared/components/data/extended/data-plane-ext-smoke/component.yaml` | `sources/values/_shared/data/data-data-plane-ext-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `foundation-bootstrap-smoke` | `environments/_shared/components/foundation/bootstrap-smoke/component.yaml` | `sources/values/_shared/foundation/foundation-bootstrap-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `foundation-operators-smoke` | `environments/_shared/components/foundation/operators-smoke/component.yaml` | `sources/values/_shared/foundation/foundation-operators-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `platform-observability-smoke` | `environments/_shared/components/observability/observability-smoke/component.yaml` | `sources/values/_shared/observability/platform-observability-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `platform-auth-smoke` | `environments/_shared/components/platform/auth/auth-smoke/component.yaml` | `sources/values/_shared/platform/platform-auth-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `platform-control-plane-smoke` | `environments/_shared/components/platform/control-plane/control-plane-smoke/component.yaml` | `sources/values/_shared/platform/platform-control-plane-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |
| `platform-secrets-smoke` | `environments/_shared/components/platform/secrets/secrets-smoke/component.yaml` | `sources/values/_shared/platform/platform-secrets-smoke.yaml` | `sources/charts/foundation/helm-test-jobs` |

## How to run (any env)

Smokes are ArgoCD hook Jobs; they re-run when the owning Application **syncs** (a refresh alone updates status, but does not re-run hooks).

- List smoke Applications for local: `kubectl -n argocd get application | rg '^local-.*-smoke'`
- Force a re-run (preferred, from `ameide-gitops` repo root): `./scripts/argocd-force-sync.sh local-data-data-plane-smoke`
- Force a re-run (raw kubectl patch): `kubectl -n argocd patch application local-data-data-plane-smoke --type merge -p '{"operation":{"sync":{"revision":"<git-sha>","prune":true}}}'`
