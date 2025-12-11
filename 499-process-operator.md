# 499 – Process Operator

**Status:** Planned
**Audience:** Platform engineers implementing Process operator
**Scope:** Implementation tracking, development phases, acceptance criteria

> **Related**:
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – Go patterns & reference implementation

---

## 1. Overview

The Process operator manages the lifecycle of **Process primitives** – long-running orchestrations backed by Temporal. Each `Process` CR results in:

- ProcessDefinition fetched from Transformation Domain
- ConfigMap with compiled definition (BPMN → Temporal mapping)
- Temporal worker Deployment
- Optional CronJobs for SLA checks / cleanup

**Key insight**: The operator wires Temporal workers to definitions; the Process image implements workflow/activity logic.

---

## 2. CRD Summary

```yaml
apiVersion: ameide.io/v1
kind: Process
metadata:
  name: l2o
spec:
  image: ghcr.io/ameide/l2o-process:0.7.0
  definitionRef:
    id: L2O_v3
    tenantId: t123
  temporal:
    namespace: sales-process
    taskQueue: l2o-tq
  dependencies:
    domains:
      - name: sales
      - name: billing
    agents:
      - name: l2o-coach
  rollout:
    strategy: RollingUpdate
    maxConcurrencyPerTenant: 100
  observability:
    metricsTags:
      processType: L2O
status:
  conditions:
    - type: Ready
    - type: DefinitionResolved
    - type: WorkersReady
    - type: Degraded
  definition:
    version: 3
    checksum: "sha256:abc123..."
  observedGeneration: 2
```

---

## 3. Reconcile Flow

```
1. Fetch Process CR
2. Handle deletion (finalizer cleanup – optionally stop new workflow starts)
3. Ensure finalizer present
4. Validate spec (definitionRef, temporal namespace, dependencies exist)
5. reconcileDefinition() ← Fetch from Transformation, create ConfigMap
6. reconcileWorkers()    ← Temporal worker Deployment
7. Update status (conditions, definition.checksum, observedGeneration)
```

---

## 4. CRD Types (Go)

```go
type ProcessSpec struct {
    Image         string               `json:"image"`
    DefinitionRef ProcessDefinitionRef `json:"definitionRef"`
    Temporal      TemporalConfig       `json:"temporal"`
    Dependencies  ProcessDependencies  `json:"dependencies,omitempty"`
    Rollout       ProcessRolloutSpec   `json:"rollout,omitempty"`
    Observability ProcessObservability `json:"observability,omitempty"`
}

type ProcessDefinitionRef struct {
    ID       string `json:"id"`
    TenantID string `json:"tenantId"`
}

type TemporalConfig struct {
    Namespace string `json:"namespace"`
    TaskQueue string `json:"taskQueue"`
}

type ProcessDependencies struct {
    Domains []DependencyRef `json:"domains,omitempty"`
    Agents  []DependencyRef `json:"agents,omitempty"`
}

type ProcessStatus struct {
    Conditions         []metav1.Condition       `json:"conditions,omitempty"`
    ObservedGeneration int64                    `json:"observedGeneration,omitempty"`
    Definition         ProcessDefinitionStatus  `json:"definition,omitempty"`
}

type ProcessDefinitionStatus struct {
    Version  int    `json:"version,omitempty"`
    Checksum string `json:"checksum,omitempty"`
}
```

---

## 5. Development Phases

### Phase 1: Core Reconciliation

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **CRD types** | Define `Process`, `ProcessSpec`, `ProcessStatus` | Types compile, CRD YAML generated |
| **Basic reconciler** | 7-step reconcile skeleton | Controller logs on Process create |
| **Finalizer handling** | Add/remove finalizer | Delete blocks until cleanup done |
| **Dependency validation** | Check referenced Domains/Agents exist and are Ready | Condition `Degraded` if deps missing |
| **Worker Deployment** | Create Deployment for Temporal workers | `kubectl get deploy` shows workers |
| **Status conditions** | Set `Ready`, `WorkersReady` | Conditions reflect deployment state |

### Phase 2: Transformation Integration

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Transformation client** | gRPC client to fetch ProcessDefinition | Can call `GetProcessDefinition` RPC |
| **Definition fetching** | `reconcileDefinition()` fetches by ID | Definition JSON retrieved |
| **ConfigMap creation** | Store definition in ConfigMap | ConfigMap contains definition |
| **Checksum tracking** | Compute SHA256 of definition | `status.definition.checksum` set |
| **Version tracking** | Extract version from definition | `status.definition.version` set |
| **Definition refresh** | Re-fetch on spec change or periodic | ConfigMap updated when definition changes |

### Phase 3: Temporal Wiring

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Temporal namespace validation** | Check namespace exists before deploying | Error if namespace doesn't exist |
| **Env injection** | Set `TEMPORAL_NAMESPACE`, `TASK_QUEUE`, `PROCESS_DEFINITION_ID` | Worker container has correct env |
| **ConfigMap mount** | Mount definition ConfigMap into worker pods | Workers can read definition at startup |
| **Task queue config** | Configure worker to poll correct queue | Workers register on expected queue |
| **Health check** | Ping Temporal admin API periodically | `Degraded` condition if Temporal unhealthy |

### Phase 4: Production Readiness

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Canary rollout** | Optional canary Deployment for new versions | Canary workers run alongside stable |
| **SLA CronJobs** | Create CronJobs for SLA checks if configured | CronJobs appear, run on schedule |
| **Graceful shutdown** | On delete, stop new workflow starts | No new workflows started during deletion |
| **Workflow drain** | Wait for in-flight workflows (configurable) | Clean shutdown or force-cancel per policy |
| **Metrics** | Expose operator + worker queue metrics | Prometheus scrapes process metrics |

### Phase 5: Testing & Validation

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Unit tests** | Fake Transformation client, fake k8s client | Tests pass without real services |
| **envtest tests** | Create Process, verify Deployment | Integration tests pass |
| **Temporal test server** | E2E with embedded Temporal | Workflows actually execute |
| **CLI integration** | `ameide primitive verify` checks Process | CLI reports Process health |

---

## 6. Key Implementation Details

### 6.1 Definition Fetch Flow

```
Process CR created
        ↓
reconcileDefinition() calls Transformation gRPC
        ↓
GetProcessDefinition(id, tenantId) → ProcessDefinition JSON
        ↓
Compute SHA256 checksum
        ↓
CreateOrUpdate ConfigMap with definition
        ↓
Set status.definition.checksum, status.definition.version
```

### 6.2 Reconciler Struct

```go
type ProcessReconciler struct {
    client.Client
    Scheme              *runtime.Scheme
    Recorder            record.EventRecorder
    TransformationClient transformationv1.TransformationServiceClient
    TemporalAdminClient  temporaladmin.Client // optional
}
```

### 6.3 Worker Deployment Template

```go
func mutateWorkerDeployment(deploy *appsv1.Deployment, process *amv1.Process, configMapName string) {
    deploy.Spec.Template.Spec = corev1.PodSpec{
        Containers: []corev1.Container{{
            Name:  "worker",
            Image: process.Spec.Image,
            Env: []corev1.EnvVar{
                {Name: "TEMPORAL_NAMESPACE", Value: process.Spec.Temporal.Namespace},
                {Name: "TASK_QUEUE", Value: process.Spec.Temporal.TaskQueue},
                {Name: "PROCESS_DEFINITION_ID", Value: process.Spec.DefinitionRef.ID},
            },
            VolumeMounts: []corev1.VolumeMount{{
                Name:      "definition",
                MountPath: "/etc/process-definition",
            }},
        }},
        Volumes: []corev1.Volume{{
            Name: "definition",
            VolumeSource: corev1.VolumeSource{
                ConfigMap: &corev1.ConfigMapVolumeSource{
                    LocalObjectReference: corev1.LocalObjectReference{Name: configMapName},
                },
            },
        }},
    }
}
```

### 6.4 Owned Resources

```go
ctrl.NewControllerManagedBy(mgr).
    For(&amv1.Process{}).
    Owns(&appsv1.Deployment{}).
    Owns(&corev1.ConfigMap{}).
    Owns(&batchv1.CronJob{}).
    Complete(r)
```

---

## 7. Non-Goals

| What | Why |
|------|-----|
| Workflow business logic | Lives in Process image |
| BPMN editing | Transformation Domain responsibility |
| Temporal cluster management | CNPG-style operator or managed service |
| Activity implementations | Process image responsibility |

---

## 8. Dependencies

| Dependency | Purpose |
|------------|---------|
| **Transformation Domain** | Source of ProcessDefinitions |
| **Temporal** | Workflow execution engine |
| **Domain operator** | Referenced domains must be Ready |
| **Agent operator** | Referenced agents must be Ready |

---

## 9. Open Questions

| Question | Status |
|----------|--------|
| Definition cache TTL? | TBD – periodic refresh vs event-driven |
| Workflow drain timeout? | TBD – per-process or global policy |
| Multi-tenant task queues? | Out of scope – one queue per Process |

---

## 10. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [495-ameide-operators.md](495-ameide-operators.md) | Process operator responsibilities (§2) |
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Go patterns & adaptation (§10.9) |
| [477-primitive-stack.md](477-primitive-stack.md) | Process in primitive architecture |
| [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | ProcessDefinition as design artifact |
