# 585: KEDA (Kubernetes Event-driven Autoscaling) — GitOps Installation

**Status:** Implemented (GitOps)  
**Audience:** Platform engineering, GitOps operators  
**Scope:** Document how KEDA is installed and verified in `ameide-gitops` following the repo’s standard cluster-scoped operator patterns (CRDs in wave 010, controllers in wave 020).

## 1) Why KEDA (and why not OLM/OperatorHub)

We deploy KEDA to support event-driven autoscaling primitives such as:

- `ScaledObject` (scale Deployments/StatefulSets based on metrics/events)
- `ScaledJob` (create Jobs from queue depth / event sources)

This repo standardizes on **ArgoCD-managed Helm/manifests**, not OLM. KEDA is therefore installed the same way as Strimzi/CNPG/External Secrets:

- CRDs first (cluster-scoped)
- Operator/controllers second (cluster-scoped)
- Workloads that use KEDA are environment-scoped

## 2) What is installed (GitOps shape)

### 2.1 Cluster CRDs (rolloutPhase 010)

- Component: `environments/_shared/components/cluster/crds/keda/component.yaml`
- Chart: `sources/charts/foundation/common/raw-manifests`
- Values: `sources/values/_shared/cluster/crds-keda.yaml`
- Manifest snapshot: `sources/charts/foundation/common/raw-manifests/files/keda-crds.yaml`

**Important:** `ScaledJob`’s CRD is large; applying it with client-side apply can fail due to the `kubectl.kubernetes.io/last-applied-configuration` annotation size limit. The CRD component uses **Server-Side Apply** (`ServerSideApply=true`) specifically to avoid that failure mode.

### 2.2 Cluster operator/controllers (rolloutPhase 020)

- Component: `environments/_shared/components/cluster/operators/keda/component.yaml`
- Chart: `sources/charts/third_party/kedacore/keda/2.18.2`
- Shared values: `sources/values/_shared/cluster/keda.yaml`
- Local overrides: `sources/values/cluster/local/keda.yaml` (probe timeout hardening for k3d)
- Namespace: `keda-system`

KEDA installs:

- `Deployment/keda-operator`
- `Deployment/keda-operator-metrics-apiserver`
- `Deployment/keda-admission-webhooks`
- `APIService/v1beta1.external.metrics.k8s.io`
- `ValidatingWebhookConfiguration/keda-admission`

## 3) Version pinning and upgrades

KEDA is pinned via the third-party chart lockfile:

- `sources/charts/third_party/charts.lock.yaml` (entry: `kedacore/keda` `2.18.2`)

Upgrade procedure (high-level):

1. Bump the pinned version in `sources/charts/third_party/charts.lock.yaml`.
2. Vendor the updated chart into `sources/charts/third_party/…` using `scripts/vendor-charts.sh`.
3. Regenerate the CRD snapshot in `sources/charts/foundation/common/raw-manifests/files/keda-crds.yaml` from the new chart’s CRD templates (keep SSA in place).
4. Update `environments/_shared/components/cluster/operators/keda/component.yaml` chart path to the new vendored version.

## 4) How to verify (local)

From a kube context pointing at `ameide-local`:

1. ArgoCD Applications exist and are healthy:
   - `kubectl get applications -n argocd | rg 'cluster-(crds-keda|keda)'`
2. KEDA pods are running:
   - `kubectl get pods -n keda-system`
3. CRDs exist:
   - `kubectl get crd scaledobjects.keda.sh scaledjobs.keda.sh triggerauthentications.keda.sh clustertriggerauthentications.keda.sh`
4. External metrics API is served:
   - `kubectl get apiservice v1beta1.external.metrics.k8s.io`

## 5) Follow-ups (not yet implemented here)

These follow-ups are now implemented in `ameide-gitops`:

- A minimal `ScaledObject` example (CPU-based autoscaling) is included for reference (disabled by default; local-only component).
- WorkRequests `ScaledJob` scaffolding now points at a dedicated **executor image** (not the devcontainer).
- A **cluster-scoped** KEDA smoke check runs early (cluster rolloutPhase `030`), before environment apps.

### Current in-repo ScaledJob example (implemented)

For 527’s execution substrate, `ameide-gitops` already provisions:

- WorkRequests queue topics: `sources/values/_shared/data/data-kafka-workrequests-topics.yaml` (enabled in `local` + `dev`).
- WorkRequests `ScaledJob` objects: `sources/values/_shared/apps/workrequests-runner.yaml` (enabled in `local` + `dev`).
- GitOps validation: `local-data-data-plane-smoke` checks both KafkaTopics and ScaledJobs when enabled (see `sources/values/_shared/data/data-data-plane-smoke.yaml`).

Implementation notes:

- The ScaledJobs run `ghcr.io/ameideio/primitive-integration-transformation-work-executor:<tag>` (a dedicated integration primitive image).
- Local/dev set `scaledJobs.maxReplicaCount: 0` by default to avoid continuous Job churn until a real producer emits valid execution intents (`WorkExecutionRequested`) onto the execution queue topics.

### ScaledObject example (CPU trigger; local-only)

- Component: `environments/local/components/apps/examples/keda-scaledobject-demo/component.yaml`
- Values: `sources/values/_shared/apps/keda-scaledobject-demo.yaml` + `sources/values/env/local/apps/keda-scaledobject-demo.yaml`
- Status: disabled by default (`enabled: false`) so it doesn’t pull any new images or affect clusters unless explicitly enabled.

### Cluster-scoped KEDA smoke (early failure signal)

- Component: `environments/_shared/components/cluster/smokes/keda-smoke/component.yaml`
- Values: `sources/values/_shared/cluster/keda-smoke.yaml`
- Checks: KEDA deployments, KEDA CRDs, and external metrics `APIService` availability.
