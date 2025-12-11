# 497 – Operator Implementation Patterns

**Status:** Active
**Audience:** Go developers implementing Domain/Process/Agent/UISurface operators
**Scope:** controller-runtime patterns, reconciler design, testing strategies

> **Related**: [495-ameide-operators.md](495-ameide-operators.md) for CRD shapes and responsibilities

---

## 1. Core Mental Model: Control Loop & Reconciler

Kubernetes' own docs and Kubebuilder/Cluster API all push the same mental model:

> A controller is a *control loop* that:
>
> 1. Watches objects (desired state),
> 2. Observes the actual state (cluster + external systems),
> 3. Repeatedly *reconciles* the two until they match.

In controller-runtime terms, a **Reconciler** gets a key (`NamespacedName`) and must:

1. Load the root object (your CR)
2. Observe children / external state
3. Compute the desired children
4. Create/patch/delete children to converge
5. Update **status** and decide whether to requeue

**Design pattern for Ameide operators:**

* **One root Kind per controller**: Domain operator reconciles `Domain`; Process operator reconciles `Process`, etc. It *may* touch other kinds (Deployments, Jobs, CNPG CRs) but has exactly one "root" CRD.
* **Level-based, not event-based**: Reconcile compares desired vs actual state, not "on add do X, on delete do Y".

This matches our primitives: each operator is the control loop for a single primitive kind.

---

## 2. Reconciler Design Patterns

### 2.1 Idempotent Reconcilers

Every best-practice doc hammers this: **reconcile must be idempotent**.

* You must be able to run `Reconcile()` many times with the same input and end up in the same state
* This is *exactly* the same reasoning as event-driven idempotent consumers (outbox/inbox pattern in [472 §3.3.2](472-ameide-information-application.md))

For Ameide primitives:

* Domain operator reconciling `Domain/orders` should be safe to call on **every** `Domain` event, resync, or restart
* Process operator reconciling `Process/l2o` similarly

### 2.2 Small, Composable Reconcile Functions

Red Hat & Operator SDK teams recommend: **avoid overstuffed reconcile functions**.

Pattern:

* Reconcile should orchestrate helper functions:
  * `reconcileDB()`
  * `reconcileDeployment()`
  * `reconcileServiceMonitor()`
* Each helper:
  * Takes the CR + a `context.Context` and a typed client
  * Ensures a single child resource is correct
  * Is individually testable

This mirrors our "shape, not meaning" rule: reconcile is glue; domain semantics live in the app image, not in the operator.

### 2.3 Declarative Reconciliation Recipe

Kubebuilder & CAPI show a recurring pattern in controller implementations:

```text
Reconcile:
  1. Fetch CR
  2. Handle deletion (finalizer)
  3. Default + validate spec
  4. Reconcile dependencies (e.g. DB, Secrets)
  5. Reconcile main workload (Deployment/Job/etc.)
  6. Update status (conditions, observedGeneration)
  7. Requeue if needed (e.g. for polling)
```

You can treat this as a **template** for all four operators, with step 4/5 varying by primitive kind.

---

## 3. API & CRD Design Patterns

### 3.1 Declarative, Not Imperative APIs

Google & others explicitly say: **use declarative CRD specs for operators, not imperative APIs.**

* Users/agents say *"I want this Domain/Process/Agent/UISurface to look like X"*
* Operator figures out *how* to get there

This dovetails perfectly with our "business writes are never imperative" stance – at infra level, no "scale this", "restart that"; only "here is my desired domain/process/agent surface, make it so".

### 3.2 Spec vs Status Separation

Pattern from Kubernetes API conventions:

* **`spec`**: desired state, set by users/GitOps
* **`status`**: observed state, set by controllers only

Good controller design:

* Treat `spec` as immutable input (except defaulting)
* Never mutate `spec` in reconcile; only patch `status`
* Encode readiness/health with **Conditions**, not just booleans

For Ameide:

* Domain CRD:
  * `spec.image`, `spec.db`, `spec.rollout…`
  * `status.conditions`, `status.db.migrationVersion`, etc.
* Process CRD:
  * `spec.definitionRef`, `spec.temporal…`
  * `status.definition.checksum`, `status.workersReady`

That's what our CLI will read when deciding if primitives are "Ready".

### 3.3 One Operator per Primitive Kind

Google & Red Hat guidance:

> "Develop one Operator per application" and "compartmentalize features via multiple controllers".

Applied to Ameide:

* One operator **per primitive kind** (Domain, Process, Agent, UISurface) – which we already have
* If a CRD grows too complex, introduce **multiple controllers** per CRD focusing on different aspects (e.g. `-status-controller`, `-autoscaler-controller`) instead of one mega reconciler

---

## 4. Event Pipeline Patterns: Informers, Caches & Queues

If you're using controller-runtime this is mostly abstracted away, but it's worth understanding the canonical pattern:

* **Informers** watch API server for changes and maintain a **local cache**
* **Listers** let you query the cache instead of pounding the API
* **Workqueue** holds objects to reconcile; often a rate-limiting queue

### 4.1 Never Do Work in Event Handlers

* Handler should just enqueue keys for reconcile
* All logic lives in `Reconcile`

### 4.2 Use Predicates/Filters

* Filter out events that don't change `spec` (only `status`) where appropriate
* Filter out updates where no meaningful fields changed
* This avoids thundering herds after controller restart

### 4.3 Always Use the Cache

* Wait for `cache.WaitForCacheSync` before reconciling
* Never do direct GETs to API server in hot paths

controller-runtime bakes these in, but you should still explicitly choose:

* What events should enqueue
* What rate limits should apply
* What max concurrency per controller is acceptable

---

## 5. Resource Relationship Patterns

### 5.1 OwnerReferences & Garbage Collection

Pattern: for "child" resources (Deployments, Jobs, Secrets) the operator creates, always set `OwnerReferences` to the root CR.

* When `Domain/orders` is deleted, its Deployment/Service/Job etc. are automatically GC'd
* Operator doesn't need to explicitly delete everything (except cases needing custom cleanup)

### 5.2 Watch Secondaries, Reconcile Primaries

Cluster API & Kubebuilder docs recommend: one "root" controller per CRD, but it can also **watch other types** and map their events back to the root.

For example:

* Domain operator:
  * Root = `Domain`
  * Watches `Deployment`, `Job`, `Secret` with an `EnqueueRequestForOwner` handler
* Process operator:
  * Root = `Process`
  * Watches `Deployment` (Temporal workers), maybe `CronJob`

This keeps your mental model: **you always reconcile in terms of the root object**, even when a child changes.

### 5.3 Don't Reconcile All Objects at Once

Operator SDK maintainers advise against "loop all CRs in one reconcile" as a general pattern; it doesn't scale and gets weird with retries.

* Default: reconcile one CR per key
* Only use "global controllers" if the problem is inherently global (cluster-wide scheduling, quotas, etc.), and even then be careful

For Ameide, most primitives are naturally per-object; you shouldn't need "reconcile everything" patterns.

---

## 6. Consistency, Conflicts & Retries

### 6.1 Optimistic Concurrency

Kubebuilder maintainers recommend handling "resource changed while you were reconciling" via **optimistic concurrency**:

* Use `resourceVersion` semantics that `client-go` gives you
* If an update fails due to conflict, requeue and re-read the resource

Pattern:

* Read CR
* Compute status
* Patch status with `client.Status().Patch(ctx, obj, patch)`
* Retry on conflict

### 6.2 Requeue Semantics

Use return values of `Reconcile()` carefully:

| Return | Effect |
|--------|--------|
| `return ctrl.Result{}, nil` | No immediate requeue; only requeue on next event/resync |
| `return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil` | Periodic polling (e.g., check external system) |
| `return ctrl.Result{}, err` | Requeue with backoff |

Pattern: use `RequeueAfter` only when you *must* poll; otherwise rely on events (informer updates) and a standard resync period.

---

## 7. Observability & Testing

### 7.1 Structured Logging & Metrics

Patterns from multiple guides:

**Logging per reconcile:**
* Key (namespace/name)
* What you're doing (creating/updating/deleting child)
* Errors with enough context

**Emit metrics:**
* Reconciliations count
* Errors count
* Queue length
* Reconcile duration

For Ameide: these metrics are how CLI / SREs will see "Domain operator is flapping" or "Process operator is saturated".

### 7.2 Unit & Integration Tests

Kubebuilder book recommends:

* **Unit tests** for reconcile logic using fake client
* **envtest** for more integrated tests against a local API server
* Don't bother trying to test informers/queues directly; controller-runtime abstracts them

This aligns with our CLI's TDD + `verify` story: tests around **primitive correctness** + operator behaviour can run in mock or envtest/cluster mode.

---

## 8. Ameide Operator Checklist

When implementing Domain/Process/Agent/UISurface operators, these patterns are **non-negotiable**:

| # | Pattern | Reference |
|---|---------|-----------|
| 1 | **One root kind per controller** – each operator reconciles a single CRD kind | §1, §3.3 |
| 2 | **Idempotent, level-based Reconcile** – compare desired vs actual, don't assume single execution | §2.1 |
| 3 | **Spec vs Status** – spec is user/GitOps; status is controller with Conditions and observedGeneration | §3.2 |
| 4 | **Declarative CRDs** – no imperative "do X now" fields; only target state | §3.1 |
| 5 | **OwnerReferences** on all children & watch secondaries via owner to reconcile roots | §5.1, §5.2 |
| 6 | **Informer + workqueue pattern** – never do work in event handlers; use predicates to reduce noise | §4 |
| 7 | **Optimistic concurrency & retry** – handle conflicts, requeue with backoff on error | §6 |
| 8 | **Small, composable reconcile functions** – orchestrate helpers; avoid 500-line reconcilers | §2.2 |
| 9 | **Clear deletion & finalizers** – explicit cleanup for external resources; remove finalizers when done | §2.3 |
| 10 | **Observability & tests** – structured logs, metrics, envtest integration; wired into CLI's `verify` | §7 |

---

## 9. Why Operators Are NOT CLI-Scaffolded

Unlike primitives (Domain/Process/Agent/UISurface services), operators are **not scaffolded** via `ameide primitive scaffold`. This is a deliberate design decision:

### 9.1 Rationale

| Factor | Primitives | Operators |
|--------|------------|-----------|
| **Count** | Unbounded (many domains, processes, agents) | Fixed at 4 (Domain, Process, Agent, UISurface) |
| **Variation** | High structural similarity | Each operator has unique reconciliation logic |
| **Boilerplate ratio** | ~80% mechanical structure | ~20% structure, ~80% domain-specific logic |
| **Creation frequency** | Often (new features = new primitives) | Rarely (operators are platform infrastructure) |
| **Author** | AI agents + developers | Platform team only |

### 9.2 What Makes Operators "Special"

1. **Unique reconciliation logic**: Domain operator manages DB schemas + migrations; Process operator manages Temporal workers + ProcessDefinitions; Agent operator manages LLM secrets + tool grants; UISurface operator manages routing + Keycloak clients. No common template covers this.

2. **Deep platform coupling**: Operators integrate with CNPG, Temporal, Keycloak, Gateway API—each integration is bespoke.

3. **Security sensitivity**: Operators run with elevated RBAC. Code review is mandatory; no auto-generation.

4. **Fixed count**: We will only ever have 4 primitive operators. The cost of writing them manually is bounded and one-time.

5. **Reference implementation model**: This document (§10) provides a concrete Domain operator skeleton. The other three operators clone this pattern with their own `reconcile*` helpers.

### 9.3 What Kubebuilder Does (and Why We Don't)

Kubebuilder scaffolds:
- CRD types (`api/v1/domain_types.go`)
- Controller skeleton (`controllers/domain_controller.go`)
- RBAC markers
- Test scaffolds

We **do use Kubebuilder patterns** but **don't use its scaffolding** because:
- Kubebuilder markers (`// +kubebuilder:...`) add magic we'd rather make explicit
- Scaffolded controllers still need 80%+ custom logic
- Our operators share a condition library and status contract that Kubebuilder doesn't know about

Instead, we provide this document as the "template" and expect implementers to copy/adapt the reference implementation.

---

## 10. Reference Implementation: Domain Operator

This section provides a concrete, copy-paste-ready skeleton for the Domain operator. Process/Agent/UISurface operators follow the same structure with different `reconcile*` helpers.

### 10.1 CRD Types (Go)

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
type Domain struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   DomainSpec   `json:"spec,omitempty"`
    Status DomainStatus `json:"status,omitempty"`
}

type DomainSpec struct {
    Image     string                       `json:"image"`
    Version   string                       `json:"version,omitempty"`
    DB        DomainDBSpec                 `json:"db"`
    Env       []corev1.EnvVar              `json:"env,omitempty"`
    Resources corev1.ResourceRequirements  `json:"resources,omitempty"`
    Observability DomainObservabilitySpec  `json:"observability,omitempty"`
    Security      DomainSecuritySpec       `json:"security,omitempty"`
    Rollout       DomainRolloutSpec        `json:"rollout,omitempty"`
}

type DomainDBSpec struct {
    ClusterRef        string `json:"clusterRef"`
    Schema            string `json:"schema"`
    MigrationJobImage string `json:"migrationJobImage"`
}

type DomainStatus struct {
    Conditions         []metav1.Condition `json:"conditions,omitempty"`
    ObservedGeneration int64              `json:"observedGeneration,omitempty"`
    DBMigrationVersion string             `json:"dbMigrationVersion,omitempty"`
    ReadyReplicas      int32              `json:"readyReplicas,omitempty"`
}

// +kubebuilder:object:root=true
type DomainList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Domain `json:"items"`
}
```

**Condition types** (shared across all operators):
- `Ready` – overall readiness
- `Progressing` – rollout in progress
- `Degraded` – partial failure
- `DBReady` – database schema ready (Domain only)
- `MigrationSucceeded` / `MigrationFailed` – migration job status
- `WorkloadReady` – Deployment available

### 10.2 Reconciler Struct

```go
type DomainReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    Recorder record.EventRecorder
    Config   OperatorConfig
}

type OperatorConfig struct {
    DefaultMigrationImage string
    DefaultResources      corev1.ResourceRequirements
    NamespaceSelector     labels.Selector
}
```

### 10.3 Main Reconcile Flow

```go
func (r *DomainReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx).WithValues("domain", req.NamespacedName)

    // 1. Fetch Domain
    var domain amv1.Domain
    if err := r.Get(ctx, req.NamespacedName, &domain); err != nil {
        if apierrors.IsNotFound(err) {
            return ctrl.Result{}, nil // deleted
        }
        return ctrl.Result{}, err
    }

    // 2. Handle deletion / finalizer
    if !domain.ObjectMeta.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, log, &domain)
    }

    // 3. Ensure finalizer present
    if added := controllerutil.AddFinalizer(&domain, domainFinalizerName); added {
        if err := r.Update(ctx, &domain); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{}, nil // requeue
    }

    // 4. Defaulting & validation
    if err := validateDomainSpec(&domain.Spec); err != nil {
        _ = r.setCondition(ctx, &domain, metav1.Condition{
            Type:    "Degraded",
            Status:  metav1.ConditionTrue,
            Reason:  "InvalidSpec",
            Message: err.Error(),
        })
        return ctrl.Result{}, nil
    }

    // 5. Reconcile dependencies (DB, Secrets)
    if res, err := r.reconcileDB(ctx, log, &domain); err != nil || !res.IsZero() {
        return res, err
    }

    // 6. Reconcile workload (Deployment, Service, HPA, NetworkPolicy, ServiceMonitor)
    if res, err := r.reconcileWorkload(ctx, log, &domain); err != nil || !res.IsZero() {
        return res, err
    }

    // 7. Update status
    if err := r.updateStatus(ctx, log, &domain); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```

### 10.4 Deletion & Finalizer

```go
func (r *DomainReconciler) reconcileDelete(
    ctx context.Context,
    log logr.Logger,
    domain *amv1.Domain,
) (ctrl.Result, error) {
    if controllerutil.ContainsFinalizer(domain, domainFinalizerName) {
        // Clean up external resources (e.g., drop DB schema if policy allows)
        if err := r.cleanupExternalResources(ctx, log, domain); err != nil {
            log.Error(err, "failed cleaning up external resources")
            return ctrl.Result{RequeueAfter: 30 * time.Second}, err
        }

        controllerutil.RemoveFinalizer(domain, domainFinalizerName)
        if err := r.Update(ctx, domain); err != nil {
            return ctrl.Result{}, err
        }
    }
    // K8s GC handles children via OwnerReferences
    return ctrl.Result{}, nil
}
```

### 10.5 Reconcile DB (Dependency)

```go
func (r *DomainReconciler) reconcileDB(
    ctx context.Context,
    log logr.Logger,
    domain *amv1.Domain,
) (ctrl.Result, error) {
    job := constructMigrationJob(domain)

    var existing batchv1.Job
    err := r.Get(ctx, types.NamespacedName{
        Namespace: domain.Namespace,
        Name:      job.Name,
    }, &existing)

    if apierrors.IsNotFound(err) {
        // Create migration Job
        if err := ctrl.SetControllerReference(domain, &job, r.Scheme); err != nil {
            return ctrl.Result{}, err
        }
        if err := r.Create(ctx, &job); err != nil {
            return ctrl.Result{}, err
        }
        _ = r.setCondition(ctx, domain, metav1.Condition{
            Type:   "DBReady",
            Status: metav1.ConditionFalse,
            Reason: "MigrationJobCreated",
        })
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    } else if err != nil {
        return ctrl.Result{}, err
    }

    // Check Job status
    if isJobFailed(&existing) {
        _ = r.setCondition(ctx, domain, metav1.Condition{
            Type:    "MigrationFailed",
            Status:  metav1.ConditionTrue,
            Reason:  "JobFailed",
            Message: "see Job logs",
        })
        return ctrl.Result{}, nil
    }
    if !isJobComplete(&existing) {
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }

    // Migration succeeded
    _ = r.setCondition(ctx, domain, metav1.Condition{
        Type:   "DBReady",
        Status: metav1.ConditionTrue,
        Reason: "MigrationSucceeded",
    })
    return ctrl.Result{}, nil
}
```

### 10.6 Reconcile Workload (Deployment, Service)

```go
func (r *DomainReconciler) reconcileWorkload(
    ctx context.Context,
    log logr.Logger,
    domain *amv1.Domain,
) (ctrl.Result, error) {
    // Deployment
    deploy := &appsv1.Deployment{ObjectMeta: metav1.ObjectMeta{
        Name:      deploymentName(domain),
        Namespace: domain.Namespace,
    }}
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, deploy, func() error {
        if err := ctrl.SetControllerReference(domain, deploy, r.Scheme); err != nil {
            return err
        }
        mutateDeploymentFromDomain(deploy, domain)
        return nil
    })
    if err != nil {
        return ctrl.Result{}, err
    }

    // Service
    svc := &corev1.Service{ObjectMeta: metav1.ObjectMeta{
        Name:      serviceName(domain),
        Namespace: domain.Namespace,
    }}
    _, err = controllerutil.CreateOrUpdate(ctx, r.Client, svc, func() error {
        if err := ctrl.SetControllerReference(domain, svc, r.Scheme); err != nil {
            return err
        }
        mutateServiceFromDomain(svc, domain)
        return nil
    })
    if err != nil {
        return ctrl.Result{}, err
    }

    // HPA, NetworkPolicy, ServiceMonitor similarly...
    return ctrl.Result{}, nil
}

func mutateDeploymentFromDomain(deploy *appsv1.Deployment, domain *amv1.Domain) {
    replicas := int32(2)
    deploy.Spec.Replicas = &replicas
    deploy.Spec.Selector = &metav1.LabelSelector{
        MatchLabels: map[string]string{"app": domain.Name},
    }
    deploy.Spec.Template.ObjectMeta.Labels = map[string]string{
        "app":                   domain.Name,
        "ameide.io/primitive":   "domain",
        "ameide.io/domain-name": domain.Name,
    }
    deploy.Spec.Template.Spec = corev1.PodSpec{
        ServiceAccountName: domain.Spec.Security.ServiceAccountName,
        Containers: []corev1.Container{{
            Name:      "app",
            Image:     domain.Spec.Image,
            Env:       domain.Spec.Env,
            Resources: domain.Spec.Resources,
        }},
    }
}
```

### 10.7 Status Update

```go
func (r *DomainReconciler) updateStatus(
    ctx context.Context,
    log logr.Logger,
    domain *amv1.Domain,
) error {
    var deploy appsv1.Deployment
    _ = r.Get(ctx, types.NamespacedName{
        Namespace: domain.Namespace,
        Name:      deploymentName(domain),
    }, &deploy)

    // Derive WorkloadReady condition
    ready := deploy.Status.ReadyReplicas >= 1
    setCondition(&domain.Status.Conditions, metav1.Condition{
        Type:   "WorkloadReady",
        Status: boolToConditionStatus(ready),
        Reason: ifThenElse(ready, "DeploymentAvailable", "DeploymentNotReady"),
    })

    // Compute overall Ready = DBReady && WorkloadReady && !MigrationFailed
    setCondition(&domain.Status.Conditions, computeDomainReady(domain.Status.Conditions))

    domain.Status.ObservedGeneration = domain.Generation
    domain.Status.ReadyReplicas = deploy.Status.ReadyReplicas

    return r.Status().Update(ctx, domain)
}
```

### 10.8 Controller Setup

```go
func (r *DomainReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&amv1.Domain{}).
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

### 10.9 Adapting for Other Operators

| Operator | Replace `reconcileDB` with | Replace `reconcileWorkload` with |
|----------|---------------------------|----------------------------------|
| **Process** | `reconcileDefinition()` (fetch ProcessDefinition, create ConfigMap) | `reconcileTemporalWorkers()` (worker Deployment, task queue config) |
| **Agent** | `reconcileSecrets()` (LLM API keys, tool credentials) | `reconcileAgentRuntime()` (agent Deployment, tool grants ConfigMap) |
| **UISurface** | `reconcileAuth()` (Keycloak client, redirect URIs) | `reconcileNextApp()` (Deployment, HTTPRoute, feature flags) |

The 7-step Reconcile flow (§10.3) remains identical; only the helper functions change.

---

## 11. References

* [Kubernetes Controllers Documentation](https://kubernetes.io/docs/concepts/architecture/controller/)
* [Cluster API: Controllers and Reconciliation](https://cluster-api.sigs.k8s.io/developer/providers/getting-started/controllers-and-reconciliation)
* [Operator SDK: Common Recommendations](https://sdk.operatorframework.io/docs/best-practices/common-recommendation/)
* [Kubebuilder Book: Implementing a Controller](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation)
* [Google Cloud: Best Practices for Building Kubernetes Operators](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps)
* [Red Hat: 7 Best Practices for Writing Kubernetes Operators](https://www.redhat.com/en/blog/7-best-practices-for-writing-kubernetes-operators-an-sre-perspective)

---

## 12. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [495-ameide-operators.md](495-ameide-operators.md) | CRD shapes, operator responsibilities, publishing |
| [498-domain-operator.md](498-domain-operator.md) | Domain operator development tracking |
| [499-process-operator.md](499-process-operator.md) | Process operator development tracking |
| [500-agent-operator.md](500-agent-operator.md) | Agent operator development tracking |
| [501-uisurface-operator.md](501-uisurface-operator.md) | UISurface operator development tracking |
| [477-primitive-stack.md](477-primitive-stack.md) | Design ≠ Deploy ≠ Runtime principle |
| [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Operator deployment architecture |
| [484-ameide-cli.md](484-ameide-cli.md) | CLI verify integration with operator status |
| [472-ameide-information-application.md](472-ameide-information-application.md) | Idempotency patterns (§3.3.2) |
