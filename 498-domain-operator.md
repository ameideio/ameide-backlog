# 498 – Domain Operator

**Status:** Active (Phase 1 Complete)
**Audience:** Platform engineers implementing Domain operator
**Scope:** Implementation tracking, development phases, acceptance criteria

> **Related**:
> - [502-domain-vertical-slice.md](502-domain-vertical-slice.md) – End-to-end implementation guide
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – Go patterns & reference implementation

## Grounding & contract alignment

- **Primitive/operator model:** Implements the Domain primitive/operator responsibilities implied by `470-ameide-vision.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, and `477-primitive-stack.md`, using the shared operator patterns from `495-ameide-operators.md` and `497-operator-implementation-patterns.md`.  
- **Vertical slice & CLI:** Instantiates the Domain side of the vertical-slice pattern in `502-domain-vertical-slice.md` and is packaged via `503-operators-helm-chart.md`, exposing status/conditions that `484a-484f` CLI workflows (`ameide primitive describe/verify/...`) consume.  
- **Scrum/Transformation usage:** Provides the control plane for Domain primitives such as the Transformation Scrum domain (`300-400/367-1-scrum-transformation.md`, `508-scrum-protos.md`), which participate in the Scrum seam defined by `506-scrum-vertical-v2.md` and the agent stack in `505-agent-developer-v2*.md`.

---

## 1. Overview

The Domain operator manages the lifecycle of **Domain primitives** – the bounded contexts that hold business data and expose gRPC APIs. Each `Domain` CR results in:

- Database schema + migration Job (via CNPG)
- Application Deployment with gRPC server
- Service (ClusterIP) for internal traffic
- HPA, NetworkPolicy, ServiceMonitor

**Key insight**: The operator owns all runtime infrastructure; the Domain image owns all business logic.

---

## 2. CRD Summary

```yaml
apiVersion: ameide.io/v1
kind: Domain
metadata:
  name: orders
spec:
  image: ghcr.io/ameide/orders-domain:1.12.3
  db:
    clusterRef: cnpg/orders
    schema: orders
    migrationJobImage: ghcr.io/ameide/migrator:latest
  resources:
    cpu: "500m"
    memory: "512Mi"
  env: [...]
  observability:
    logLevel: info
  security:
    serviceAccountName: orders-domain
    networkProfile: backend
  rollout:
    strategy: RollingUpdate
status:
  conditions:
    - type: Ready
    - type: DBReady
    - type: WorkloadReady
    - type: MigrationSucceeded
  observedGeneration: 5
  dbMigrationVersion: "V3__add_status_column"
  readyReplicas: 2
```

Full CRD Go types: [497 §10.1](497-operator-implementation-patterns.md#101-crd-types-go)

---

## 3. Reconcile Flow

```
1. Fetch Domain CR
2. Handle deletion (finalizer cleanup)
3. Ensure finalizer present
4. Validate spec
5. reconcileDB()       ← Migration Job, wait for completion
6. reconcileWorkload() ← Deployment, Service, HPA, NetworkPolicy, ServiceMonitor
7. Update status (conditions, observedGeneration)
```

Reference implementation: [497 §10.3](497-operator-implementation-patterns.md#103-main-reconcile-flow)

---

## 4. Development Phases

### Phase 1: Core Reconciliation ✅ IMPLEMENTED

> **Implementation**: See [operators/domain-operator/](../operators/domain-operator/)

| Task | Description | Status |
|------|-------------|--------|
| **CRD types** | Define `Domain`, `DomainSpec`, `DomainStatus` in Go | ✅ `api/v1/domain_types.go` |
| **Basic reconciler** | Implement 7-step reconcile flow skeleton | ✅ `internal/controller/domain_controller.go` |
| **Finalizer handling** | Add/remove finalizer on create/delete | ✅ Implemented |
| **Deployment creation** | `reconcileWorkload` creates Deployment | ✅ `internal/controller/reconcile_workload.go` |
| **Service creation** | Create ClusterIP Service for domain | ✅ Implemented |
| **Status conditions** | Set `Ready`, `WorkloadReady` conditions | ✅ `internal/controller/conditions.go` |

**Helm chart**: Operators deployable via unified chart at `operators/helm/`

### Phase 2: Database Integration ✅ IMPLEMENTED

> **Implementation**: See [operators/domain-operator/internal/controller/reconcile_db.go](../operators/domain-operator/internal/controller/reconcile_db.go)

| Task | Description | Status |
|------|-------------|--------|
| **Migration Job** | Create Job running `migrationJobImage` | ✅ `reconcileMigrationJob()` |
| **CNPG secret injection** | Mount CNPG credentials into Job | ✅ `buildMigrationJob()` |
| **Migration status** | Set `DBReady`, `MigrationSucceeded`/`MigrationFailed` | ✅ `handleExistingMigrationJob()` |
| **DB credentials in Deployment** | Inject PGHOST/PGUSER/etc into Domain Deployment | ✅ `getDBEnvVars()` |
| **Migration version tracking** | Extract version from Job logs/output | ⏳ Deferred (requires Job log parsing) |
| **Schema cleanup on delete** | Optional: drop schema on Domain deletion | ⏳ Deferred (configurable per-environment) |

### Phase 3: Production Readiness

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **HPA** | Create HPA if enabled in spec | HPA scales pods on load |
| **NetworkPolicy** | Create policy based on `networkProfile` | Only allowed traffic reaches pods |
| **ServiceMonitor** | Create ServiceMonitor for Prometheus | Metrics scraped, visible in Grafana |
| **Resource defaulting** | Apply default CPU/memory if not specified | Pods have resource limits |
| **Event recording** | Emit K8s events on reconcile actions | `kubectl describe domain` shows events |
| **Metrics** | Expose reconciliation metrics | Prometheus can scrape operator metrics |

### Phase 4: Testing & Validation

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Unit tests** | Test reconcile logic with fake client | `go test ./...` passes |
| **envtest tests** | Integration tests with local API server | Tests create/reconcile/delete Domain |
| **E2E tests** | Full cluster tests in CI | Domain→Deployment→Pod→healthy in real cluster |
| **CLI verify integration** | `ameide primitive verify` checks Domain status | CLI reports Domain health correctly |

---

## 5. Key Implementation Details

### 5.1 DB Credentials Flow

```
CNPG Cluster → Secret (auto-generated)
                  ↓
Domain Operator reads secret name from ClusterRef
                  ↓
Migration Job gets secret mounted as env
                  ↓
Domain Deployment gets same secret for runtime
```

### 5.2 Owned Resources

```go
// SetupWithManager
ctrl.NewControllerManagedBy(mgr).
    For(&amv1.Domain{}).
    Owns(&appsv1.Deployment{}).
    Owns(&batchv1.Job{}).
    Owns(&corev1.Service{}).
    Owns(&autoscalingv2.HorizontalPodAutoscaler{}).
    Owns(&networkingv1.NetworkPolicy{}).
    Owns(&monitoringv1.ServiceMonitor{}).
    Complete(r)
```

### 5.3 Condition Precedence

Overall `Ready` is computed as:
```
Ready = DBReady && WorkloadReady && !MigrationFailed && !Degraded
```

---

## 6. Non-Goals

| What | Why |
|------|-----|
| Entity schemas | Domain image defines tables via migrations |
| Business rules | Live in Domain service code |
| Tenant isolation | Enforced by app + DB RLS, not operator |
| Proto validation | Handled by Buf, not operator |
| Image building | CI builds images; operator just deploys |

---

## 7. Dependencies

| Dependency | Purpose |
|------------|---------|
| **CNPG** | Postgres clusters, secrets, connection pooling |
| **Prometheus/ServiceMonitor** | Metrics collection |
| **Gateway API** | (Not direct – domains are internal only) |
| **controller-runtime** | Reconciliation framework |

---

## 8. Open Questions

| Question | Status |
|----------|--------|
| Schema retention policy on delete? | TBD – configurable per-environment |
| Migration timeout handling? | TBD – Job deadline + requeue strategy |
| Multi-schema domains? | Out of scope – one schema per Domain |

---

## 9. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [446-namespace-isolation.md](446-namespace-isolation.md) | Operator deploys once per cluster |
| [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Cluster-scoped deployment via ApplicationSet |
| [503-operators-helm-chart.md](503-operators-helm-chart.md) | Helm chart for operator deployment |
| [495-ameide-operators.md](495-ameide-operators.md) | Domain operator responsibilities (§1) |
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Reference implementation (§10) |
| [477-primitive-stack.md](477-primitive-stack.md) | Domain in primitive architecture |
| [472-ameide-information-application.md](472-ameide-information-application.md) | Domain as bounded context |
