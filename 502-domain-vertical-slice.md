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
- Go: `packages/ameide_sdk_go/gen/ameide_core_proto/primitive/v1/`
- TypeScript: `packages/ameide_core_proto/gen/ts/ameide_core_proto/primitive/v1/`

### 8.2 Phase I: CLI Commands

| Command | Purpose | Status |
|---------|---------|--------|
| **describe** | Introspect what exists (K8s CRs + repo state) | ✅ K8s implemented, ⏳ Repo pending |
| **drift** | Detect SDK/proto staleness, missing tests, convention violations | ❌ NOT IMPLEMENTED |
| **plan** | Research what work is needed (tests to create, scaffolds) | ❌ NOT IMPLEMENTED |
| **impact** | Analyze proto changes - cascade scope, affected consumers | ❌ NOT IMPLEMENTED |
| **verify** | Run health checks with structured output | ✅ K8s checks, ⏳ Convention checks pending |
| **scaffold** | Generate mechanical proto-driven skeletons (TDD-aligned) | ❌ NOT IMPLEMENTED |

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
- GVR constants for all four primitive types (Domain, Process, Agent, UISurface)
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

### 8.5 Phase L: Scaffold Command

The scaffold command generates **mechanical, proto-driven skeletons** that are TDD-aligned:

**Key Properties**:
- **One-shot only**: Refuses to scaffold if folder exists (never overwrites)
- **TDD-aligned**: Scaffolded tests CALL handlers and FAIL until implemented
- **No business logic**: Handlers return `codes.Unimplemented`
- **Mechanical only**: No fuzzy reasoning, purely proto-driven

**Generates** (for Domain):
```
primitives/domain/{name}/
├── cmd/main.go                    # Entry point
├── internal/
│   ├── handlers/                  # RPC handlers (return Unimplemented)
│   ├── repository/                # Data access interfaces
│   └── tests/                     # Test skeletons (FAILING)
├── Dockerfile, go.mod, README.md
│
gitops/primitives/domain/{name}/   # If --include-gitops
├── values.yaml                    # Helm values
├── component.yaml                 # ApplicationSet discovery
└── kustomization.yaml
```

**Flags**:
```bash
ameide primitive scaffold \
  --kind domain \
  --name orders \
  --proto-path orders.proto \
  --lang go \
  --dry-run \
  --include-gitops \
  --include-test-harness \
  --json
```

**Status**: ❌ NOT IMPLEMENTED

### 8.6 Acceptance Criteria (CLI Track)

**Implemented**:
- [x] `buf lint` passes on primitive protos
- [x] `buf generate` creates Go/TS types
- [x] `ameide primitive describe` lists K8s CRs with status
- [x] `ameide primitive verify` runs K8s health checks
- [x] DBReady and MigrationStatus checks for Domain
- [x] JSON output matches proto shapes
- [x] Unit tests pass (`go test ./packages/ameide_core_cli/...`)

**Pending** (per 484a-e full scope):
- [ ] `describe` includes repo state (expected vs actual primitives)
- [ ] `drift` detects SDK staleness and missing tests
- [ ] `plan` suggests work needed for a primitive
- [ ] `impact` analyzes proto change cascades
- [ ] `scaffold` generates TDD-aligned skeletons
- [ ] `verify --check naming` validates RPC naming conventions
- [ ] `verify --check eda` validates outbox/event wiring
- [ ] `verify --check security` runs secret/vuln scans
- [ ] `verify --check tests` executes unit/integration tests

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

---

## 11. Known Gaps & Issues

This section tracks implementation gaps discovered during review. These must be addressed before declaring the vertical slice complete.

### 11.1 Cross-Namespace DB Secrets Unsupported

**Location**: [reconcile_db.go:172-360](../operators/domain-operator/internal/controller/reconcile_db.go)

**Issue**: The `spec.db.clusterRef` format supports `namespace/cluster-name`, but the implementation only looks up secrets in the Domain's own namespace. CNPG clusters in different namespaces (common pattern) won't work.

**Impact**: Domains cannot reference shared CNPG clusters deployed in a central `database` namespace.

**Fix Required**: Add cross-namespace secret lookup with appropriate RBAC.

### 11.2 status.dbMigrationVersion Never Populated

**Location**:
- [domain_types.go:86-102](../operators/domain-operator/api/v1/domain_types.go) - Status field defined
- [primitive.go:298-325](../packages/ameide_core_cli/internal/commands/primitive.go) - CLI expects it

**Issue**: `DomainStatus.DBMigrationVersion` is defined in the CRD but never set by the controller. The CLI `checkMigrationStatus()` checks for `migrationVersion` in DomainDetails but it's never populated.

**Impact**: CLI cannot report which migration version is deployed.

**Fix Required**: Controller must parse migration Job logs or use a ConfigMap to track completed version.

### 11.3 Full Loop Demo Cannot Exercise Migrations

**Location**: [domain-sample.yaml:7-25](../operators/helm/examples/domain-sample.yaml)

**Issue**: The sample Domain CR lacks `spec.db` configuration. Phase M E2E demo cannot validate the Migration Job flow because there's no sample that exercises it.

**Impact**: Phase M acceptance criteria ("Migration runs → Workload ready") cannot be verified.

**Fix Required**: Add `domain-with-db-sample.yaml` that includes:
- `spec.db.clusterRef`: Reference to CNPG cluster
- `spec.db.migrationJobImage`: Flyway/Liquibase image
- Requires CNPG cluster to exist in test environment

### 11.4 Proto Contract and CLI Scope Diverge from Docs

**Location**:
- [484b lines 103-206](484b-ameide-cli-proto-contract.md) - Specifies `DriftInfo`, `ImpactInfo`, `PlanResult` messages
- [primitive_service.proto](../packages/ameide_core_proto/src/ameide_core_proto/primitive/v1/primitive_service.proto) - Actual proto

**Issue**: The proto files implement a subset of what 484b specifies. Missing:
- `DriftInfo` message and `DetectDrift` RPC
- `ImpactInfo` message and `AnalyzeImpact` RPC
- `PlanResult` message and `GeneratePlan` RPC
- `ScaffoldResult` message and `Scaffold` RPC

**Impact**: CLI commands `drift`, `impact`, `plan`, `scaffold` have no proto contract to implement against.

**Fix Required**: Extend proto files per 484b specification.

### 11.5 Verify Only Checks Cluster Conditions, Not Repo/Tests

**Location**:
- [484a lines 70-80](484a-ameide-cli-primitive-workflows.md) - Full check scope
- [primitive.go:414-719](../packages/ameide_core_cli/internal/commands/primitive.go) - Current implementation

**Issue**: `ameide primitive verify` only validates K8s cluster state (conditions, replicas). Per 484a, it should also verify:
- Naming conventions (RPC business verbs, not CRUD)
- EDA wiring (outbox table, event emission)
- Security hooks (secret scan, vuln scan)
- Test execution (unit/integration tests pass)

**Impact**: Agents cannot use `verify` to ensure code quality, only deployment health.

**Fix Required**: Implement `--check naming`, `--check eda`, `--check security`, `--check tests` flags.

### 11.6 Phase L and Phase M Deliverables Unverified

**Location**:
- [Section 2 lines 200-206](#2-implementation-sequence) - Phase L/M acceptance criteria
- [primitive_test.go](../packages/ameide_core_cli/internal/commands/primitive_test.go) - Unit tests only

**Issue**:
- **Phase L (CLI Tests)**: Only unit tests exist. No integration tests that exercise CLI → K8s → Operator flow.
- **Phase M (E2E Demo)**: No demo script exists. Acceptance criteria unmarked.

**Impact**: Cannot verify the vertical slice works end-to-end.

**Fix Required**:
- Phase L: Add integration test that creates Domain CR and verifies CLI output
- Phase M: Create `scripts/demo-domain-slice.sh` that runs full flow

### 11.7 Summary Table

| Gap | Severity | Blocking E2E? | Fix Complexity |
|-----|----------|---------------|----------------|
| Cross-namespace secrets | Medium | No (workaround: same-ns CNPG) | Medium |
| dbMigrationVersion | Low | No | Low |
| Sample with DB | High | **Yes** | Low |
| Proto contract gaps | Medium | No (CLI works without) | Medium |
| Verify repo/tests | Medium | No | High |
| Phase L/M unverified | High | **Yes** | Medium |
