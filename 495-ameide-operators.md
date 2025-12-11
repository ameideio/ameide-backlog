## 0. Shared operator pattern

All four primitive operators share the same skeleton:

* Implemented in Go with `controller-runtime`
* One controller per CRD kind: `Domain`, `Process`, `Agent`, `UISurface`
* GitOps applies only the CRs; operators own all children (Deployments, Jobs, HPAs, CNPG bits, ServiceMonitors, NetworkPolicies, etc.) 
* CRDs are *runtime wiring only*, they never model tables, fields, or UI layout 

**Common structure per operator:**

* `spec` – desired state (image, version, bindings, resources…)
* `status` – observed state + conditions
* Reconcile loop:

  1. Validate spec (cheap checks)
  2. Compute desired child objects
  3. Create/patch children
  4. Derive `status.conditions` from children
* Finalizers for clean‑up (DB schema, queues, agent secrets, routes)

Think of each operator as a **compiler from a Primitive CRD to a bundle of K8s + external infra objects**, with *no* business logic inside (that lives in the primitive's own service).

> **Scaffolding context**: For how operator patterns relate to Ameide's overall code generation philosophy (vs Kubebuilder, Backstage, Buf), see [484-ameide-cli.md](484-ameide-cli.md).
>
> **Implementation patterns**: For Go-level reconciler design, controller-runtime usage, and testing strategies, see [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md).

---

## 1. Domain operator (Domain → app + DB + infra)

**CRD shape (conceptual)**

```yaml
apiVersion: ameide.io/v1
kind: Domain
metadata:
  name: orders
spec:
  image: ghcr.io/ameide/orders-domain:1.12.3
  proto:
    module: github.com/ameide/apis/domains/orders
    version: v1.5.0
  db:
    clusterRef: cnpg/orders
    schema: orders
    migrationJobImage: ghcr.io/ameide/migrator:latest
  resources:
    cpu: "500m"
    memory: "512Mi"
  env:
    - name: FEATURE_FLAG_X
      value: "true"
  observability:
    logLevel: info
  security:
    serviceAccountName: orders-domain
    networkProfile: backend
  rollout:
    strategy: RollingUpdate
    canary: false
status:
  conditions:
    - type: Ready
      status: "True"
      reason: AllResourcesHealthy
```

**Responsibilities** (already hinted in 472/473)  

1. **Validate spec**

   * Image/tag present & allowed (policy check)
   * DB binding is valid (CNPG cluster exists, schema name acceptable)
   * Proto labels consistent if present (optional early check)

2. **Reconcile database layer**

   * Ensure a **schema + user** in CNPG for this domain:

     * Either via a `Cluster`/`Database` CRD (if using CNPG sub‑resources)
     * Or by creating a one‑shot **migration Job** with privileged DB creds
   * Run Flyway (or similar) migrations from the domain image or a dedicated migrator image
   * Update `status.conditions`:

     * `MigrationSucceeded` / `MigrationFailed`
     * Include last migration version in `status.db.migrationVersion`

3. **Reconcile application runtime**

   * Build a **Deployment** spec:

     * Image from `spec.image`
     * Env from `spec.env` + platform standard env (tenant headers, tracing, etc.)
     * DB connection injected from `ExternalSecret` / CNPG secret
     * Resource requests/limits from `spec.resources`
   * Ensure:

     * `Service` (ClusterIP) for internal traffic
     * `HPA` if enabled
     * `NetworkPolicy` following `spec.security.networkProfile`
     * `ServiceMonitor` for Prometheus

4. **Status & health**

   * Watch `Deployment`, `Job` (migrations), `ServiceMonitor`:

     * If Deployment not ready → `Ready=False`, `reason=DeploymentNotReady`
     * If migrations failed → `Ready=False`, `reason=MigrationFailed`
   * Surface a small set of conditions:

     * `Ready`, `Progressing`, `Degraded`, `MigrationFailed`

5. **Deletion/finalizers**

   * On `Domain` deletion:

     * Optionally drop schema (depending on retention policy)
     * Remove monitoring, HPAs, NetworkPolicies
   * Remove finalizer once child resources are gone

**Deliberate non‑responsibilities**

* No knowledge of **entity schemas** or **business rules**
* No rewriting of config *inside* the image
* No direct tenant per‑row logic (that’s enforced by the app + DB RLS)

---

## 2. Process operator (Process → Temporal workers + BPMN bindings)

Processes are special because they tie Temporal and design‑time **ProcessDefinitions** together. 

**CRD shape (conceptual)**

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
  conditions: [...]
  definition:
    version: 3
    checksum: "sha256:..."
```

**Responsibilities** 

1. **Validate references**

   * Check `definitionRef` points to an existing **ProcessDefinition** in Transformation domain (via HTTP/gRPC call)
   * Ensure Temporal namespace exists and is reachable
   * Validate that referenced Domain/Agent primitives exist and are `Ready`

2. **Materialize the definition**

   * Fetch ProcessDefinition (BPMN‑compliant JSON) from Transformation Domain
   * Optionally convert/compile it into a Temporal‑friendly artifact (e.g. mapping table, config)
   * Store it in a **ConfigMap**:

     * Used by workers to bind BPMN task IDs to specific activities / RPCs
   * Stamp `status.definition.version` + `checksum` so we know which definition is currently active

3. **Reconcile Temporal workers**

   * Create a **Deployment** for Temporal workers:

     * Image from `spec.image`
     * Env:

       * `AMEIDE_PROCESS_DEFINITION_ID`
       * `TEMPORAL_NAMESPACE`
       * `TASK_QUEUE`
     * Mount: ConfigMap with compiled definition
   * For higher safety:

     * Canary behaviour: if `spec.rollout.strategy=Canary`, spin up a small canary Deployment and only switch traffic when healthy

4. **Optionally reconcile compensations/timers**

   * For some processes, we might define **CronJobs** for:

     * SLA checks
     * Timeout sweeps
     * Cleanup processes

5. **Status & health**

   * Conditions:

     * `DefinitionResolved` (Transformation definition fetched & validated)
     * `WorkersReady`
     * `Degraded` if Temporal connectivity issues
   * Surface last seen Temporal health (optional: ping Temporal admin APIs)

6. **Deletion**

   * Optionally:

     * Stop new workflow starts (e.g. update some control flag)
     * Let existing workflows finish (or cancel on policy)
     * Tear down worker Deployment once safe

**Non‑responsibilities**

* Operator does not implement *any* process business logic; it just ensures:

  * correct workers exist
  * they know which definition to run
  * they are wired to the right domains/agents & Temporal namespace

---

## 3. Agent operator (Agent → LLM runtime + tool grants)

Agent operator wires **Agent CRDs** to actual LLM runtimes with the right tools and secrets. 

**CRD shape (conceptual)**

```yaml
apiVersion: ameide.io/v1
kind: Agent
metadata:
  name: core-platform-coder
spec:
  image: ghcr.io/ameide/agent-langgraph:2.1.0
  definitionRef:
    id: core-platform-coder-v4
    tenantId: t123
  model:
    provider: openai
    name: gpt-5.1-pro
  tools:
    allowed:
      - domain: transformation
      - process: l2o
      - primitive: primitive-cli
  riskTier: high
  concurrency:
    maxParallelSessions: 20
  observability:
    logPrompts: redacted
    emitTokens: true
status:
  conditions: [...]
```

**Responsibilities** 

1. **Validate references & policy**

   * Ensure `definitionRef` points to an **AgentDefinition** in Transformation
   * Check `riskTier` + `tools.allowed` against central security policy:

     * e.g., some tools only allowed for `riskTier=high`
   * Validate model/provider is allowed for tenant/SKU

2. **Reconcile secrets**

   * Create/ensure `ExternalSecret` entries for:

     * LLM provider API keys
     * Tool‑specific creds (e.g. GitHub token, Jira token)
   * Mount secrets into agent Deployment as env or files

3. **Reconcile agent runtime Deployment**

   * Container image from `spec.image`
   * Env:

     * `AGENT_DEFINITION_ID`, `TENANT_ID`, model config
   * Configure concurrency via:

     * Horizontal Pod Autoscaler (based on queue depth or CPU)
     * Or language‑level concurrency flags

4. **Tool grants & policy enforcement**

   * Option 1 (K8s‑side): create a `ConfigMap` listing allowed tools; agent runtime reads it
   * Option 2 (central service): call a “tool grants” service per agent and maintain a local cache
   * In both cases, the operator ensures the config matches `spec.tools.allowed`

5. **Status & health**

   * Conditions:

     * `DefinitionResolved`
     * `SecretsReady`
     * `RuntimeReady`
     * `PolicyViolation` (e.g. illegal tool+riskTier combination, or missing secret)
   * Expose metrics tags so security dashboards can filter by `riskTier`, `tenantId`, etc. 

6. **Deletion**

   * Clean up Deployment/Service/Secrets
   * Possibly revoke tool grants in central registry

**Non‑responsibilities**

* No prompt orchestration logic inside operator
* No direct calls to LLM APIs
* No persistent state: chats, logs, decisions live in agent services / threads/chat domains, not in the operator

---

## 4. UISurface operator (UISurface → Next.js app + routing + auth)

UISurface operator is basically “make this Next.js app exist in the right namespace with the right routing & auth.”

**CRD shape (conceptual)**

```yaml
apiVersion: ameide.io/v1
kind: UISurface
metadata:
  name: www-ameide-platform
spec:
  image: ghcr.io/ameide/www_ameide_platform:3.0.1
  host: app.ameide.com
  pathPrefix: /
  auth:
    scopes:
      - l2o:read
      - orders:write
  dependencies:
    domains:
      - platform
      - sales
    processes:
      - l2o
  features:
    flags:
      enableTransformation: true
  resources:
    cpu: "500m"
    memory: "1Gi"
status:
  conditions: [...]
```

**Responsibilities**

1. **Validate dependencies**

   * Check all referenced Domain/Process/Agent primitives exist and are `Ready`
   * Optionally annotate the Deployment with dependency info for observability

2. **Reconcile Deployment & Service**

   * Standard runtime for Next.js:

     * Image from `spec.image`
     * Config for Ameide TS SDK (base URLs, env, etc.)
   * Ensure `Service` and whichever ingress model you use (Gateway API / Ingress)

3. **Reconcile routing & auth integration**

   * Generate a **HTTPRoute/Ingress**:

     * Host & path from `spec.host` / `spec.pathPrefix`
     * Attach auth policy (e.g., OIDC JWT, scopes) based on `spec.auth.scopes`
   * Ensure appropriate Keycloak client/redirect URIs exist (either via integration with Keycloak operator or a sidecar job)

4. **Feature flags & config**

   * Create `ConfigMap` for feature flags and general UI config
   * Mount as env or file for Next.js

5. **Status**

   * `RoutingReady`, `AuthConfigured`, `Ready`

---

## 5. Cross‑cutting bits

### 5.1 Where operators live vs where CRDs live

* **Operators** live in the **core monorepo** (built and shipped as platform components).
* **CRDs (instances)** live in GitOps repos:

  * platform GitOps for shared primitives (`ameide-{env}`)
  * tenant GitOps for tenant primitives (`tenant-{id}-{env}-cust`)
* Argo CD applies *only* the CRs and operator Deployments; operators own children. 

### 5.2 Status surface for agents/CLI

Operators are the bridge between GitOps and the Ameide CLI / AI agents:

* CLI / agents read `status.conditions` on Domain/Process/Agent/UISurface CRs to understand:

  * Is a primitive ready?
  * Is an upgrade stuck on migrations?
  * Is a Process running the expected ProcessDefinition checksum?
* Operators are **write‑only** on `status`, not `spec`.

  * Spec is owned by Git (humans or agents via PRs).
  * This keeps the contract: Git → CR spec; operator → runtime + status.

---

## 6. Operator Publishing & Deployment

Operators are built in the **core monorepo** and deployed via **GitOps + ArgoCD**. GitOps repos hold only runtime config, never operator source code.

### 6.1 Core Repo Structure

```text
ameide-core/
  operators/
    domain-operator/
    process-operator/
    agent-operator/
    uisurface-operator/
  charts/
    ameide-operators/           # umbrella Helm chart for all 4 operators
      Chart.yaml
      values.yaml
      templates/...
```

### 6.2 CI Pipeline (Core Repo)

On merge/tag to `main`:

1. **Build images** for each operator:
   * `ghcr.io/ameide/domain-operator:<version>`
   * `ghcr.io/ameide/process-operator:<version>`
   * `ghcr.io/ameide/agent-operator:<version>`
   * `ghcr.io/ameide/uisurface-operator:<version>`

2. **Package Helm chart**:
   * `helm package charts/ameide-operators`
   * Chart `version:` matches operator release (e.g. `0.5.0`)

3. **Publish chart as OCI artifact**:
   * `ghcr.io/ameide/helm/ameide-operators:0.5.0`

> Operators are published as **artifacts** (container images + Helm chart). No operator manifests are copied into GitOps repos as code.

### 6.3 GitOps Repo Structure

```text
ameide-gitops/
  envs/
    prod/
      apps/
        platform-operators.yaml      # Argo Application for operators
      values/
        operators/
          prod-values.yaml           # env-specific overrides
    staging/
      apps/platform-operators.yaml
      values/operators/staging-values.yaml
```

### 6.4 ArgoCD Installation

Argo Application points to the **chart artifact**, not the source repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-operators
spec:
  source:
    repoURL: oci://ghcr.io/ameide/helm
    chart: ameide-operators
    targetRevision: 0.5.0
    helm:
      valueFiles:
        - values/operators/prod-values.yaml
  destination:
    namespace: ameide-system
```

ArgoCD:
* Pulls the `ameide-operators` chart version `0.5.0`
* Renders it with env-specific `values.yaml`
* Applies CRDs + operator Deployments into the cluster (infra wave)

### 6.5 Version Rollout Flow

```
1. Cut release in core repo: operators-v0.6.0
2. CI publishes images + chart to GHCR
3. PR in GitOps repo: targetRevision: 0.5.0 → 0.6.0
4. ArgoCD upgrades operator Deployments (infra wave)
5. Primitive CRs unchanged; operators reconcile them with new logic
```

### 6.6 Separation of Concerns

| Repo | Owns |
|------|------|
| **Core** (`ameide-core`) | Operator code, Helm chart templates, image builds |
| **GitOps** (`ameide-gitops`) | Which version to run, which cluster/namespace, env-specific values |

This matches the "design ≠ deploy ≠ runtime" principle:
* Core repo defines **what** operators do
* GitOps repo defines **where/which version** to run
* ArgoCD **pulls artifacts** and applies manifests
* Operators **reconcile** primitive CRs into workloads

---

## 7. Implementation Status

The four primitive operators are **implemented and CI/CD integrated** in the core monorepo:

### 7.1 Directory Structure

```text
operators/
  shared/                           # Shared Go module
    api/v1/conditions.go            # Condition vocabulary (Ready, WorkloadReady, DBReady, etc.)
    go.mod
  domain-operator/
    api/v1/
      domain_types.go               # CRD types with kubebuilder markers
      groupversion_info.go          # API registration
      zz_generated.deepcopy.go      # Generated DeepCopy methods
    internal/controller/
      domain_controller.go          # Reconciler (7-step pattern from 497)
    cmd/main.go                     # Controller manager entry point
    config/crd/bases/
      ameide.io_domains.yaml        # Generated CRD manifest
    Dockerfile.release              # Multi-stage build → distroless
    Dockerfile.dev
    Makefile                        # controller-gen targets
    go.mod, go.sum
  process-operator/                 # Same structure
  agent-operator/                   # Same structure
  uisurface-operator/               # Same structure
```

### 7.2 Shared Condition Vocabulary

All operators use shared condition types from `operators/shared/api/v1/conditions.go`:

| Condition | Meaning |
|-----------|---------|
| `Ready` | Top-level: all sub-conditions healthy |
| `WorkloadReady` | Deployment has available replicas |
| `DBReady` | Database connection/schema healthy |
| `MigrationSucceeded` | Schema migrations completed |
| `MigrationFailed` | Migration job failed |
| `Degraded` | Partial functionality |

### 7.3 Reconciler Pattern

Each operator follows the 7-step reconcile pattern from 497:

```go
func (r *DomainReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Load primary (Domain CR)
    // 2. Handle deletion (finalizers)
    // 3. Observe owned resources (Deployment, Service)
    // 4. Compute desired state
    // 5. Converge (CreateOrUpdate with OwnerReferences)
    // 6. Update status conditions
    // 7. Return (requeue if needed)
}
```

Key implementation details:
- **CreateOrUpdate pattern**: Uses `controllerutil.CreateOrUpdate()` for idempotent child resource management
- **OwnerReferences**: All child resources (Deployments, Services) have OwnerReferences pointing to the parent CR for automatic garbage collection
- **Condition computation**: Status conditions derived from child resource states

### 7.4 CI/CD Integration

Operators are built via the standard `cd-service-images.yml` workflow:

```yaml
# .github/workflows/cd-service-images.yml (matrix includes)
- name: domain-operator
  release_dockerfile: operators/domain-operator/Dockerfile.release
  dev_dockerfile: operators/domain-operator/Dockerfile.dev
  image: domain-operator
- name: process-operator
  # ...
- name: agent-operator
  # ...
- name: uisurface-operator
  # ...
```

Published to:
- `ghcr.io/ameideio/domain-operator:dev` (dev branch)
- `ghcr.io/ameideio/domain-operator:main` (main branch)
- `ghcr.io/ameideio/domain-operator:<version>` (tags)

### 7.5 Dockerfile Pattern

Operators use a minimal multi-stage Dockerfile:

```dockerfile
FROM golang:1.25-alpine AS builder
# Copy shared module (dependency)
COPY operators/shared ./operators/shared
# Copy operator module
COPY operators/domain-operator ./operators/domain-operator
# Build with CGO_ENABLED=0 for static binary
RUN go build -ldflags="-s -w" -o /out/manager ./cmd

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /out/manager /manager
USER 65532:65532
ENTRYPOINT ["/manager"]
```

Benefits:
- **Small images**: ~15MB distroless base
- **Security**: Non-root user, no shell
- **CI consistency**: Same workflow as services

### 7.6 Go Workspace Integration

Operators are part of the monorepo Go workspace:

```go
// go.work
use (
    ./operators/shared
    ./operators/domain-operator
    ./operators/process-operator
    ./operators/agent-operator
    ./operators/uisurface-operator
)
```

Each operator's `go.mod` uses a local replace directive for the shared module:

```go
replace github.com/ameideio/ameide/operators/shared => ../shared
```

---

## 8. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Go-level reconciler design, controller-runtime patterns, testing strategies |
| [498-domain-operator.md](498-domain-operator.md) | Domain operator development tracking |
| [499-process-operator.md](499-process-operator.md) | Process operator development tracking |
| [500-agent-operator.md](500-agent-operator.md) | Agent operator development tracking |
| [501-uisurface-operator.md](501-uisurface-operator.md) | UISurface operator development tracking |
| [477-primitive-stack.md](477-primitive-stack.md) | Primitive architecture context |
| [484-ameide-cli.md](484-ameide-cli.md) | CLI integration with operator status |

