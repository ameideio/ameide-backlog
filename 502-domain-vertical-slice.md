# 502 – Domain Vertical Slice: End-to-End Implementation

**Status:** Active (operator + CLI foundation landed; GitOps + guardrails still in progress)
**Audience:** Platform engineers, AI agents implementing Domain
**Scope:** Complete vertical slice: Operator + CLI + Proto + GitOps

**Authority & supersession**

- This backlog is **authoritative for the vertical-slice pattern and shared condition vocabulary** used by primitive operators and the `ameide primitive` CLI.  
- The **Domain operator** (`498-domain-operator.md`) is expected to follow this slice exactly; Process (`499`) and Agent (`500`/`504`) slices must align their status/condition modeling with the definitions here.  
- Any older backlog that invents bespoke condition names or Ready/Degraded semantics for a primitive is superseded by the shared vocabulary in §1.3 of this file.

**Contract surfaces (shared across primitives)**

- Condition types (`Ready`, `WorkloadReady`, `DBReady`, `MigrationSucceeded`, `MigrationFailed`, `Degraded`) and reasons defined in `operators/shared/api/v1/conditions.go`.  
- CLI expectations for `describe`/`verify`/`drift`/`plan`/`impact` output as defined in the 484a–484f CLI backlogs.  
- Operator CRD layout, spec/status patterns, and namespace/helm deployment patterns as defined in `495-ameide-operators.md` and `503-operators-helm-chart.md`.

## Grounding & cross-references

- **Architecture grounding:** Encodes the vertical-slice pattern implied by `470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, `476-ameide-security-trust.md`, `477-primitive-stack.md`, and the integration rules in `496-eda-principles-v6.md` for Domain primitives.  
- **Operator & CLI relationships:** Serves as the reference implementation that `498-domain-operator.md`, `495-ameide-operators.md`, `497-operator-implementation-patterns.md`, `503-operators-helm-chart.md`, and the 484a–484f CLI backlogs follow when defining CRDs, conditions, Helm packaging, and `ameide primitive` behavior for all primitives.  
- **Cross-primitive alignment:** Process (`499-process-operator.md`), Agent (`500-agent-operator.md` / `504-agent-vertical-slice.md`), and UISurface (`501-uisurface-operator.md`) operators are expected to mirror this slice’s condition vocabulary, reconcile phases, and GitOps wiring.  
- **Scrum stack usage:** The Transformation Scrum domain defined in `300-400/367-1-scrum-transformation.md` and `508-scrum-protos.md` should be implemented as a Domain primitive that follows this vertical slice; the Scrum runtime seam in `506-scrum-vertical-v2.md` and the agent stack in `505-agent-developer-v2.md` assume this pattern for status and CLI visibility.

> **Related**:
> - [498-domain-operator.md](498-domain-operator.md) – Operator development tracking
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – Go patterns
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Namespace isolation (operators cluster-scoped)
> - [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) – Cluster-scoped deployment
> - [484a-ameide-cli-primitive-workflows.md](484a-ameide-cli-primitive-workflows.md) – CLI commands & agentic workflow
> - [484b-ameide-cli-proto-contract.md](484b-ameide-cli-proto-contract.md) – Proto message definitions

---

## 0. Status snapshot (repo state)

| Track/Phase | State | Notes |
|-------------|-------|-------|
| **Operator A (CRD schema)** | ✅ | `operators/domain-operator/api/v1/*.go` + generated CRDs define the Domain spec/conditions. |
| **Operator B (Reconciler)** | ✅ | `internal/controller/domain_controller.go` implements the full seven-step reconcile pattern with finalizers. |
| **Operator C (Workload)** | ✅ | Deployment + Service reconciliation lives in `reconcile_workload.go`; resources/env/security fields are wired into the Pod template. |
| **Operator C2 (Database)** | ✅ | `reconcile_db.go` copies CNPG secrets, wires migration Jobs, and surfaces `DBReady`/`MigrationSucceeded`. |
| **Operator D (Status)** | ✅ | Conditions computed in `domain_controller.go` keep Ready aligned with Workload/DB health. |
| **Operator E (Helm chart)** | ✅ | `operators/helm/templates/domain-operator-deployment.yaml` and packaged CRDs deploy the controller cluster-wide. |
| **Operator F/G (GitOps + sample)** | ✅ | Sample Domain CR + component live under `gitops/ameide-gitops/environments/_shared/primitives/domain/orders.yaml` and `.../components/apps/primitives/orders/component.yaml`. |
| **CLI H (Proto)** | ✅ | `packages/ameide_core_proto/.../primitive_types.proto` drives Domain JSON output. |
| **CLI I (Commands)** | ✅ | `describe`/`verify` support `--repo-root/--gitops-root`, and `verify --mode all` runs naming/security/EDA/test heuristics alongside cluster checks. |
| **CLI J (K8s client)** | ✅ | `internal/k8s/client.go` exposes the Domain GVR and is consumed by describe/verify. |
| **CLI K (JSON output)** | ✅ | Domain-specific details (`image`, `dbSchema`, replicas) are populated in `primitive.go`. |
| **CLI L (Tests)** | ⚠️ Partial | There are unit tests for describe/table output, but end-to-end tests covering verify/drift flows are still TODO. |
| **End-to-End Demo (Phase M)** | ✅ | `scripts/demo-domain-slice.sh` mirrors the agent demo (apply sample CR → wait Ready → run CLI). |

> Keep this summary in sync with the agent slice table so both documents reflect real progress each sprint.

---

## 1. Vertical Slice Overview

A **vertical slice** delivers a complete, deployable feature across all layers. For Domain, this means **both** the operator infrastructure AND the CLI tooling:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Domain Vertical Slice                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  OPERATOR TRACK (Phases A-G)           CLI TRACK (Phases H-L)           │
│  ─────────────────────────────         ─────────────────────────        │
│                                                                         │
│  A. CRD Schema                         H. Proto Messages                │
│      └─ api/v1/domain_types.go             └─ primitive/v1/primitive.proto
│         ↓                                      ↓                        │
│  B. Reconciler                         I. CLI Commands                  │
│      └─ domain_controller.go               └─ describe, verify, drift  │
│         ↓                                      ↓                        │
│  C. Workload                           J. K8s Client                    │
│      └─ Deployment + Service               └─ Read Domain CRs           │
│         ↓                                      ↓                        │
│  C2. Database                          K. JSON Output                   │
│      └─ CNPG + Migration Job               └─ Matches proto shapes      │
│         ↓                                      ↓                        │
│  D. Status                             L. Integration Test              │
│      └─ Ready, WorkloadReady,              └─ CLI → Operator → Status   │
│         DBReady, MigrationSucceeded                                     │
│         ↓                                                               │
│  E. Helm Chart                                                          │
│         ↓                                                               │
│  F. GitOps                                                              │
│         ↓                                                               │
│  G. Sample CR                                                           │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                      M. End-to-End Demo                                 │
│   Create Domain CR → Migration runs → Deployment ready → CLI reports    │
└─────────────────────────────────────────────────────────────────────────┘
```

**Deliverable**: Full loop where:
1. `kubectl apply` a Domain CR with `spec.db` configured
2. Operator runs Migration Job (connects to CNPG, applies schema)
3. Operator reconciles Deployment + Service (with DB credentials injected)
4. `ameide primitive describe --kind domain --json` shows status including DBReady
5. `ameide primitive verify --kind domain --name orders --json` validates health

---

## 1.1 Scope & Non-Scope

### What This Slice Covers

A complete vertical slice includes **all** Domain functionality:

- Domain CRD types with kubebuilder markers
- Full reconciler: Deployment + Service + Migration Job
- **CNPG integration**: Database connection, secret injection
- **Migration Job**: Run Flyway/similar migrations on deploy
- Status conditions (Ready, WorkloadReady, DBReady, MigrationSucceeded)
- Helm chart for operator deployment
- CLI `describe` and `verify` commands (read-only)
- Proto message shapes for JSON output

### What This Slice Defers

| Deferred | Reason | Future Slice |
|----------|--------|--------------|
| **HPA, NetworkPolicy, ServiceMonitor** | Production hardening | 498 Phase 3 |

> **Note**: CLI write operations (scaffold, generate) use Git/PR workflow, not direct imperative API calls. This is by design, not a deferral.

---

## 1.2 Repository Split

This vertical slice spans **two repositories**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ameide-core (core repo)                                                │
│  ────────────────────────                                               │
│  • operators/domain-operator/     ← CRD types, reconciler, Makefile     │
│  • operators/helm/                ← Helm chart for all operators        │
│  • packages/ameide_core_cli/      ← CLI commands                        │
│  • packages/ameide_core_proto/    ← Proto message definitions           │
│                                                                         │
│  What lives here: Implementation code, build artifacts, tests           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  ameide-gitops (gitops repo)                                            │
│  ────────────────────────────                                           │
│  • envs/dev/apps/platform-operators.yaml    ← ArgoCD Application        │
│  • envs/dev/values/operators/               ← Environment-specific vals │
│  • envs/dev/primitives/domain/orders.yaml   ← Sample Domain CR          │
│                                                                         │
│  What lives here: Runtime config, CRD instances, environment values     │
└─────────────────────────────────────────────────────────────────────────┘
```

**CLI defaults:**
- `--repo-root` defaults to current working directory (core repo)
- `--gitops-root` defaults to `../ameide-gitops` or env var `AMEIDE_GITOPS_ROOT`

---

## 1.3 Shared Condition Vocabulary

All primitive operators use a **consistent condition vocabulary** defined in a shared package. This ensures CLI tools can reason about status uniformly across Domain, Process, Agent, and UISurface:

| Condition Type | Meaning | Set By |
|----------------|---------|--------|
| `Ready` | Primitive is fully operational | Computed from others |
| `WorkloadReady` | Deployment has available replicas | `reconcileWorkload()` |
| `DBReady` | Database schema/connection ready | `reconcileDB()` (future) |
| `MigrationSucceeded` | Latest migration completed | Migration Job watcher |
| `MigrationFailed` | Migration Job failed | Migration Job watcher |
| `Degraded` | Running but not at full capacity | Health checks |

**Computation rule** (same for all primitives):
```
Ready = WorkloadReady && !Degraded && (DBReady if spec.db defined)
```

**Shared types location**: `operators/shared/api/v1/conditions.go`

```go
package v1

// Canonical condition types for all primitive operators
const (
    ConditionReady             = "Ready"
    ConditionWorkloadReady     = "WorkloadReady"
    ConditionDBReady           = "DBReady"
    ConditionMigrationSucceeded = "MigrationSucceeded"
    ConditionMigrationFailed   = "MigrationFailed"
    ConditionDegraded          = "Degraded"
)

// Condition reasons
const (
    ReasonReconciling     = "Reconciling"
    ReasonReady           = "Ready"
    ReasonDeploymentReady = "DeploymentReady"
    ReasonWaitingReplicas = "WaitingForReplicas"
    ReasonMigrationPending = "MigrationPending"
)
```

This vocabulary is referenced in:
- [495-ameide-operators.md](495-ameide-operators.md) §2 – CRD status shapes
- [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) §3.2 – Spec/Status separation
- [498-domain-operator.md](498-domain-operator.md) §5.3 – Condition precedence

### 1.3.1 Process + Agent alignment

- **Process operator (`499-process-operator.md`)**: Process primitives MUST expose `Ready` and `Degraded` using the same semantics, so `ameide primitive verify` can treat Domain and Process consistently when reporting workflow health. Any Temporal- or SLA-specific conditions should build on top of, not replace, this vocabulary.  
- **Agent operator (`500-agent-operator.md` / `504-agent-vertical-slice.md`)**: Agent primitives MUST surface `Ready`, `DefinitionResolved`, `SecretsReady`, `ToolingReady`, `PolicyCompliant`, and `Degraded` in a way that composes cleanly into `Ready` as described above. CLI slices for Agents reuse the same `Ready` and `Degraded` meaning as Domains/Processes, so status tables in 499/500/504 should be kept in sync with this section.

---

## 2. Implementation Sequence

Execute these phases in order. Each phase has clear acceptance criteria.

### Operator Track

| Phase | Focus | Files Created | Acceptance Criteria |
|-------|-------|---------------|---------------------|
| **A** | CRD Types | `operators/domain-operator/api/v1/*.go` | `make manifests` generates valid CRD YAML |
| **B** | Reconciler | `internal/controller/*.go` | Controller starts, logs reconcile events |
| **C** | Workload | `reconcile_workload.go` | Deployment + Service created from CR |
| **C2** | Database | `reconcile_db.go` | Migration Job created, CNPG secrets injected |
| **D** | Status | `conditions.go` | `status.conditions` includes DBReady, MigrationSucceeded |
| **E** | Helm | `charts/ameide-operators/` | `helm template` renders valid manifests |
| **F** | GitOps | gitops repo changes | ArgoCD syncs operator to dev cluster |
| **G** | Sample CR | gitops `primitives/domain/` | Full reconcile cycle with DB works in dev |

### CLI Track

| Phase | Focus | Files Created | Acceptance Criteria |
|-------|-------|---------------|---------------------|
| **H** | Proto | `ameide_core_proto/primitive/v1/*.proto` | `buf generate` creates Go/TS types |
| **I** | CLI Commands | `packages/ameide_core_cli/cmd/primitive/*.go` | `ameide primitive describe/verify/drift/plan/impact --help` works |
| **J** | K8s Client | `internal/k8s/domain_client.go` | Can list/get Domain CRs |
| **K** | JSON Output | Matches proto shapes | Output parseable by agents |
| **L** | CLI Tests | `*_test.go` | Unit + integration tests pass |

### Integration

| Phase | Focus | Acceptance Criteria |
|-------|-------|---------------------|
| **M** | E2E Demo | Create CR → Migration runs → Workload ready → CLI reports Ready |

---

## 3. Phase A: CRD Types ✅ IMPLEMENTED

> **Implementation**: See [operators/domain-operator/api/v1/](../operators/domain-operator/api/v1/)

### 3.1 Directory Structure

All primitive operators follow this structure (Domain/Process/Agent/UISurface today; Projection/Integration follow the current posture in `backlog/520-primitives-stack-v6.md`):

```
operators/{primitive}-operator/
├── api/v1/                    # CRD types
│   ├── {primitive}_types.go   # Spec, Status, kubebuilder markers
│   ├── groupversion_info.go   # API registration
│   └── zz_generated.deepcopy.go
├── internal/controller/       # Reconciler logic
├── cmd/main.go               # Entrypoint
├── config/crd/bases/         # Generated CRD YAML
├── go.mod
├── Makefile
├── Dockerfile.release
└── Dockerfile.dev
```

### 3.2 Acceptance Criteria (Phase A) ✅

- [x] `go mod tidy` succeeds
- [x] `make generate` creates `zz_generated.deepcopy.go`
- [x] `make manifests` creates `config/crd/bases/ameide.io_domains.yaml`
- [x] CRD YAML validates with `kubectl apply --dry-run=server`

---

## 4. Phase B: Reconciler Skeleton ✅ IMPLEMENTED

> **Implementation**: See [operators/domain-operator/cmd/main.go](../operators/domain-operator/cmd/main.go) and [operators/domain-operator/internal/controller/](../operators/domain-operator/internal/controller/)

### 4.1 Key Components

- **Entry point**: `cmd/main.go` - controller-runtime manager with health/ready probes
- **Reconciler**: `internal/controller/domain_controller.go` - 7-step reconcile pattern
- **Manager setup**: Uses `metricsserver.Options` for metrics binding

### 4.2 Acceptance Criteria (Phase B) ✅

- [x] `go build ./cmd/...` succeeds
- [x] `make run` starts controller, connects to cluster
- [x] Creating a Domain CR triggers reconcile log
- [x] Deleting a Domain CR triggers finalizer cleanup log

---

## 5. Phase C: Workload Reconciliation ✅ IMPLEMENTED

> **Implementation**: See [operators/domain-operator/internal/controller/reconcile_workload.go](../operators/domain-operator/internal/controller/reconcile_workload.go)

### 5.1 Key Features

- **CreateOrUpdate pattern**: Uses `controllerutil.CreateOrUpdate` for idempotent reconciliation
- **OwnerReferences**: Automatic garbage collection when Domain CR is deleted
- **Labels**: Consistent labeling with `ameide.io/primitive` and `ameide.io/domain`
- **Workload naming**: Deployment and Service resources follow the `{domain}-domain` suffix that `verify`/demo scripts must reference

### 5.2 Acceptance Criteria (Phase C) ✅

- [x] Creating Domain CR creates Deployment
- [x] Creating Domain CR creates Service
- [x] Deployment has correct labels, image, env
- [x] Service selector matches Deployment labels
- [x] Deleting Domain CR deletes owned resources (GC)

---

## 5.5 Phase C2: Database Integration ✅ IMPLEMENTED

> **Implementation**: See [operators/domain-operator/internal/controller/reconcile_db.go](../operators/domain-operator/internal/controller/reconcile_db.go)

This phase implements CNPG database integration for Domain primitives.

### 5.5.1 Reconcile Flow

```
1. Check if spec.db is defined
2. Lookup CNPG Cluster secret from spec.db.clusterRef
3. Create/Update Migration Job
   - Image: spec.db.migrationJobImage
   - Env: PGHOST, PGUSER, PGPASSWORD, PGDATABASE from CNPG secret
   - Command: run Flyway or similar migrations
4. Watch Job completion
5. Set DBReady condition based on Job status
6. Set MigrationSucceeded or MigrationFailed condition
7. Inject DB credentials into Domain Deployment
```

### 5.5.2 Key Implementation Details

**CNPG Secret Lookup:**
```go
// spec.db.clusterRef format: "namespace/cluster-name" or just "cluster-name" (same namespace)
secretName := fmt.Sprintf("%s-app", clusterName) // CNPG naming convention
```

**Migration Job Template:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {domain}-migration
  ownerReferences:
    - apiVersion: ameide.io/v1
      kind: Domain
      name: {domain}
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: {spec.db.migrationJobImage}
          env:
            - name: PGHOST
              valueFrom:
                secretKeyRef:
                  name: {cnpg-cluster}-app
                  key: host
            - name: PGDATABASE
              valueFrom:
                secretKeyRef:
                  name: {cnpg-cluster}-app
                  key: dbname
            # ... PGUSER, PGPASSWORD similarly
      restartPolicy: Never
  backoffLimit: 3
```

### 5.5.3 Acceptance Criteria (Phase C2) ✅

- [x] Migration Job created when `spec.db` is defined
- [x] Job uses correct CNPG secret for credentials
- [x] Job runs `migrationJobImage` with proper env vars
- [x] `DBReady` condition set to True when Job succeeds
- [x] `MigrationSucceeded` condition shows migration status
- [x] `MigrationFailed` condition set if Job fails
- [x] Domain Deployment has DB credentials injected
- [ ] Job re-runs when `spec.db.migrationJobImage` changes (deferred - requires Job diff logic)

---

## 6. Phase D: Status & Conditions ✅ IMPLEMENTED

> **Implementation**: See [operators/domain-operator/internal/controller/conditions.go](../operators/domain-operator/internal/controller/conditions.go)

### 6.1 Condition Vocabulary

Uses shared vocabulary from `operators/shared/api/v1/conditions.go`:

| Condition | Meaning | Status |
|-----------|---------|--------|
| `Ready` | Computed: WorkloadReady && DBReady && !Degraded | ✅ Implemented |
| `WorkloadReady` | Deployment has available replicas | ✅ Implemented |
| `DBReady` | Database schema ready, migration completed | ✅ Implemented |
| `MigrationSucceeded` | Latest migration Job completed successfully | ✅ Implemented |
| `MigrationFailed` | Migration Job failed | ✅ Implemented |
| `Degraded` | Running but with errors | ✅ Implemented |

### 6.2 Acceptance Criteria (Phase D) ✅

- [x] `kubectl get domain` shows Ready column
- [x] `status.conditions` includes Ready, WorkloadReady
- [x] `status.readyReplicas` reflects Deployment status
- [x] `status.observedGeneration` matches `metadata.generation`
- [x] `status.conditions` includes DBReady, MigrationSucceeded/Failed
- [x] Ready computation includes DBReady when spec.db is defined

---

## 7. Phase E: Helm Chart ✅ IMPLEMENTED

> **Implementation**: See [operators/helm/](../operators/helm/)

The unified Helm chart deploys the currently implemented primitive operators (Domain/Process/Agent/UISurface) from a single installation.

### 7.1 Chart Structure

```
operators/helm/
├── Chart.yaml              # Chart metadata (v0.1.0)
├── values.yaml             # Default values for all operators
├── crds/                   # CRDs (auto-installed by Helm)
│   ├── ameide.io_domains.yaml
│   ├── ameide.io_processes.yaml
│   ├── ameide.io_agents.yaml
│   └── ameide.io_uisurfaces.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── serviceaccount.yaml
│   ├── clusterrole.yaml
│   ├── clusterrolebinding.yaml
│   ├── domain-operator-deployment.yaml
│   ├── process-operator-deployment.yaml
│   ├── agent-operator-deployment.yaml
│   ├── uisurface-operator-deployment.yaml
│   └── NOTES.txt
└── examples/               # Sample CRs for testing
    ├── domain-sample.yaml
    ├── process-sample.yaml
    ├── agent-sample.yaml
    └── uisurface-sample.yaml
```

### 7.2 Key Features

- **Single chart for all operators**: Each operator can be enabled/disabled independently
- **Shared RBAC**: One ClusterRole covers the currently implemented primitive CRDs (Domain/Process/Agent/UISurface)
- **CRDs in /crds folder**: Helm installs CRDs before templates
- **Security by default**: Non-root, drop ALL capabilities
- **Health probes**: Liveness/readiness on all operators

### 7.3 Usage

```bash
# Install all operators
helm install ameide ./operators/helm -n ameide-system --create-namespace

# Install only domain operator
helm install ameide ./operators/helm \
  --set operators.process.enabled=false \
  --set operators.agent.enabled=false \
  --set operators.uisurface.enabled=false

# Test with sample CRs
kubectl apply -f operators/helm/examples/
```

### 7.4 Acceptance Criteria (Phase E) ✅

- [x] `helm lint` passes
- [x] `helm template` renders valid manifests
- [x] All four operator Deployments rendered
- [x] ClusterRole covers all AMEIDE CRDs
- [x] CRDs copied from operator config/crd/bases

---

## 7.5 Phase F: GitOps Integration ✅ IMPLEMENTED

> **Implementation**: See [ameide-gitops/environments/_shared/components/cluster/](https://github.com/ameideio/ameide-gitops)

The GitOps integration deploys operators using the cluster-scoped ApplicationSet pattern:

### 7.5.1 Components Created

| Component | Path | rolloutPhase |
|-----------|------|--------------|
| **CRDs** | `environments/_shared/components/cluster/crds/ameide/` | 010 |
| **Operators** | `environments/_shared/components/cluster/operators/ameide-operators/` | 020 |

### 7.5.2 Helm Chart Location

Chart copied to `sources/charts/platform/ameide-operators/` with:
- Domain, Process, Agent, UISurface operator Deployments
- Shared ClusterRole and ServiceAccount
- `crdsOnly` mode for CRD/operator separation
- Security contexts, resource limits

### 7.5.3 Values Files

| File | Purpose |
|------|---------|
| `sources/values/_shared/cluster/crds-ameide.yaml` | `crdsOnly: true` for CRD installation |
| `sources/values/_shared/cluster/ameide-operators.yaml` | Full operator configuration |

### 7.5.4 Acceptance Criteria (Phase F) ✅

- [x] CRD component created at rolloutPhase 010
- [x] Operator component created at rolloutPhase 020
- [x] Helm chart copied to ameide-gitops
- [x] Values files created for both modes
- [x] Components follow existing Strimzi/CNPG pattern

---

## 7.6 Phase G: Sample CR ✅ IMPLEMENTED (Helm + GitOps)

> **Implementation**: See [operators/helm/examples/](../operators/helm/examples/)

Sample CRs are included in the Helm chart at `examples/`:

| File | Description |
|------|-------------|
| `domain-sample.yaml` | Domain without database dependencies |
| `domain-with-db-sample.yaml` | Exercises CNPG + migration job flow |
| `process-sample.yaml` | Process worker with Temporal config |
| `agent-sample.yaml` | Agent primitive with tool grants |
| `uisurface-sample.yaml` | UISurface pointing at a placeholder image |

> **GitOps status**: `environments/_shared/primitives/domain/orders.yaml` plus the `domain-orders` component apply the same sample (including a lightweight CNPG cluster) via the ApplicationSet matrix, so dev/staging/prod pick up a Domain primitive automatically.

### Acceptance Criteria (Phase G)

- [x] Sample CRs exist under `operators/helm/examples/`
- [x] `kubectl apply -f operators/helm/examples/domain-sample.yaml` reconciles without DB
- [x] `kubectl apply -f operators/helm/examples/domain-with-db-sample.yaml` reconciles with CNPG + migrations
- [x] Domain sample synced via GitOps ApplicationSet (`environments/_shared/primitives/domain/orders.yaml`)

---

## 8. CLI Track: Agentic Guardrail System

> **Implementation**: See [packages/ameide_core_cli/internal/commands/primitive.go](../packages/ameide_core_cli/internal/commands/primitive.go)
> **Full Specification**: See [484a-ameide-cli-primitive-workflows.md](484a-ameide-cli-primitive-workflows.md)

The CLI is designed as a **guardrail system for AI agents** following the "Shape vs Meaning" principle:
- **CLI owns Shape**: Directory layout, proto/SDK freshness, convention enforcement
- **Agent owns Meaning**: Business logic, test assertions, implementation details

### 8.0 Design Philosophy

```
OBSERVE → REASON → DRY-RUN → HUMAN GATE → ACT → VERIFY
```

CLI commands support this workflow:
- **OBSERVE**: `describe`, `drift` - understand current state
- **REASON**: `plan`, `impact` - research what's needed
- **ACT**: `scaffold` - generate mechanical skeletons
- **VERIFY**: `verify` - validate health and conventions

### 8.1 Phase H: Proto Messages ✅ IMPLEMENTED

Proto definitions in `packages/ameide_core_proto/src/ameide_core_proto/primitive/v1/`:

| File | Messages |
|------|----------|
| `primitive_types.proto` | `PrimitiveKind`, `Condition`, `PrimitiveInfo`, `CheckResult`, `DriftInfo`, `ImpactInfo` |
| `primitive_service.proto` | `PrimitiveService` with all RPCs |

Generated code in:
- Go: `packages/ameide_sdk_go/proto/ameide_core_proto/primitive/v1/`
- TypeScript: `packages/ameide_sdk_ts/src/_proto/ameide_core_proto/primitive/v1/`

### 8.2 Phase I: CLI Commands

| Command | Purpose | Status |
|---------|---------|--------|
| **describe** | Introspect what exists (K8s CRs + repo state) | ✅ K8s implemented, ⏳ repo diffing pending |
| **drift** | Detect SDK/proto staleness, missing tests, convention violations | ✅ Proto timestamps + README/tests heuristics |
| **plan** | Research what work is needed (tests to create, scaffolds) | ✅ Proto-driven suggestions + RPC-derived tests |
| **impact** | Analyze proto changes - cascade scope, affected consumers | ✅ Go import scan (heuristic) |
| **verify** | Run health checks with structured output | ✅ K8s checks, ⏳ naming/EDA/security pending |
| **scaffold** | Generate mechanical proto-driven skeletons (TDD-aligned) | ✅ Domain + Go (GitOps/test-harness optional) |
| **prompt** | Emit starting prompt for agents (tying CLI workflow to backlogs) | ✅ Outputs OBSERVE→VERIFY checklist referencing 484a/502 |

Repo-analysis helpers (`packages/ameide_core_cli/internal/commands/primitive_analysis.go`) back the new commands:
- **Drift** compares proto vs `packages/ameide_sdk_go` timestamps, checks for missing RPC tests, and flags missing README/tests directories.
- **Plan** parses proto RPCs to enumerate scaffolding paths/tests using the shared proto contract from 484b.
- **Impact** walks Go files for generated SDK import paths to approximate downstream consumers. (TypeScript/Python scanning remains future work.)

**Flags** (common to all commands):
```bash
--kind domain|process|agent|uisurface
--name <primitive-name>
--namespace <k8s-namespace>
--json                    # Structured output for agents
--repo-root <path>        # Core repo root (default: cwd)
--gitops-root <path>      # GitOps repo root (default: env or ../ameide-gitops)
```

### 8.3 Phase J: K8s Client ✅ IMPLEMENTED

Dynamic client implementation in `packages/ameide_core_cli/internal/k8s/client.go`:
- Uses `k8s.io/client-go/dynamic` for CRD operations
- GVR constants for primitive types (Domain/Process/Agent/UISurface today; Projection/Integration planned)
- Kubeconfig auto-detection (env var, default path, in-cluster)

### 8.4 Phase K: Verification Checks

| Check | What It Validates | Status |
|-------|-------------------|--------|
| **WorkloadReady** | Deployment has available replicas | ✅ Implemented |
| **ConditionsHealthy** | Ready=True, Degraded=False | ✅ Implemented |
| **DBReady** | Database connectivity (Domain only) | ✅ Implemented |
| **MigrationStatus** | Migration job succeeded/failed (Domain only) | ✅ Implemented |
| **naming** | RPC naming conventions (business verbs, not CRUD) | ❌ NOT IMPLEMENTED |
| **eda** | Outbox wiring, event emission, idempotency | ❌ NOT IMPLEMENTED |
| **security** | Secret scan, vuln scan, SAST hooks | ❌ NOT IMPLEMENTED |
| **schema** | Buf compatibility, event metadata | ❌ NOT IMPLEMENTED |
| **tests** | Unit/integration test execution | ❌ NOT IMPLEMENTED |

### 8.5 Phase L: Scaffold Command ✅ IMPLEMENTED (Domain + Go)

Implementation: `packages/ameide_core_cli/internal/commands/primitive_scaffold.go`

Highlights:
- **Proto-driven**: Parses RPC signatures from the provided proto (via `parseRPCDefinitions`) and emits handler stubs + failing tests per RPC. Output JSON matches the new `ScaffoldRequest/ScaffoldResponse` messages in `primitive_service.proto`.
- **One-shot**: Refuses to scaffold if `primitives/{kind}/{name}` already exists (dry-run still validates conflicts). Supports `--dry-run` to list files without writing.
- **Shape only**:
  - `cmd/main.go` placeholder
  - `internal/handlers/handlers.go` with `status.Error(codes.Unimplemented, "...")`
  - `internal/tests/*_test.go` that immediately `t.Fatalf`, guaranteeing RED state
  - `README.md`, `go.mod` (with local replace to `packages/ameide_sdk_go`)
  - GitOps stubs and tests scaffolded by default (no optional include flags)
  - No per-component `tests/run_integration_tests.sh` harness as a required contract (legacy only; 430v2 uses native tooling via `ameide test`)
- **Scope**: Domain primitives in Go (initial slice). Other kinds/langs still TODO.

Usage example:
```bash
ameide primitive scaffold \
  --kind domain \
  --name orders \
  --proto-path packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation_service.proto \
  --lang go \
  --json
```

### 8.6 Acceptance Criteria (CLI Track)

**Implemented**:
- [x] `buf lint` passes on primitive protos
- [x] `buf generate` creates Go/TS types
- [x] `ameide primitive describe` lists K8s CRs with status
- [x] `ameide primitive verify` runs K8s health checks (WorkloadReady, ConditionsHealthy, DBReady, MigrationStatus)
- [x] `ameide primitive drift` now reports proto/SDK timestamp drift + README/tests conventions
- [x] `ameide primitive plan` suggests scaffolding/test work items from proto RPCs
- [x] `ameide primitive impact` scans Go imports for downstream consumers
- [x] Repo-analysis helpers covered by unit tests (`primitive_analysis_test.go`)
- [x] JSON output matches proto shapes (Plan/Drift/Impact results)
- [x] Unit tests pass (`go test ./packages/ameide_core_cli/...`)
- [x] `ameide primitive scaffold` (Domain/Go) generates handlers, tests, README, go.mod, optional GitOps/test harness

**Pending** (per 484a-e full scope):
- [ ] `describe` includes repo state (expected vs actual primitives)
- [ ] `drift` understands non-Go consumers + richer convention checks
- [ ] `plan` folds in GitOps/statefulness suggestions
- [ ] `impact` understands TS/Python imports and proto usage graphs
- [ ] `scaffold` generates TDD-aligned skeletons
- [ ] `verify --check naming` validates RPC naming conventions
- [ ] `verify --check eda` validates outbox/event wiring
- [ ] `verify --check security` runs secret/vuln scans
- [ ] `ameide test` executes unit/integration tests (Phase 0/1/2 per 430v2; in-repo only)

---

## 9. Phase M: End-to-End Demo (Pending)

The E2E demo validates the full vertical slice by creating a Domain CR and verifying the operator reconciles it correctly. A runnable helper script now lives at [`scripts/demo-domain-slice.sh`](../scripts/demo-domain-slice.sh); pass `--with-db` to exercise the CNPG + migration path (requires a CNPG operator) and remember that Deployments/Services are suffixed `-domain`.

### Demo Flow

```
1. kubectl apply Domain CR
2. Operator reconciles → Deployment + Service created
3. ameide primitive describe → JSON shows status
4. ameide primitive verify → Returns "pass"
5. kubectl delete Domain → Resources cleaned up
```

### Acceptance Criteria (Phase M)

- [x] Demo script implemented at `scripts/demo-domain-slice.sh` (DB path optional via `--with-db`)
- [ ] Domain CR creates Deployment + Service
- [ ] CLI describe shows correct status
- [ ] CLI verify returns "pass" when healthy
- [ ] CLI verify returns "fail" with issues when unhealthy
- [ ] Cleanup removes all resources

---

## 10. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [446-namespace-isolation.md](446-namespace-isolation.md) | Namespace isolation – operators deploy once per cluster |
| [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Cluster-scoped deployment via ApplicationSet |
| [503-operators-helm-chart.md](503-operators-helm-chart.md) | Helm chart for operator deployment (Phase E) |
| [498-domain-operator.md](498-domain-operator.md) | Operator development phases & acceptance criteria |
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Go patterns & reference implementation |
| [495-ameide-operators.md](495-ameide-operators.md) | CRD shapes & responsibilities |
| [484a-ameide-cli-primitive-workflows.md](484a-ameide-cli-primitive-workflows.md) | CLI commands, TDD loop, agentic workflow |
| [484b-ameide-cli-proto-contract.md](484b-ameide-cli-proto-contract.md) | Proto message definitions for CLI |
| [484d-ameide-cli-migration.md](484d-ameide-cli-migration.md) | CLI phased implementation plan |

---

## 11. Known Gaps & Issues

This section tracks implementation gaps discovered during review. These must be addressed before declaring the vertical slice complete.

### 11.1 GitOps Sample Is Not Environment-Scoped (historically: `primitives-dev`)

**Location**:
- [`environments/_shared/primitives/domain/orders.yaml`](../gitops/ameide-gitops/environments/_shared/primitives/domain/orders.yaml)
- [`environments/_shared/components/apps/primitives/orders/component.yaml`](../gitops/ameide-gitops/environments/_shared/components/apps/primitives/orders/component.yaml)

**Historical context**: Early drafts deployed the sample Domain to a dedicated `primitives-dev` namespace.

**Update (2026-01)**: The GitOps sample no longer assumes a `primitives-dev` namespace; it renders into the **release namespace** (and the standalone sample manifest omits `metadata.namespace` so `kubectl apply -n <ns>` controls placement).

**Issue**: The GitOps-managed Domain sample (and its bundled CNPG Cluster) is still treated as a shared, always-on component. That’s ideal for dev, but staging/production need different namespaces, storage classes, and potentially **no sample workload at all**.

**Impact**: Higher environments risk carrying the dev sample (and test CNPG cluster) unless operators override the component manually.

**Fix Required**: Guard the sample behind an env allow list (dev-only), or provide environment-specific overlays for namespace/storage/policy so higher environments do not inherit the dev sample.

### 11.2 Verify Heuristics vs Toolchain

**Location**:
- [484a lines 70-80](484a-ameide-cli-primitive-workflows.md) - Full check scope
- [`primitive.go`](../packages/ameide_core_cli/internal/commands/primitive.go) – repo check implementations

**Issue**: `ameide primitive verify --mode all` now runs naming/security/EDA/test heuristics, but they’re still lightweight string scans and `go test` invocations. The documentation calls for the full Buf/gitleaks/govuln/Semgrep stack.

**Impact**: Agents get better feedback than before, but the guardrail isn’t authoritative—CI may still catch additional problems.

**Fix Required**: Swap the heuristics for the real toolchain and surface those results in the JSON payload.

### 11.3 CLI Root Commands Not Fully Migrated

**Location**:
- [484d §1-2](484d-ameide-cli-migration.md) – Legacy command removal plan
- [`cmd/ameide/main.go`](../packages/ameide_core_cli/cmd/ameide/main.go) – still registers `dev`, `generate`, `model`, `command`, `query`

**Issue**: The CLI still exposes legacy commands that 484d marked for removal, and `primitive describe` continues to assume the current working directory even though other commands have repo-root flags.

**Impact**: Agents see conflicting entry points and must change directories to run describe/drift consistently.

**Fix Required**: Retire the deprecated commands and plumb `--repo-root/--gitops-root` through describe as well.

### 11.4 Phase L Verification Coverage Still Thin

**Location**:
- [Section 2](#2-implementation-sequence) – Phase L expectations
- [`primitive_test.go`](../packages/ameide_core_cli/internal/commands/primitive_test.go) – unit coverage only

**Issue**: We now have a demo script for Phase M, but no automated integration test that exercises CLI ↔︎ operator ↔︎ cluster. Phase L remains at unit-test-only coverage.

**Impact**: Regressions in JSON output or K8s interactions go unnoticed until a human runs the demo script.

**Fix Required**: Introduce integration tests (kind/minikube or envtest) that create a stub Domain CR and assert the CLI’s JSON output matches the proto schema, especially for verify/drift/plan.

### 11.5 Summary Table

| Gap | Severity | Blocking E2E? | Fix Complexity |
|-----|----------|---------------|----------------|
| GitOps sample hard-codes dev namespace | Medium | No (can disable component) | Low |
| Verify heuristics vs real toolchain | Medium | No | Medium |
| CLI migration/describe-root gaps | Medium | No | Medium |
| Phase L integration tests missing | Medium | No | Medium |
