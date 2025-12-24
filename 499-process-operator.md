# 499 – Process Operator

**Status:** Active (Phase 1 Complete)
**Audience:** Platform engineers implementing Process operator
**Scope:** Implementation tracking, development phases, acceptance criteria

**Authority & supersession**

- This backlog is **authoritative for the Process operator control plane**: the `Process` CRD shape, reconciliation flow, and how Process workers are wired/deployed.  
- **Runtime event contracts and workflow structure** live in `506-scrum-vertical-v2.md` (Scrum seam) and any methodology-specific workflow docs; Temporal workflows must follow those contracts.  
- **Primitive/condition vocabulary** is shared with other operators and owned by `495-ameide-operators.md` and `502-domain-vertical-slice.md`.  
- Older stage or methodology docs must not override the runtime rule stated here: **workflows interact with Transformation only via bus messages defined in 506-v2/508**.

**Contract surfaces (owned elsewhere, referenced here)**

- `scrum.domain.intents.v1` / `scrum.domain.facts.v1` (Scrum domain intents/facts) and `scrum.process.facts.v1` (process facts) topics and envelopes are defined in `506-scrum-vertical-v2.md` and `508-scrum-protos.md`.  
- ProcessDefinition schema and storage are defined in the Transformation/ProcessDefinition backlogs (`367-1-*`, `471-ameide-business-architecture.md`).  
- Shared operator condition types are defined in `operators/shared/api/v1/conditions.go`.

## Runtime contract (Scrum governance workflows)

- **Domain writes via intents:** Temporal workflows that implement Scrum governance must request all Scrum state changes by publishing **domain intents** on `scrum.domain.intents.v1` (e.g., `StartSprintRequested`, `EndSprintRequested`, `CommitSprintBacklogRequested`, `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`).  
- **Domain state via facts:** They observe Scrum state by consuming **domain facts** from `scrum.domain.facts.v1` (e.g., `SprintCreated`, `SprintStarted`, `SprintBacklogCommitted`, `ProductBacklogItemDoneRecorded`, `IncrementUpdated`).  
- **Process facts only for governance cues:** Any additional events they emit are **process facts** on `scrum.process.facts.v1` (e.g., `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`, `SprintTimeboxReachedEnd`), never alternate copies of Scrum domain facts.  
- **No runtime RPC coupling:** All synchronous calls from the Process operator into Transformation (e.g., `GetProcessDefinition`) are control‑plane only; runtime workflows must not issue direct RPCs to mutate or read Scrum domain state, but instead follow the seam in `506-scrum-vertical-v2.md` / `508-scrum-protos.md`.

> **Related**:
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – Go patterns & reference implementation

---

## 1. Overview

The Process operator manages the lifecycle of **Process primitives** – long-running orchestrations backed by Temporal. Each `Process` CR results in:

- ProcessDefinition fetched from a **Definition Registry** (which may be implemented inside Transformation as a separate subsystem, but is not the Scrum domain API)
- ConfigMap with the **raw process definition** / BPMN‑annotated JSON as a design‑time reference (no runtime “BPMN → Temporal compilation” happens in the operator)
- Temporal worker Deployment
- Optional CronJobs for SLA checks / cleanup

**Key insight**: The operator wires Temporal workers to definitions and exposes them as configuration; the Process image implements workflow/activity logic and is where BPMN‑inspired designs are translated into Temporal workflows via agentic development, not by the operator at runtime.

### Scope boundaries (non-negotiable)

- The Process operator reconciles **only** the `Process` kind.
- It must not reconcile `Integration` or `Projection` workloads; those are owned by their dedicated operators.

---

## 2. CRD Summary

```yaml
apiVersion: ameide.io/v1
kind: Process
metadata:
  name: l2o
spec:
  image: ghcr.io/ameide/l2o-process:0.7.0
  imagePullPolicy: IfNotPresent # set Always only for mutable tags (transitional; see 602/603)
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
    # Optional: eventing/bus bindings (topics, clusters) are configured here
    # so Process workers can publish/subscribe to the correct `scrum.*` /
    # `process.*` streams without hardcoding environment details.
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

> **Control plane vs. runtime:** `reconcileDefinition()` is allowed to call Transformation synchronously via gRPC/Connect to resolve static `ProcessDefinition` metadata (control plane). The **runtime** Temporal workflows described in `506-scrum-vertical-v2.md` MUST interact with Transformation only via bus messages (`scrum.domain.intents.v1` and `scrum.domain.facts.v1`) and must not issue direct RPCs to mutate or read domain state.

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

### Phase 1: Core Reconciliation ✅ IMPLEMENTED

> **Implementation**: See [operators/process-operator/](../operators/process-operator/)

| Task | Description | Status |
|------|-------------|--------|
| **CRD types** | Define `Process`, `ProcessSpec`, `ProcessStatus` | ✅ `api/v1/process_types.go` |
| **Basic reconciler** | 7-step reconcile skeleton | ✅ `internal/controller/process_controller.go` |
| **Finalizer handling** | Add/remove finalizer | ✅ Implemented |
| **Dependency validation** | Check referenced Domains/Agents exist and are Ready | ⏳ Pending (Phase 2) |
| **Worker Deployment** | Create Deployment for Temporal workers | ✅ `internal/controller/reconcile_workload.go` |
| **Status conditions** | Set `Ready`, `WorkersReady` | ✅ `internal/controller/conditions.go` |

**Helm chart**: Operators deployable via unified chart at `operators/helm/`

### Phase 2: (intentionally no design‑time integration)

Design‑time artifacts such as BPMN diagrams and ProcessDefinitions live entirely in the Transformation Domain and are consumed by developer/agent workflows and the Process primitive’s own code. The Process operator **does not**:

- fetch ProcessDefinitions,
- create ConfigMaps with compiled BPMN/definitions, or
- compute definition checksums/versions.

Any references to ProcessDefinitions in the `Process` spec are treated as **opaque metadata only** (for example, passed through as environment variables) and are not used by the operator to call back into Transformation. All design‑time/definition integration is the responsibility of CLI tooling and Process primitives, not the operator.

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
| **Safe Temporal deployments** | Adopt a safe rollout strategy for workflow code (Temporal Worker Versioning, patching, or task‑queue‑per‑version) so mixed worker versions never break determinism or task handling on a Task Queue | Operator configuration and deployment docs describe how Process workers use Worker Versioning or equivalent; rollouts do not introduce “workflow type not found” or non‑deterministic replay errors |
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

### 6.1 Reconciler Struct

```go
type ProcessReconciler struct {
    client.Client
    Scheme              *runtime.Scheme
    Recorder            record.EventRecorder
    TemporalAdminClient temporaladmin.Client // optional
}
```

### 6.2 Worker Deployment Template

```go
func mutateWorkerDeployment(deploy *appsv1.Deployment, process *amv1.Process, configMapName string) {
    // NOTE: avoid overwriting whole PodSpec (defaulted fields will cause endless updates).
    deploy.Spec.Template.Spec.Volumes = []corev1.Volume{{
        Name: "definition",
        VolumeSource: corev1.VolumeSource{
            ConfigMap: &corev1.ConfigMapVolumeSource{
                LocalObjectReference: corev1.LocalObjectReference{Name: configMapName},
            },
        },
    }}

    idx := -1
    for i := range deploy.Spec.Template.Spec.Containers {
        if deploy.Spec.Template.Spec.Containers[i].Name == "worker" {
            idx = i
            break
        }
    }
    if idx == -1 {
        deploy.Spec.Template.Spec.Containers = append(deploy.Spec.Template.Spec.Containers, corev1.Container{Name: "worker"})
        idx = len(deploy.Spec.Template.Spec.Containers) - 1
    }
    c := &deploy.Spec.Template.Spec.Containers[idx]
    c.Image = process.Spec.Image
    c.Env = []corev1.EnvVar{
        {Name: "TEMPORAL_NAMESPACE", Value: process.Spec.Temporal.Namespace},
        {Name: "TASK_QUEUE", Value: process.Spec.Temporal.TaskQueue},
        {Name: "PROCESS_DEFINITION_ID", Value: process.Spec.DefinitionRef.ID},
    }
    c.VolumeMounts = []corev1.VolumeMount{{
        Name:      "definition",
        MountPath: "/etc/process-definition",
    }}
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
| **Transformation Domain** | Source of design-time ProcessDefinitions (used by CLI/agents and Process primitives, not by the operator) |
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
| [446-namespace-isolation.md](446-namespace-isolation.md) | Operator deploys once per cluster |
| [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Cluster-scoped deployment via ApplicationSet |
| [503-operators-helm-chart.md](503-operators-helm-chart.md) | Helm chart for operator deployment |
| [495-ameide-operators.md](495-ameide-operators.md) | Process operator responsibilities (§2) |
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Go patterns & adaptation (§10.9) |
| [477-primitive-stack.md](477-primitive-stack.md) | Process in primitive architecture |
| [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | ProcessDefinition as design artifact |
