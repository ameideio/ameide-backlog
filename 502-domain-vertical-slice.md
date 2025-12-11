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

## 3. Phase A: CRD Types

### 3.1 Directory Structure

```
operators/domain-operator/
├── api/
│   └── v1/
│       ├── domain_types.go      # Main CRD types
│       ├── groupversion_info.go # API registration
│       └── zz_generated.deepcopy.go # Generated (kubebuilder)
├── cmd/
│   └── main.go                  # Entrypoint
├── internal/
│   └── controller/
│       └── (Phase B)
├── config/
│   ├── crd/
│   │   └── bases/               # Generated CRD YAML
│   ├── rbac/
│   │   └── role.yaml            # Generated RBAC
│   └── manager/
│       └── manager.yaml         # Controller manager deployment
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
└── PROJECT                      # Kubebuilder project file
```

### 3.2 Initialize Module

```bash
# From repo root
mkdir -p operators/domain-operator
cd operators/domain-operator

# Initialize Go module
go mod init github.com/ameideio/ameide/operators/domain-operator

# Add to workspace
cd ../..
echo '  ./operators/domain-operator' >> go.work
```

### 3.3 CRD Types (api/v1/domain_types.go)

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Ready",type="string",JSONPath=".status.conditions[?(@.type=='Ready')].status"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

package v1

import (
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// DomainSpec defines the desired state of Domain
type DomainSpec struct {
    // Image is the container image for this domain service
    // +kubebuilder:validation:Required
    Image string `json:"image"`

    // Version is the semantic version (optional, can be derived from image tag)
    // +optional
    Version string `json:"version,omitempty"`

    // DB configures database connectivity
    // +kubebuilder:validation:Required
    DB DomainDBSpec `json:"db"`

    // Env are additional environment variables
    // +optional
    Env []corev1.EnvVar `json:"env,omitempty"`

    // Resources for the domain container
    // +optional
    Resources corev1.ResourceRequirements `json:"resources,omitempty"`

    // Observability configuration
    // +optional
    Observability DomainObservabilitySpec `json:"observability,omitempty"`

    // Security configuration
    // +optional
    Security DomainSecuritySpec `json:"security,omitempty"`

    // Rollout configuration
    // +optional
    Rollout DomainRolloutSpec `json:"rollout,omitempty"`
}

// DomainDBSpec configures the database for a domain
type DomainDBSpec struct {
    // ClusterRef references a CNPG Cluster (format: namespace/name or just name)
    // +kubebuilder:validation:Required
    ClusterRef string `json:"clusterRef"`

    // Schema is the Postgres schema name for this domain
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Pattern=`^[a-z][a-z0-9_]*$`
    Schema string `json:"schema"`

    // MigrationJobImage is the image that runs Flyway/similar migrations
    // +kubebuilder:validation:Required
    MigrationJobImage string `json:"migrationJobImage"`
}

// DomainObservabilitySpec configures observability
type DomainObservabilitySpec struct {
    // LogLevel for the domain service
    // +kubebuilder:validation:Enum=debug;info;warn;error
    // +kubebuilder:default=info
    LogLevel string `json:"logLevel,omitempty"`
}

// DomainSecuritySpec configures security settings
type DomainSecuritySpec struct {
    // ServiceAccountName for the domain pods
    // +optional
    ServiceAccountName string `json:"serviceAccountName,omitempty"`

    // NetworkProfile selects a predefined NetworkPolicy
    // +kubebuilder:validation:Enum=backend;frontend;isolated
    // +kubebuilder:default=backend
    NetworkProfile string `json:"networkProfile,omitempty"`
}

// DomainRolloutSpec configures deployment strategy
type DomainRolloutSpec struct {
    // Strategy for deployment updates
    // +kubebuilder:validation:Enum=RollingUpdate;Recreate
    // +kubebuilder:default=RollingUpdate
    Strategy string `json:"strategy,omitempty"`

    // Replicas is the desired number of pods
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:default=2
    Replicas int32 `json:"replicas,omitempty"`
}

// DomainStatus defines the observed state of Domain
type DomainStatus struct {
    // Conditions represent the latest available observations
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // ObservedGeneration is the last observed generation
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // DBMigrationVersion is the last successful migration version
    // +optional
    DBMigrationVersion string `json:"dbMigrationVersion,omitempty"`

    // ReadyReplicas is the number of ready pods
    // +optional
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`
}

// +kubebuilder:object:root=true

// Domain is the Schema for the domains API
type Domain struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   DomainSpec   `json:"spec,omitempty"`
    Status DomainStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// DomainList contains a list of Domain
type DomainList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Domain `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Domain{}, &DomainList{})
}
```

### 3.4 Group Version Info (api/v1/groupversion_info.go)

```go
// +kubebuilder:object:generate=true
// +groupName=ameide.io

package v1

import (
    "k8s.io/apimachinery/pkg/runtime/schema"
    "sigs.k8s.io/controller-runtime/pkg/scheme"
)

var (
    // GroupVersion is group version used to register these objects
    GroupVersion = schema.GroupVersion{Group: "ameide.io", Version: "v1"}

    // SchemeBuilder is used to add go types to the GroupVersionKind scheme
    SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}

    // AddToScheme adds the types in this group-version to the given scheme.
    AddToScheme = SchemeBuilder.AddToScheme
)
```

### 3.5 Makefile

```makefile
# operators/domain-operator/Makefile

# Image URL to use all building/pushing image targets
IMG ?= ghcr.io/ameide/domain-operator:dev

# Get the currently used golang install path (in GOPATH/bin, unless GOBIN is set)
ifeq (,$(shell go env GOBIN))
GOBIN=$(shell go env GOPATH)/bin
else
GOBIN=$(shell go env GOBIN)
endif

# ENVTEST_K8S_VERSION refers to the version of kubebuilder assets to be downloaded by envtest binary.
ENVTEST_K8S_VERSION = 1.31.0

## Location to install dependencies to
LOCALBIN ?= $(shell pwd)/bin
$(LOCALBIN):
	mkdir -p $(LOCALBIN)

## Tool Binaries
KUBECTL ?= kubectl
CONTROLLER_GEN ?= $(LOCALBIN)/controller-gen
ENVTEST ?= $(LOCALBIN)/setup-envtest

## Tool Versions
CONTROLLER_TOOLS_VERSION ?= v0.16.5
ENVTEST_VERSION ?= release-0.19

.PHONY: all
all: build

##@ General

.PHONY: help
help: ## Display this help.
	@awk 'BEGIN {FS = ":.*##"; printf "\nUsage:\n  make \033[36m<target>\033[0m\n"} /^[a-zA-Z_0-9-]+:.*?##/ { printf "  \033[36m%-15s\033[0m %s\n", $$1, $$2 } /^##@/ { printf "\n\033[1m%s\033[0m\n", substr($$0, 5) } ' $(MAKEFILE_LIST)

##@ Development

.PHONY: manifests
manifests: controller-gen ## Generate CRD manifests
	$(CONTROLLER_GEN) rbac:roleName=domain-operator-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases

.PHONY: generate
generate: controller-gen ## Generate code (deepcopy, etc.)
	$(CONTROLLER_GEN) object:headerFile="hack/boilerplate.go.txt" paths="./..."

.PHONY: fmt
fmt: ## Run go fmt
	go fmt ./...

.PHONY: vet
vet: ## Run go vet
	go vet ./...

.PHONY: test
test: manifests generate fmt vet envtest ## Run tests
	KUBEBUILDER_ASSETS="$(shell $(ENVTEST) use $(ENVTEST_K8S_VERSION) --bin-dir $(LOCALBIN) -p path)" go test $$(go list ./... | grep -v /e2e) -coverprofile cover.out

.PHONY: test-e2e
test-e2e: ## Run e2e tests (requires running cluster)
	go test ./test/e2e/ -v -ginkgo.v

##@ Build

.PHONY: build
build: manifests generate fmt vet ## Build manager binary
	go build -o bin/manager cmd/main.go

.PHONY: run
run: manifests generate fmt vet ## Run against the configured cluster
	go run cmd/main.go

.PHONY: docker-build
docker-build: ## Build docker image
	docker build -t ${IMG} .

.PHONY: docker-push
docker-push: ## Push docker image
	docker push ${IMG}

##@ Dependencies

.PHONY: controller-gen
controller-gen: $(CONTROLLER_GEN) ## Download controller-gen locally if necessary.
$(CONTROLLER_GEN): $(LOCALBIN)
	$(call go-install-tool,$(CONTROLLER_GEN),sigs.k8s.io/controller-tools/cmd/controller-gen,$(CONTROLLER_TOOLS_VERSION))

.PHONY: envtest
envtest: $(ENVTEST) ## Download setup-envtest locally if necessary.
$(ENVTEST): $(LOCALBIN)
	$(call go-install-tool,$(ENVTEST),sigs.k8s.io/controller-runtime/tools/setup-envtest,$(ENVTEST_VERSION))

# go-install-tool will 'go install' any package with custom target and target version.
define go-install-tool
@[ -f $(1) ] || { \
set -e; \
package=$(2)@$(3) ;\
echo "Downloading $${package}" ;\
GOBIN=$(LOCALBIN) go install $${package} ;\
}
endef
```

### 3.6 Acceptance Criteria (Phase A)

- [ ] `go mod tidy` succeeds
- [ ] `make generate` creates `zz_generated.deepcopy.go`
- [ ] `make manifests` creates `config/crd/bases/ameide.io_domains.yaml`
- [ ] CRD YAML validates with `kubectl apply --dry-run=server`

---

## 4. Phase B: Reconciler Skeleton

### 4.1 Main Entrypoint (cmd/main.go)

```go
package main

import (
    "flag"
    "os"

    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    metricsserver "sigs.k8s.io/controller-runtime/pkg/metrics/server"

    ameidev1 "github.com/ameideio/ameide/operators/domain-operator/api/v1"
    "github.com/ameideio/ameide/operators/domain-operator/internal/controller"
)

var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    utilruntime.Must(ameidev1.AddToScheme(scheme))
}

func main() {
    var metricsAddr string
    var probeAddr string
    var enableLeaderElection bool

    flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
    flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
    flag.BoolVar(&enableLeaderElection, "leader-elect", false, "Enable leader election for controller manager.")

    opts := zap.Options{Development: true}
    opts.BindFlags(flag.CommandLine)
    flag.Parse()

    ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,
        Metrics: metricsserver.Options{
            BindAddress: metricsAddr,
        },
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "domain-operator.ameide.io",
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }

    if err = (&controller.DomainReconciler{
        Client:   mgr.GetClient(),
        Scheme:   mgr.GetScheme(),
        Recorder: mgr.GetEventRecorderFor("domain-controller"),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "Domain")
        os.Exit(1)
    }

    if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up health check")
        os.Exit(1)
    }
    if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up ready check")
        os.Exit(1)
    }

    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

### 4.2 Reconciler (internal/controller/domain_controller.go)

```go
package controller

import (
    "context"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    batchv1 "k8s.io/api/batch/v1"
    corev1 "k8s.io/api/core/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/client-go/tools/record"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/predicate"

    ameidev1 "github.com/ameideio/ameide/operators/domain-operator/api/v1"
)

const (
    domainFinalizerName = "domain.ameide.io/finalizer"
)

// DomainReconciler reconciles a Domain object
type DomainReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    Recorder record.EventRecorder
}

// +kubebuilder:rbac:groups=ameide.io,resources=domains,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=ameide.io,resources=domains/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=ameide.io,resources=domains/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch,resources=jobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=events,verbs=create;patch

func (r *DomainReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx).WithValues("domain", req.NamespacedName)
    log.Info("Reconciling Domain")

    // 1. Fetch Domain
    var domain ameidev1.Domain
    if err := r.Get(ctx, req.NamespacedName, &domain); err != nil {
        if apierrors.IsNotFound(err) {
            log.Info("Domain not found, likely deleted")
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    // 2. Handle deletion
    if !domain.ObjectMeta.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, &domain)
    }

    // 3. Ensure finalizer present
    if !controllerutil.ContainsFinalizer(&domain, domainFinalizerName) {
        log.Info("Adding finalizer")
        controllerutil.AddFinalizer(&domain, domainFinalizerName)
        if err := r.Update(ctx, &domain); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{}, nil // Requeue after finalizer added
    }

    // 4. Validate spec
    if err := r.validateSpec(&domain); err != nil {
        log.Error(err, "Invalid spec")
        r.setCondition(&domain, ConditionDegraded, "InvalidSpec", err.Error())
        if statusErr := r.Status().Update(ctx, &domain); statusErr != nil {
            log.Error(statusErr, "Failed to update status")
        }
        return ctrl.Result{}, nil // Don't requeue invalid specs
    }

    // 5. Reconcile DB (Phase C - stub for now)
    // if res, err := r.reconcileDB(ctx, &domain); err != nil || !res.IsZero() {
    //     return res, err
    // }

    // 6. Reconcile workload
    if res, err := r.reconcileWorkload(ctx, &domain); err != nil || !res.IsZero() {
        return res, err
    }

    // 7. Update status
    if err := r.updateStatus(ctx, &domain); err != nil {
        return ctrl.Result{}, err
    }

    log.Info("Reconciliation complete")
    return ctrl.Result{}, nil
}

func (r *DomainReconciler) reconcileDelete(ctx context.Context, domain *ameidev1.Domain) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    if controllerutil.ContainsFinalizer(domain, domainFinalizerName) {
        log.Info("Running finalizer cleanup")

        // TODO: Clean up external resources (DB schema if policy allows)

        controllerutil.RemoveFinalizer(domain, domainFinalizerName)
        if err := r.Update(ctx, domain); err != nil {
            return ctrl.Result{}, err
        }
    }

    return ctrl.Result{}, nil
}

func (r *DomainReconciler) validateSpec(domain *ameidev1.Domain) error {
    // Basic validation (kubebuilder handles most via markers)
    if domain.Spec.Image == "" {
        return fmt.Errorf("spec.image is required")
    }
    if domain.Spec.DB.ClusterRef == "" {
        return fmt.Errorf("spec.db.clusterRef is required")
    }
    return nil
}

func (r *DomainReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&ameidev1.Domain{}).
        Owns(&appsv1.Deployment{}).
        Owns(&batchv1.Job{}).
        Owns(&corev1.Service{}).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 4,
        }).
        WithEventFilter(predicate.GenerationChangedPredicate{}).
        Complete(r)
}
```

### 4.3 Acceptance Criteria (Phase B)

- [ ] `go build ./cmd/...` succeeds
- [ ] `make run` starts controller, connects to cluster
- [ ] Creating a Domain CR triggers reconcile log
- [ ] Deleting a Domain CR triggers finalizer cleanup log

---

## 5. Phase C: Workload Reconciliation

### 5.1 Reconcile Workload (internal/controller/reconcile_workload.go)

```go
package controller

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/util/intstr"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"

    ameidev1 "github.com/ameideio/ameide/operators/domain-operator/api/v1"
)

func (r *DomainReconciler) reconcileWorkload(ctx context.Context, domain *ameidev1.Domain) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // Reconcile Deployment
    if err := r.reconcileDeployment(ctx, domain); err != nil {
        log.Error(err, "Failed to reconcile Deployment")
        return ctrl.Result{}, err
    }

    // Reconcile Service
    if err := r.reconcileService(ctx, domain); err != nil {
        log.Error(err, "Failed to reconcile Service")
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}

func (r *DomainReconciler) reconcileDeployment(ctx context.Context, domain *ameidev1.Domain) error {
    log := log.FromContext(ctx)

    deploy := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      deploymentName(domain),
            Namespace: domain.Namespace,
        },
    }

    op, err := controllerutil.CreateOrUpdate(ctx, r.Client, deploy, func() error {
        if err := ctrl.SetControllerReference(domain, deploy, r.Scheme); err != nil {
            return err
        }
        mutateDeployment(deploy, domain)
        return nil
    })

    if err != nil {
        return err
    }

    log.Info("Deployment reconciled", "operation", op)
    return nil
}

func (r *DomainReconciler) reconcileService(ctx context.Context, domain *ameidev1.Domain) error {
    log := log.FromContext(ctx)

    svc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      serviceName(domain),
            Namespace: domain.Namespace,
        },
    }

    op, err := controllerutil.CreateOrUpdate(ctx, r.Client, svc, func() error {
        if err := ctrl.SetControllerReference(domain, svc, r.Scheme); err != nil {
            return err
        }
        mutateService(svc, domain)
        return nil
    })

    if err != nil {
        return err
    }

    log.Info("Service reconciled", "operation", op)
    return nil
}

func deploymentName(domain *ameidev1.Domain) string {
    return fmt.Sprintf("%s-domain", domain.Name)
}

func serviceName(domain *ameidev1.Domain) string {
    return fmt.Sprintf("%s-domain", domain.Name)
}

func labels(domain *ameidev1.Domain) map[string]string {
    return map[string]string{
        "app":                      domain.Name,
        "ameide.io/primitive-kind": "domain",
        "ameide.io/primitive-name": domain.Name,
    }
}

func mutateDeployment(deploy *appsv1.Deployment, domain *ameidev1.Domain) {
    replicas := domain.Spec.Rollout.Replicas
    if replicas == 0 {
        replicas = 2
    }

    deploy.Spec.Replicas = &replicas
    deploy.Spec.Selector = &metav1.LabelSelector{
        MatchLabels: labels(domain),
    }

    deploy.Spec.Template.ObjectMeta.Labels = labels(domain)
    deploy.Spec.Template.Spec = corev1.PodSpec{
        ServiceAccountName: domain.Spec.Security.ServiceAccountName,
        Containers: []corev1.Container{{
            Name:      "domain",
            Image:     domain.Spec.Image,
            Env:       buildEnv(domain),
            Resources: domain.Spec.Resources,
            Ports: []corev1.ContainerPort{{
                Name:          "grpc",
                ContainerPort: 8080,
                Protocol:      corev1.ProtocolTCP,
            }},
            LivenessProbe: &corev1.Probe{
                ProbeHandler: corev1.ProbeHandler{
                    GRPC: &corev1.GRPCAction{
                        Port: 8080,
                    },
                },
                InitialDelaySeconds: 10,
                PeriodSeconds:       10,
            },
            ReadinessProbe: &corev1.Probe{
                ProbeHandler: corev1.ProbeHandler{
                    GRPC: &corev1.GRPCAction{
                        Port: 8080,
                    },
                },
                InitialDelaySeconds: 5,
                PeriodSeconds:       5,
            },
        }},
    }
}

func mutateService(svc *corev1.Service, domain *ameidev1.Domain) {
    svc.Spec.Selector = labels(domain)
    svc.Spec.Ports = []corev1.ServicePort{{
        Name:       "grpc",
        Port:       8080,
        TargetPort: intstr.FromString("grpc"),
        Protocol:   corev1.ProtocolTCP,
    }}
    svc.Spec.Type = corev1.ServiceTypeClusterIP
}

func buildEnv(domain *ameidev1.Domain) []corev1.EnvVar {
    env := []corev1.EnvVar{
        {Name: "AMEIDE_DOMAIN_NAME", Value: domain.Name},
        {Name: "AMEIDE_DB_SCHEMA", Value: domain.Spec.DB.Schema},
        {Name: "LOG_LEVEL", Value: domain.Spec.Observability.LogLevel},
    }
    env = append(env, domain.Spec.Env...)
    return env
}
```

### 5.2 Acceptance Criteria (Phase C)

- [ ] Creating Domain CR creates Deployment
- [ ] Creating Domain CR creates Service
- [ ] Deployment has correct labels, image, env
- [ ] Service selector matches Deployment labels
- [ ] Deleting Domain CR deletes owned resources (GC)

---

## 6. Phase D: Status & Conditions

### 6.1 Conditions (internal/controller/conditions.go)

```go
package controller

import (
    "context"

    appsv1 "k8s.io/api/apps/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    "sigs.k8s.io/controller-runtime/pkg/log"

    ameidev1 "github.com/ameideio/ameide/operators/domain-operator/api/v1"
)

const (
    ConditionReady         = "Ready"
    ConditionWorkloadReady = "WorkloadReady"
    ConditionDBReady       = "DBReady"
    ConditionDegraded      = "Degraded"
)

func (r *DomainReconciler) setCondition(domain *ameidev1.Domain, condType, reason, message string) {
    condition := metav1.Condition{
        Type:               condType,
        Status:             metav1.ConditionTrue,
        ObservedGeneration: domain.Generation,
        LastTransitionTime: metav1.Now(),
        Reason:             reason,
        Message:            message,
    }

    // Find and update existing condition or append
    found := false
    for i, c := range domain.Status.Conditions {
        if c.Type == condType {
            domain.Status.Conditions[i] = condition
            found = true
            break
        }
    }
    if !found {
        domain.Status.Conditions = append(domain.Status.Conditions, condition)
    }
}

func (r *DomainReconciler) setConditionStatus(domain *ameidev1.Domain, condType string, status metav1.ConditionStatus, reason, message string) {
    condition := metav1.Condition{
        Type:               condType,
        Status:             status,
        ObservedGeneration: domain.Generation,
        LastTransitionTime: metav1.Now(),
        Reason:             reason,
        Message:            message,
    }

    found := false
    for i, c := range domain.Status.Conditions {
        if c.Type == condType {
            // Only update if status changed
            if c.Status != status {
                domain.Status.Conditions[i] = condition
            }
            found = true
            break
        }
    }
    if !found {
        domain.Status.Conditions = append(domain.Status.Conditions, condition)
    }
}

func (r *DomainReconciler) updateStatus(ctx context.Context, domain *ameidev1.Domain) error {
    log := log.FromContext(ctx)

    // Get Deployment status
    var deploy appsv1.Deployment
    deployKey := types.NamespacedName{
        Namespace: domain.Namespace,
        Name:      deploymentName(domain),
    }
    if err := r.Get(ctx, deployKey, &deploy); err != nil {
        log.Error(err, "Failed to get Deployment for status")
        r.setConditionStatus(domain, ConditionWorkloadReady, metav1.ConditionFalse, "DeploymentNotFound", err.Error())
    } else {
        domain.Status.ReadyReplicas = deploy.Status.ReadyReplicas

        if deploy.Status.ReadyReplicas >= 1 {
            r.setConditionStatus(domain, ConditionWorkloadReady, metav1.ConditionTrue, "DeploymentAvailable", "Deployment has ready replicas")
        } else {
            r.setConditionStatus(domain, ConditionWorkloadReady, metav1.ConditionFalse, "DeploymentNotReady", "Waiting for replicas")
        }
    }

    // TODO: Check DBReady condition from migration Job

    // Compute overall Ready
    r.computeReadyCondition(domain)

    domain.Status.ObservedGeneration = domain.Generation

    log.Info("Updating status", "readyReplicas", domain.Status.ReadyReplicas)
    return r.Status().Update(ctx, domain)
}

func (r *DomainReconciler) computeReadyCondition(domain *ameidev1.Domain) {
    // Ready = WorkloadReady && !Degraded
    workloadReady := false
    degraded := false

    for _, c := range domain.Status.Conditions {
        switch c.Type {
        case ConditionWorkloadReady:
            workloadReady = c.Status == metav1.ConditionTrue
        case ConditionDegraded:
            degraded = c.Status == metav1.ConditionTrue
        }
    }

    if workloadReady && !degraded {
        r.setConditionStatus(domain, ConditionReady, metav1.ConditionTrue, "AllResourcesHealthy", "Domain is ready")
    } else if degraded {
        r.setConditionStatus(domain, ConditionReady, metav1.ConditionFalse, "Degraded", "Domain has errors")
    } else {
        r.setConditionStatus(domain, ConditionReady, metav1.ConditionFalse, "NotReady", "Waiting for workload")
    }
}
```

### 6.2 Acceptance Criteria (Phase D)

- [ ] `kubectl get domain` shows Ready column
- [ ] `status.conditions` includes Ready, WorkloadReady
- [ ] `status.readyReplicas` reflects Deployment status
- [ ] `status.observedGeneration` matches `metadata.generation`

---

## 7. Phase E-H: Helm, GitOps, E2E, CLI

See [498-domain-operator.md](498-domain-operator.md) Phases 3-4 for detailed tasks.

**Quick summary:**

### Phase E: Helm Chart

```
charts/ameide-operators/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── domain-operator/
    │   ├── deployment.yaml
    │   ├── rbac.yaml
    │   ├── service.yaml
    │   └── serviceaccount.yaml
    └── crds/
        └── ameide.io_domains.yaml
```

### Phase F: GitOps

```yaml
# ameide-gitops/envs/dev/apps/platform-operators.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-operators
spec:
  source:
    repoURL: oci://ghcr.io/ameide/helm
    chart: ameide-operators
    targetRevision: 0.1.0
  destination:
    namespace: ameide-system
```

### Phase G: Sample Domain CR

```yaml
# ameide-gitops/envs/dev/primitives/domain/orders.yaml
apiVersion: ameide.io/v1
kind: Domain
metadata:
  name: orders
  namespace: ameide-dev
spec:
  image: ghcr.io/ameide/orders-domain:dev
  db:
    clusterRef: cnpg/orders-cluster
    schema: orders
    migrationJobImage: ghcr.io/ameide/migrator:latest
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
```

---

## 8. Phase H: Proto Messages (CLI)

The CLI uses proto messages to define JSON output shapes. This ensures agents can parse CLI output reliably.

### 8.1 Directory Structure

```
packages/ameide_core_proto/src/ameide_core_proto/
└── primitive/
    └── v1/
        ├── primitive.proto       # Core types (PrimitiveKind enum)
        ├── describe.proto        # DescribeRequest/Result
        ├── verify.proto          # VerifyRequest/Result
        ├── drift.proto           # DriftRequest/Result
        └── impact.proto          # ImpactRequest/Result
```

### 8.2 Core Proto (primitive/v1/primitive.proto)

```proto
syntax = "proto3";

package ameide_core_proto.primitive.v1;

option go_package = "github.com/ameideio/ameide/packages/ameide_core_proto/gen/go/primitive/v1;primitivev1";

// PrimitiveKind identifies the four primitive types
enum PrimitiveKind {
  PRIMITIVE_KIND_UNSPECIFIED = 0;
  PRIMITIVE_KIND_DOMAIN = 1;
  PRIMITIVE_KIND_PROCESS = 2;
  PRIMITIVE_KIND_AGENT = 3;
  PRIMITIVE_KIND_UISURFACE = 4;
}

// Condition mirrors K8s metav1.Condition
message Condition {
  string type = 1;
  string status = 2;      // "True", "False", "Unknown"
  string reason = 3;
  string message = 4;
  string last_transition_time = 5;
}
```

### 8.3 Describe Proto (primitive/v1/describe.proto)

```proto
syntax = "proto3";

package ameide_core_proto.primitive.v1;

import "ameide_core_proto/primitive/v1/primitive.proto";

option go_package = "github.com/ameideio/ameide/packages/ameide_core_proto/gen/go/primitive/v1;primitivev1";

message DescribeRequest {
  PrimitiveKind kind = 1;      // Optional: filter by kind
  string name = 2;              // Optional: filter by name
  string namespace = 3;         // Optional: filter by namespace
}

message DescribeResult {
  repeated PrimitiveInfo primitives = 1;
}

message PrimitiveInfo {
  PrimitiveKind kind = 1;
  string name = 2;
  string namespace = 3;
  string status = 4;            // "Ready", "NotReady", "Degraded", "Unknown"
  repeated Condition conditions = 5;

  // Domain-specific fields
  DomainInfo domain = 10;
  // Process-specific fields
  ProcessInfo process = 11;
  // Agent-specific fields
  AgentInfo agent = 12;
  // UISurface-specific fields
  UISurfaceInfo uisurface = 13;
}

message DomainInfo {
  string image = 1;
  string db_schema = 2;
  string migration_version = 3;
  int32 ready_replicas = 4;
  int32 desired_replicas = 5;
}

message ProcessInfo {
  string definition_id = 1;
  string definition_checksum = 2;
  string temporal_namespace = 3;
  string task_queue = 4;
}

message AgentInfo {
  string definition_id = 1;
  string model_provider = 2;
  string model_name = 3;
  string risk_tier = 4;
}

message UISurfaceInfo {
  string host = 1;
  string path_prefix = 2;
  repeated string scopes = 3;
}
```

### 8.4 Verify Proto (primitive/v1/verify.proto)

```proto
syntax = "proto3";

package ameide_core_proto.primitive.v1;

import "ameide_core_proto/primitive/v1/primitive.proto";

option go_package = "github.com/ameideio/ameide/packages/ameide_core_proto/gen/go/primitive/v1;primitivev1";

message VerifyRequest {
  PrimitiveKind kind = 1;
  string name = 2;
  string namespace = 3;
  repeated string checks = 4;   // Optional: specific checks (naming, eda, security)
}

message VerifyResult {
  string summary = 1;           // "pass" or "fail"
  PrimitiveHealth health = 2;
  repeated CheckResult checks = 3;
  repeated string next_steps = 4;
}

message PrimitiveHealth {
  PrimitiveKind kind = 1;
  string name = 2;
  string namespace = 3;
  bool ready = 4;
  repeated Condition conditions = 5;
}

message CheckResult {
  string check = 1;             // "conditions", "deployment", "service", etc.
  string status = 2;            // "PASS", "FAIL", "WARN", "SKIP"
  string message = 3;
  repeated CheckIssue issues = 4;
}

message CheckIssue {
  string resource = 1;
  string field = 2;
  string expected = 3;
  string actual = 4;
}
```

### 8.5 Acceptance Criteria (Phase H)

- [ ] `buf lint` passes on all primitive protos
- [ ] `buf generate` creates Go types in `gen/go/primitive/v1/`
- [ ] Generated types are importable in CLI code

---

## 9. Phase I: CLI Commands

### 9.1 CLI Structure

```
packages/ameide_core_cli/
├── cmd/
│   ├── root.go
│   └── primitive/
│       ├── primitive.go        # Parent command
│       ├── describe.go         # ameide primitive describe
│       ├── verify.go           # ameide primitive verify
│       ├── drift.go            # ameide primitive drift
│       └── impact.go           # ameide primitive impact
├── internal/
│   ├── k8s/
│   │   ├── client.go           # K8s client factory
│   │   └── domain.go           # Domain CR operations
│   ├── output/
│   │   └── json.go             # JSON output formatting
│   └── checks/
│       ├── conditions.go       # Check CR conditions
│       ├── deployment.go       # Check Deployment health
│       └── service.go          # Check Service exists
├── go.mod
└── main.go
```

### 9.2 Primitive Parent Command (cmd/primitive/primitive.go)

```go
package primitive

import (
    "github.com/spf13/cobra"
)

func NewPrimitiveCmd() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "primitive",
        Short: "Manage Ameide primitives (Domain, Process, Agent, UISurface)",
        Long: `Commands for observing and verifying primitive resources.

Primitives are the four building blocks of Ameide:
  - Domain:    Bounded contexts with data and gRPC APIs
  - Process:   Long-running orchestrations (Temporal)
  - Agent:     LLM-powered autonomous actors
  - UISurface: Next.js web applications`,
    }

    cmd.AddCommand(newDescribeCmd())
    cmd.AddCommand(newVerifyCmd())
    cmd.AddCommand(newDriftCmd())
    cmd.AddCommand(newImpactCmd())

    return cmd
}
```

### 9.3 Describe Command (cmd/primitive/describe.go)

```go
package primitive

import (
    "context"
    "encoding/json"
    "fmt"
    "os"

    "github.com/spf13/cobra"

    primitivev1 "github.com/ameideio/ameide/packages/ameide_core_proto/gen/go/primitive/v1"
    "github.com/ameideio/ameide/packages/ameide_core_cli/internal/k8s"
)

var (
    describeKind      string
    describeName      string
    describeNamespace string
    describeJSON      bool
)

func newDescribeCmd() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "describe",
        Short: "Describe primitives and their status",
        Long: `Show current state of primitive resources.

Examples:
  # List all primitives
  ameide primitive describe --json

  # Describe specific Domain
  ameide primitive describe --kind domain --name orders --json

  # Filter by namespace
  ameide primitive describe --namespace ameide-dev --json`,
        RunE: runDescribe,
    }

    cmd.Flags().StringVar(&describeKind, "kind", "", "Filter by primitive kind (domain, process, agent, uisurface)")
    cmd.Flags().StringVar(&describeName, "name", "", "Filter by name")
    cmd.Flags().StringVar(&describeNamespace, "namespace", "", "Filter by namespace")
    cmd.Flags().BoolVar(&describeJSON, "json", false, "Output as JSON")

    return cmd
}

func runDescribe(cmd *cobra.Command, args []string) error {
    ctx := context.Background()

    client, err := k8s.NewClient()
    if err != nil {
        return fmt.Errorf("failed to create k8s client: %w", err)
    }

    req := &primitivev1.DescribeRequest{
        Kind:      parseKind(describeKind),
        Name:      describeName,
        Namespace: describeNamespace,
    }

    result, err := describePrimitives(ctx, client, req)
    if err != nil {
        return err
    }

    if describeJSON {
        enc := json.NewEncoder(os.Stdout)
        enc.SetIndent("", "  ")
        return enc.Encode(result)
    }

    // Human-readable output
    for _, p := range result.Primitives {
        fmt.Printf("%s/%s (%s): %s\n",
            p.Kind.String(), p.Name, p.Namespace, p.Status)
    }
    return nil
}

func describePrimitives(ctx context.Context, client *k8s.Client, req *primitivev1.DescribeRequest) (*primitivev1.DescribeResult, error) {
    result := &primitivev1.DescribeResult{
        Primitives: []*primitivev1.PrimitiveInfo{},
    }

    // If kind is Domain or unspecified, list Domains
    if req.Kind == primitivev1.PrimitiveKind_PRIMITIVE_KIND_UNSPECIFIED ||
       req.Kind == primitivev1.PrimitiveKind_PRIMITIVE_KIND_DOMAIN {
        domains, err := client.ListDomains(ctx, req.Namespace)
        if err != nil {
            return nil, err
        }
        for _, d := range domains {
            if req.Name != "" && d.Name != req.Name {
                continue
            }
            result.Primitives = append(result.Primitives, domainToInfo(&d))
        }
    }

    // Similar for Process, Agent, UISurface...

    return result, nil
}

func parseKind(s string) primitivev1.PrimitiveKind {
    switch s {
    case "domain":
        return primitivev1.PrimitiveKind_PRIMITIVE_KIND_DOMAIN
    case "process":
        return primitivev1.PrimitiveKind_PRIMITIVE_KIND_PROCESS
    case "agent":
        return primitivev1.PrimitiveKind_PRIMITIVE_KIND_AGENT
    case "uisurface":
        return primitivev1.PrimitiveKind_PRIMITIVE_KIND_UISURFACE
    default:
        return primitivev1.PrimitiveKind_PRIMITIVE_KIND_UNSPECIFIED
    }
}
```

### 9.4 Verify Command (cmd/primitive/verify.go)

```go
package primitive

import (
    "context"
    "encoding/json"
    "fmt"
    "os"

    "github.com/spf13/cobra"

    primitivev1 "github.com/ameideio/ameide/packages/ameide_core_proto/gen/go/primitive/v1"
    "github.com/ameideio/ameide/packages/ameide_core_cli/internal/k8s"
    "github.com/ameideio/ameide/packages/ameide_core_cli/internal/checks"
)

var (
    verifyKind      string
    verifyName      string
    verifyNamespace string
    verifyChecks    []string
    verifyJSON      bool
)

func newVerifyCmd() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "verify",
        Short: "Verify primitive health and configuration",
        Long: `Run verification checks on primitive resources.

Checks include:
  - conditions: CR status conditions are healthy
  - deployment: Deployment is available
  - service: Service exists and has endpoints

Examples:
  # Verify specific Domain
  ameide primitive verify --kind domain --name orders --json

  # Run specific checks
  ameide primitive verify --kind domain --name orders --check conditions --json`,
        RunE: runVerify,
    }

    cmd.Flags().StringVar(&verifyKind, "kind", "", "Primitive kind (required)")
    cmd.Flags().StringVar(&verifyName, "name", "", "Primitive name (required)")
    cmd.Flags().StringVar(&verifyNamespace, "namespace", "default", "Namespace")
    cmd.Flags().StringSliceVar(&verifyChecks, "check", nil, "Specific checks to run")
    cmd.Flags().BoolVar(&verifyJSON, "json", false, "Output as JSON")

    cmd.MarkFlagRequired("kind")
    cmd.MarkFlagRequired("name")

    return cmd
}

func runVerify(cmd *cobra.Command, args []string) error {
    ctx := context.Background()

    client, err := k8s.NewClient()
    if err != nil {
        return fmt.Errorf("failed to create k8s client: %w", err)
    }

    req := &primitivev1.VerifyRequest{
        Kind:      parseKind(verifyKind),
        Name:      verifyName,
        Namespace: verifyNamespace,
        Checks:    verifyChecks,
    }

    result, err := verifyPrimitive(ctx, client, req)
    if err != nil {
        return err
    }

    if verifyJSON {
        enc := json.NewEncoder(os.Stdout)
        enc.SetIndent("", "  ")
        return enc.Encode(result)
    }

    // Human-readable output
    fmt.Printf("Verify %s/%s: %s\n", req.Kind, req.Name, result.Summary)
    for _, c := range result.Checks {
        icon := "✓"
        if c.Status == "FAIL" {
            icon = "✗"
        } else if c.Status == "WARN" {
            icon = "!"
        }
        fmt.Printf("  %s %s: %s\n", icon, c.Check, c.Message)
    }

    if result.Summary == "fail" {
        os.Exit(1)
    }
    return nil
}

func verifyPrimitive(ctx context.Context, client *k8s.Client, req *primitivev1.VerifyRequest) (*primitivev1.VerifyResult, error) {
    result := &primitivev1.VerifyResult{
        Summary:   "pass",
        Checks:    []*primitivev1.CheckResult{},
        NextSteps: []string{},
    }

    switch req.Kind {
    case primitivev1.PrimitiveKind_PRIMITIVE_KIND_DOMAIN:
        return verifyDomain(ctx, client, req, result)
    // case primitivev1.PrimitiveKind_PRIMITIVE_KIND_PROCESS:
    //     return verifyProcess(ctx, client, req, result)
    default:
        return nil, fmt.Errorf("unsupported kind: %s", req.Kind)
    }
}

func verifyDomain(ctx context.Context, client *k8s.Client, req *primitivev1.VerifyRequest, result *primitivev1.VerifyResult) (*primitivev1.VerifyResult, error) {
    domain, err := client.GetDomain(ctx, req.Namespace, req.Name)
    if err != nil {
        return nil, fmt.Errorf("failed to get Domain: %w", err)
    }

    // Check conditions
    condCheck := checks.CheckDomainConditions(domain)
    result.Checks = append(result.Checks, condCheck)
    if condCheck.Status == "FAIL" {
        result.Summary = "fail"
    }

    // Check deployment
    deployCheck, err := checks.CheckDomainDeployment(ctx, client, domain)
    if err != nil {
        return nil, err
    }
    result.Checks = append(result.Checks, deployCheck)
    if deployCheck.Status == "FAIL" {
        result.Summary = "fail"
    }

    // Check service
    svcCheck, err := checks.CheckDomainService(ctx, client, domain)
    if err != nil {
        return nil, err
    }
    result.Checks = append(result.Checks, svcCheck)
    if svcCheck.Status == "FAIL" {
        result.Summary = "fail"
    }

    // Build health info
    result.Health = &primitivev1.PrimitiveHealth{
        Kind:      primitivev1.PrimitiveKind_PRIMITIVE_KIND_DOMAIN,
        Name:      domain.Name,
        Namespace: domain.Namespace,
        Ready:     result.Summary == "pass",
    }

    return result, nil
}
```

### 9.5 Acceptance Criteria (Phase I)

- [ ] `ameide primitive --help` shows all subcommands
- [ ] `ameide primitive describe --json` outputs valid JSON
- [ ] `ameide primitive verify --kind domain --name orders --json` runs checks

---

## 10. Phase J: K8s Client

### 10.1 Client Factory (internal/k8s/client.go)

```go
package k8s

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/client-go/tools/clientcmd"
    "sigs.k8s.io/controller-runtime/pkg/client"

    ameidev1 "github.com/ameideio/ameide/operators/domain-operator/api/v1"
)

type Client struct {
    client.Client
}

func NewClient() (*Client, error) {
    // Load kubeconfig
    loadingRules := clientcmd.NewDefaultClientConfigLoadingRules()
    configOverrides := &clientcmd.ConfigOverrides{}
    kubeConfig := clientcmd.NewNonInteractiveDeferredLoadingClientConfig(loadingRules, configOverrides)

    config, err := kubeConfig.ClientConfig()
    if err != nil {
        return nil, fmt.Errorf("failed to load kubeconfig: %w", err)
    }

    // Build scheme with our CRDs
    scheme := runtime.NewScheme()
    if err := ameidev1.AddToScheme(scheme); err != nil {
        return nil, fmt.Errorf("failed to add ameide types to scheme: %w", err)
    }

    // Create client
    c, err := client.New(config, client.Options{Scheme: scheme})
    if err != nil {
        return nil, fmt.Errorf("failed to create client: %w", err)
    }

    return &Client{Client: c}, nil
}
```

### 10.2 Domain Operations (internal/k8s/domain.go)

```go
package k8s

import (
    "context"

    "sigs.k8s.io/controller-runtime/pkg/client"

    ameidev1 "github.com/ameideio/ameide/operators/domain-operator/api/v1"
)

func (c *Client) ListDomains(ctx context.Context, namespace string) ([]ameidev1.Domain, error) {
    list := &ameidev1.DomainList{}
    opts := []client.ListOption{}
    if namespace != "" {
        opts = append(opts, client.InNamespace(namespace))
    }

    if err := c.List(ctx, list, opts...); err != nil {
        return nil, err
    }

    return list.Items, nil
}

func (c *Client) GetDomain(ctx context.Context, namespace, name string) (*ameidev1.Domain, error) {
    domain := &ameidev1.Domain{}
    key := client.ObjectKey{Namespace: namespace, Name: name}

    if err := c.Get(ctx, key, domain); err != nil {
        return nil, err
    }

    return domain, nil
}
```

### 10.3 Acceptance Criteria (Phase J)

- [ ] `NewClient()` successfully connects to cluster
- [ ] `ListDomains()` returns Domain CRs
- [ ] `GetDomain()` returns specific Domain

---

## 11. Phase K: Check Implementations

### 11.1 Condition Checks (internal/checks/conditions.go)

```go
package checks

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

    primitivev1 "github.com/ameideio/ameide/packages/ameide_core_proto/gen/go/primitive/v1"
    ameidev1 "github.com/ameideio/ameide/operators/domain-operator/api/v1"
)

func CheckDomainConditions(domain *ameidev1.Domain) *primitivev1.CheckResult {
    result := &primitivev1.CheckResult{
        Check:  "conditions",
        Status: "PASS",
        Issues: []*primitivev1.CheckIssue{},
    }

    // Check Ready condition
    readyFound := false
    for _, cond := range domain.Status.Conditions {
        if cond.Type == "Ready" {
            readyFound = true
            if cond.Status != metav1.ConditionTrue {
                result.Status = "FAIL"
                result.Message = "Domain is not Ready"
                result.Issues = append(result.Issues, &primitivev1.CheckIssue{
                    Resource: "Domain",
                    Field:    "status.conditions[Ready]",
                    Expected: "True",
                    Actual:   string(cond.Status),
                })
            } else {
                result.Message = "Domain is Ready"
            }
            break
        }
    }

    if !readyFound {
        result.Status = "FAIL"
        result.Message = "Ready condition not found"
        result.Issues = append(result.Issues, &primitivev1.CheckIssue{
            Resource: "Domain",
            Field:    "status.conditions",
            Expected: "Ready condition present",
            Actual:   "Missing",
        })
    }

    return result
}
```

### 11.2 Deployment Checks (internal/checks/deployment.go)

```go
package checks

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"

    primitivev1 "github.com/ameideio/ameide/packages/ameide_core_proto/gen/go/primitive/v1"
    ameidev1 "github.com/ameideio/ameide/operators/domain-operator/api/v1"
    "github.com/ameideio/ameide/packages/ameide_core_cli/internal/k8s"
)

func CheckDomainDeployment(ctx context.Context, c *k8s.Client, domain *ameidev1.Domain) (*primitivev1.CheckResult, error) {
    result := &primitivev1.CheckResult{
        Check:  "deployment",
        Status: "PASS",
        Issues: []*primitivev1.CheckIssue{},
    }

    deployName := fmt.Sprintf("%s-domain", domain.Name)
    deploy := &appsv1.Deployment{}
    key := client.ObjectKey{Namespace: domain.Namespace, Name: deployName}

    if err := c.Get(ctx, key, deploy); err != nil {
        result.Status = "FAIL"
        result.Message = fmt.Sprintf("Deployment %s not found", deployName)
        return result, nil
    }

    // Check replicas
    desired := int32(1)
    if deploy.Spec.Replicas != nil {
        desired = *deploy.Spec.Replicas
    }

    if deploy.Status.ReadyReplicas < desired {
        result.Status = "FAIL"
        result.Message = fmt.Sprintf("Deployment has %d/%d ready replicas",
            deploy.Status.ReadyReplicas, desired)
        result.Issues = append(result.Issues, &primitivev1.CheckIssue{
            Resource: "Deployment",
            Field:    "status.readyReplicas",
            Expected: fmt.Sprintf("%d", desired),
            Actual:   fmt.Sprintf("%d", deploy.Status.ReadyReplicas),
        })
    } else {
        result.Message = fmt.Sprintf("Deployment has %d/%d ready replicas",
            deploy.Status.ReadyReplicas, desired)
    }

    return result, nil
}
```

### 11.3 Acceptance Criteria (Phase K)

- [ ] `CheckDomainConditions` correctly evaluates Ready condition
- [ ] `CheckDomainDeployment` finds Deployment and checks replicas
- [ ] JSON output matches proto shapes exactly

---

## 12. Phase L: CLI Tests

### 12.1 Unit Tests (cmd/primitive/describe_test.go)

```go
package primitive

import (
    "testing"

    "github.com/stretchr/testify/assert"

    primitivev1 "github.com/ameideio/ameide/packages/ameide_core_proto/gen/go/primitive/v1"
)

func TestParseKind(t *testing.T) {
    tests := []struct {
        input    string
        expected primitivev1.PrimitiveKind
    }{
        {"domain", primitivev1.PrimitiveKind_PRIMITIVE_KIND_DOMAIN},
        {"process", primitivev1.PrimitiveKind_PRIMITIVE_KIND_PROCESS},
        {"agent", primitivev1.PrimitiveKind_PRIMITIVE_KIND_AGENT},
        {"uisurface", primitivev1.PrimitiveKind_PRIMITIVE_KIND_UISURFACE},
        {"unknown", primitivev1.PrimitiveKind_PRIMITIVE_KIND_UNSPECIFIED},
        {"", primitivev1.PrimitiveKind_PRIMITIVE_KIND_UNSPECIFIED},
    }

    for _, tt := range tests {
        t.Run(tt.input, func(t *testing.T) {
            result := parseKind(tt.input)
            assert.Equal(t, tt.expected, result)
        })
    }
}
```

### 12.2 Integration Tests (internal/checks/conditions_test.go)

```go
package checks

import (
    "testing"

    "github.com/stretchr/testify/assert"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

    ameidev1 "github.com/ameideio/ameide/operators/domain-operator/api/v1"
)

func TestCheckDomainConditions_Ready(t *testing.T) {
    domain := &ameidev1.Domain{
        Status: ameidev1.DomainStatus{
            Conditions: []metav1.Condition{
                {
                    Type:   "Ready",
                    Status: metav1.ConditionTrue,
                    Reason: "AllResourcesHealthy",
                },
            },
        },
    }

    result := CheckDomainConditions(domain)

    assert.Equal(t, "PASS", result.Status)
    assert.Equal(t, "Domain is Ready", result.Message)
    assert.Empty(t, result.Issues)
}

func TestCheckDomainConditions_NotReady(t *testing.T) {
    domain := &ameidev1.Domain{
        Status: ameidev1.DomainStatus{
            Conditions: []metav1.Condition{
                {
                    Type:   "Ready",
                    Status: metav1.ConditionFalse,
                    Reason: "DeploymentNotReady",
                },
            },
        },
    }

    result := CheckDomainConditions(domain)

    assert.Equal(t, "FAIL", result.Status)
    assert.Len(t, result.Issues, 1)
}
```

### 12.3 Acceptance Criteria (Phase L)

- [ ] `go test ./cmd/primitive/...` passes
- [ ] `go test ./internal/checks/...` passes
- [ ] Tests cover happy path and error cases

---

## 13. Phase M: End-to-End Demo

### 13.1 Demo Script

```bash
#!/bin/bash
# demo-domain-vertical-slice.sh

set -e

echo "=== Domain Vertical Slice E2E Demo ==="

# 1. Apply Domain CR
echo "1. Creating Domain CR..."
kubectl apply -f - <<EOF
apiVersion: ameide.io/v1
kind: Domain
metadata:
  name: demo-orders
  namespace: default
spec:
  image: nginx:latest  # Placeholder image for demo
  db:
    clusterRef: cnpg/demo
    schema: orders
    migrationJobImage: busybox:latest
  rollout:
    replicas: 1
EOF

# 2. Wait for operator to reconcile
echo "2. Waiting for operator to reconcile..."
sleep 5

# 3. Check with kubectl
echo "3. Checking with kubectl..."
kubectl get domain demo-orders -o wide
kubectl get deployment demo-orders-domain
kubectl get service demo-orders-domain

# 4. Check with CLI describe
echo "4. Running CLI describe..."
ameide primitive describe --kind domain --name demo-orders --json | jq .

# 5. Check with CLI verify
echo "5. Running CLI verify..."
ameide primitive verify --kind domain --name demo-orders --json | jq .

# 6. Cleanup
echo "6. Cleanup..."
kubectl delete domain demo-orders

echo "=== Demo Complete ==="
```

### 13.2 Expected Output

```json
// ameide primitive describe --kind domain --name demo-orders --json
{
  "primitives": [
    {
      "kind": "PRIMITIVE_KIND_DOMAIN",
      "name": "demo-orders",
      "namespace": "default",
      "status": "Ready",
      "conditions": [
        {"type": "Ready", "status": "True", "reason": "AllResourcesHealthy"},
        {"type": "WorkloadReady", "status": "True", "reason": "DeploymentAvailable"}
      ],
      "domain": {
        "image": "nginx:latest",
        "db_schema": "orders",
        "ready_replicas": 1,
        "desired_replicas": 1
      }
    }
  ]
}
```

```json
// ameide primitive verify --kind domain --name demo-orders --json
{
  "summary": "pass",
  "health": {
    "kind": "PRIMITIVE_KIND_DOMAIN",
    "name": "demo-orders",
    "namespace": "default",
    "ready": true
  },
  "checks": [
    {"check": "conditions", "status": "PASS", "message": "Domain is Ready"},
    {"check": "deployment", "status": "PASS", "message": "Deployment has 1/1 ready replicas"},
    {"check": "service", "status": "PASS", "message": "Service exists"}
  ],
  "next_steps": []
}
```

### 13.3 Acceptance Criteria (Phase M)

- [ ] Demo script runs without errors
- [ ] Domain CR creates Deployment + Service
- [ ] CLI describe shows correct status
- [ ] CLI verify returns "pass" when healthy
- [ ] CLI verify returns "fail" with issues when unhealthy
- [ ] Cleanup removes all resources

---

## 14. Dependencies (go.mod)

```go
module github.com/ameideio/ameide/operators/domain-operator

go 1.25

require (
    k8s.io/api v0.32.0
    k8s.io/apimachinery v0.32.0
    k8s.io/client-go v0.32.0
    sigs.k8s.io/controller-runtime v0.19.3
)
```

---

## 9. Test Strategy

| Test Type | Framework | What to Test |
|-----------|-----------|--------------|
| **Unit** | `go test` + fake client | Reconcile logic, condition computation |
| **envtest** | controller-runtime envtest | Create/update/delete Domain, verify children |
| **E2E** | Ginkgo + real cluster | Full reconcile cycle, status propagation |

**Unit test example:**

```go
func TestComputeReadyCondition(t *testing.T) {
    domain := &ameidev1.Domain{
        Status: ameidev1.DomainStatus{
            Conditions: []metav1.Condition{
                {Type: "WorkloadReady", Status: metav1.ConditionTrue},
            },
        },
    }

    r := &DomainReconciler{}
    r.computeReadyCondition(domain)

    ready := findCondition(domain.Status.Conditions, "Ready")
    assert.Equal(t, metav1.ConditionTrue, ready.Status)
}
```

---

## 16. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [498-domain-operator.md](498-domain-operator.md) | Operator development phases & acceptance criteria |
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Go patterns & reference implementation |
| [495-ameide-operators.md](495-ameide-operators.md) | CRD shapes & responsibilities |
| [484a-ameide-cli-primitive-workflows.md](484a-ameide-cli-primitive-workflows.md) | CLI commands, TDD loop, agentic workflow |
| [484b-ameide-cli-proto-contract.md](484b-ameide-cli-proto-contract.md) | Proto message definitions for CLI |
| [484d-ameide-cli-migration.md](484d-ameide-cli-migration.md) | CLI phased implementation plan |
