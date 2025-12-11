# 502 – Domain Vertical Slice: End-to-End Implementation

**Status:** Active
**Audience:** Platform engineers, AI agents implementing Domain
**Scope:** Complete vertical slice: Operator + CLI + Proto + GitOps

> **Related**:
> - [498-domain-operator.md](498-domain-operator.md) – Operator development tracking
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – Go patterns
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
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
│  D. Status                             K. JSON Output                   │
│      └─ Conditions                         └─ Matches proto shapes      │
│         ↓                                      ↓                        │
│  E. Helm Chart                         L. Integration Test              │
│         ↓                                  └─ CLI → Operator → Status   │
│  F. GitOps                                                              │
│         ↓                                                               │
│  G. Sample CR                                                           │
│                                                                         │
│  ═══════════════════════════════════════════════════════════════════   │
│                      M. End-to-End Demo                                 │
│      Create Domain CR → Operator reconciles → CLI reports Ready         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Deliverable**: Full loop where:
1. `kubectl apply` a Domain CR
2. Operator reconciles Deployment + Service
3. `ameide primitive describe --kind domain --json` shows status
4. `ameide primitive verify --kind domain --name orders --json` validates health

---

## 1.1 Scope & Non-Scope

### What This Slice Covers

- Domain CRD types with kubebuilder markers
- Basic reconciler: Deployment + Service creation
- Status conditions (Ready, WorkloadReady)
- Helm chart for operator deployment
- CLI `describe` and `verify` commands (read-only)
- Proto message shapes for JSON output

### What This Slice Defers

| Deferred | Reason | Future Slice |
|----------|--------|--------------|
| **DB migrations** | `spec.db` is accepted but not acted on | 498 Phase 2 |
| **CNPG integration** | Schema creation, secret injection | 498 Phase 2 |
| **HPA, NetworkPolicy, ServiceMonitor** | Production readiness | 498 Phase 3 |
| **CLI scaffold command** | Write operations need Git/PR flow | 484d Phase 2 |
| **Transformation integration** | Requires Transformation Domain APIs | 484d Phase 4 |

> **Key constraint**: This slice is **read-only** for CLI operations. Any future write operations (scaffolding GitOps CRs, generating code) will go through Git/PR workflow, not direct imperative API calls.

---

## 1.2 Repository Split

This vertical slice spans **two repositories**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ameide-core (core repo)                                                │
│  ────────────────────────                                               │
│  • operators/domain-operator/     ← CRD types, reconciler, Makefile     │
│  • packages/ameide_core_cli/      ← CLI commands                        │
│  • packages/ameide_core_proto/    ← Proto message definitions           │
│  • charts/ameide-operators/       ← Helm chart templates                │
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
| **D** | Status | `conditions.go` | `status.conditions` updated, `Ready` computed |
| **E** | Helm | `charts/ameide-operators/` | `helm template` renders valid manifests |
| **F** | GitOps | gitops repo changes | ArgoCD syncs operator to dev cluster |
| **G** | Sample CR | gitops `primitives/domain/` | Full reconcile cycle works in dev |

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
| **M** | E2E Demo | Create CR → Operator reconciles → CLI reports Ready |

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

## 6. Phase D: Status & Conditions ✅ IMPLEMENTED

> **Implementation**: See [operators/domain-operator/internal/controller/conditions.go](../operators/domain-operator/internal/controller/conditions.go)

### 6.1 Condition Vocabulary

Uses shared vocabulary from `operators/shared/api/v1/conditions.go`:

| Condition | Meaning |
|-----------|---------|
| `Ready` | Computed: WorkloadReady && !Degraded |
| `WorkloadReady` | Deployment has available replicas |
| `DBReady` | Database schema ready (future) |
| `Degraded` | Running but with errors |

### 6.2 Acceptance Criteria (Phase D) ✅

- [x] `kubectl get domain` shows Ready column
- [x] `status.conditions` includes Ready, WorkloadReady
- [x] `status.readyReplicas` reflects Deployment status
- [x] `status.observedGeneration` matches `metadata.generation`

---

## 7. Phase E-G: Helm, GitOps, Sample CR (Pending)

See [498-domain-operator.md](498-domain-operator.md) Phases 3-4 for detailed tasks.

| Phase | Deliverable | Status |
|-------|-------------|--------|
| **E** | Helm chart in `charts/ameide-operators/` | Pending |
| **F** | ArgoCD Application in gitops repo | Pending |
| **G** | Sample Domain CR in `envs/dev/primitives/domain/` | Pending |

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
| [498-domain-operator.md](498-domain-operator.md) | Operator development phases & acceptance criteria |
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Go patterns & reference implementation |
| [495-ameide-operators.md](495-ameide-operators.md) | CRD shapes & responsibilities |
| [484a-ameide-cli-primitive-workflows.md](484a-ameide-cli-primitive-workflows.md) | CLI commands, TDD loop, agentic workflow |
| [484b-ameide-cli-proto-contract.md](484b-ameide-cli-proto-contract.md) | Proto message definitions for CLI |
| [484d-ameide-cli-migration.md](484d-ameide-cli-migration.md) | CLI phased implementation plan |
