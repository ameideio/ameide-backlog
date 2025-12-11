# 502 – Domain Vertical Slice: End-to-End Implementation

**Status:** Active
**Audience:** Platform engineers, AI agents implementing Domain
**Scope:** Complete vertical slice: Operator + CLI + Proto + GitOps

> **Related**:
> - [498-domain-operator.md](498-domain-operator.md) – Operator development tracking
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – Go patterns
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Namespace isolation (operators cluster-scoped)
> - [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) – Cluster-scoped deployment
> - [484a-ameide-cli-primitive-workflows.md](484a-ameide-cli-primitive-workflows.md) – CLI commands & agentic workflow
> - [484b-ameide-cli-proto-contract.md](484b-ameide-cli-proto-contract.md) – Proto message definitions

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
| **I** | CLI Commands | `packages/ameide_core_cli/cmd/primitive/*.go` | `ameide primitive describe --help` works |
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

All four primitive operators follow this structure:

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
- **Labels**: Consistent labeling with `ameide.io/primitive-kind` and `ameide.io/primitive-name`

### 5.2 Acceptance Criteria (Phase C) ✅

- [x] Creating Domain CR creates Deployment
- [x] Creating Domain CR creates Service
- [x] Deployment has correct labels, image, env
- [x] Service selector matches Deployment labels
- [x] Deleting Domain CR deletes owned resources (GC)

---

## 5.5 Phase C2: Database Integration (Pending)

> **Implementation**: `operators/domain-operator/internal/controller/reconcile_db.go` (to be created)

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

### 5.5.3 Acceptance Criteria (Phase C2)

- [ ] Migration Job created when `spec.db` is defined
- [ ] Job uses correct CNPG secret for credentials
- [ ] Job runs `migrationJobImage` with proper env vars
- [ ] `DBReady` condition set to True when Job succeeds
- [ ] `MigrationSucceeded` condition shows migration version
- [ ] `MigrationFailed` condition set if Job fails
- [ ] Domain Deployment has DB credentials injected
- [ ] Job re-runs when `spec.db.migrationJobImage` changes

---

## 6. Phase D: Status & Conditions ✅ IMPLEMENTED (Partial)

> **Implementation**: See [operators/domain-operator/internal/controller/conditions.go](../operators/domain-operator/internal/controller/conditions.go)

### 6.1 Condition Vocabulary

Uses shared vocabulary from `operators/shared/api/v1/conditions.go`:

| Condition | Meaning | Status |
|-----------|---------|--------|
| `Ready` | Computed: WorkloadReady && DBReady && !Degraded | ✅ Implemented |
| `WorkloadReady` | Deployment has available replicas | ✅ Implemented |
| `DBReady` | Database schema ready, migration completed | ⏳ Pending (Phase C2) |
| `MigrationSucceeded` | Latest migration Job completed successfully | ⏳ Pending (Phase C2) |
| `MigrationFailed` | Migration Job failed | ⏳ Pending (Phase C2) |
| `Degraded` | Running but with errors | ✅ Implemented |

### 6.2 Acceptance Criteria (Phase D)

**Implemented:**
- [x] `kubectl get domain` shows Ready column
- [x] `status.conditions` includes Ready, WorkloadReady
- [x] `status.readyReplicas` reflects Deployment status
- [x] `status.observedGeneration` matches `metadata.generation`

**Pending (after Phase C2):**
- [ ] `status.conditions` includes DBReady, MigrationSucceeded
- [ ] `status.dbMigrationVersion` shows current migration version
- [ ] Ready computation includes DBReady when spec.db is defined

---

## 7. Phase E: Helm Chart ✅ IMPLEMENTED

> **Implementation**: See [operators/helm/](../operators/helm/)

The unified Helm chart deploys all four primitive operators from a single installation.

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
- **Shared RBAC**: One ClusterRole covers all four primitive CRDs
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

## 7.6 Phase G: Sample CR ✅ IMPLEMENTED

> **Implementation**: See [operators/helm/examples/](../operators/helm/examples/) (also in gitops chart)

Sample CRs are included in the Helm chart at `examples/`:

| File | Description |
|------|-------------|
| `domain-sample.yaml` | Sample Domain CR with gRPC service |
| `process-sample.yaml` | Sample Process CR for Temporal worker |
| `agent-sample.yaml` | Sample Agent CR with LLM config |
| `uisurface-sample.yaml` | Sample UISurface CR for Next.js app |

### Acceptance Criteria (Phase G) ✅

- [x] Sample CRs exist in Helm chart examples/
- [x] `kubectl apply -f examples/domain-sample.yaml` creates CR
- [x] Operator reconciles sample CR successfully

---

## 8. Phase H-L: CLI Track ✅ IMPLEMENTED

> **Implementation**: See [packages/ameide_core_cli/internal/commands/primitive.go](../packages/ameide_core_cli/internal/commands/primitive.go)

The CLI track implements `ameide primitive` commands that interact with operator CRDs.

### 8.1 Proto Messages ✅

Proto definitions in `packages/ameide_core_proto/src/ameide_core_proto/primitive/v1/`:

| File | Messages |
|------|----------|
| `primitive_types.proto` | `PrimitiveKind` enum, `Condition`, `PrimitiveInfo`, `CheckResult`, kind-specific details |
| `primitive_service.proto` | `PrimitiveService` with `Describe` and `Verify` RPCs |

Generated code in:
- Go: `packages/ameide_sdk_go/gen/ameide_core_proto/primitive/v1/`
- TypeScript: `packages/ameide_core_proto/gen/ts/ameide_core_proto/primitive/v1/`

### 8.2 CLI Commands ✅

| Command | Description | Key Flags |
|---------|-------------|-----------|
| `ameide primitive describe` | List primitives and status | `--kind`, `--name`, `--namespace`, `--selector`, `--json` |
| `ameide primitive verify` | Health checks on primitives | `--kind`, `--name`, `--namespace`, `--check`, `--json` |

### 8.3 K8s Client ✅

Dynamic client implementation in `packages/ameide_core_cli/internal/k8s/client.go`:
- Uses `k8s.io/client-go/dynamic` for CRD operations
- GVR constants for all four primitive types (Domain, Process, Agent, UISurface)
- Kubeconfig auto-detection (env var, default path, in-cluster)

### 8.4 Check Implementations ✅

| Check | Description |
|-------|-------------|
| `WorkloadReady` | Verifies Deployment has available replicas |
| `ConditionsHealthy` | Validates Ready condition is True, Degraded is False |

### 8.5 Acceptance Criteria (Phases H-L) ✅

- [x] `buf lint` passes on primitive protos
- [x] `buf generate` creates Go/TS types
- [x] `ameide primitive --help` shows subcommands
- [x] `ameide primitive describe --help` shows flags
- [x] `ameide primitive verify --help` shows flags
- [x] K8s client can list Domain CRs via dynamic client
- [x] JSON output matches proto shapes
- [x] Unit tests pass (`go test ./packages/ameide_core_cli/...`)

---

## 9. Phase M: End-to-End Demo (Pending)

The E2E demo validates the full vertical slice by creating a Domain CR and verifying the operator reconciles it correctly.

### Demo Flow

```
1. kubectl apply Domain CR
2. Operator reconciles → Deployment + Service created
3. ameide primitive describe → JSON shows status
4. ameide primitive verify → Returns "pass"
5. kubectl delete Domain → Resources cleaned up
```

### Acceptance Criteria (Phase M)

- [ ] Demo script runs without errors
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
