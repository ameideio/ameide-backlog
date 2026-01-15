# 681: Argo CD UI performance (large fleets)

**Problem**: Argo CD UI feels sluggish with ~400 Applications.

This doc captures what we observed in `ameide-gitops` and the configuration changes that are most likely to improve perceived UI responsiveness **without** changing GitOps posture (all changes via Git/Helm values).

---

## 1) Why the UI can be slow at ~400 apps (even when “healthy”)

Two constraints matter:

1. The UI is driven by `argocd-server` API responses (not the controller directly).
2. The Applications list experience tends to scale with “how many Applications you can see” because the UI frequently retrieves and processes a large list of app summaries.

So UI sluggishness is usually dominated by:

- `argocd-server` CPU and response serialization (and compression, if enabled)
- Redis/cache response time (server reads app state)
- “churn” (many app updates) causing frequent list updates/re-renders
- browser-side render cost when the list is large

---

## 2) Current repo posture (what’s true today)

### 2.1 Structural app count is expected

The fleet size is not accidental; it’s generated:

- `argocd/applicationsets/ameide.yaml` uses a matrix generator (`config/clusters/{type}.yaml` × `environments/_shared/**/component.yaml` globs).
- `config/clusters/azure.yaml` defines 3 environments (dev/staging/production).

This naturally lands in the “few hundred Applications” range before previews.

### 2.2 Managed Argo CD Helm values already include cmd-params

Managed values are in `sources/values/common/argocd.yaml` and already set a small `configs.params` block (cmd-params):

- `configs.params.controller.diff.server.side: true`
- `configs.params.controller.app.state.cache.expiration: 6h`
- `configs.params.server.insecure: "true"`

So the situation is not “managed envs rely on defaults”; it’s “managed envs have a few params, but not the ones that target UI throughput/churn”.

### 2.3 Tracking method is already annotation-based

`sources/values/common/argocd.yaml` sets:

- `configs.cm.application.resourceTrackingMethod: annotation`

This matches Argo CD defaults/typical guidance. Switching to `annotation+label` does **not** change tracking semantics (still annotation-tracked) and should not be expected to materially improve UI performance.

---

## 3) Recommended changes (ordered)

### Priority 1: Make the Applications list response cheaper (server throughput)

1. **Enable gzip explicitly** via cmd-params:
   - `configs.params.server.enable.gzip: "true"`

2. **Scale `argocd-server`**:
   - increase `server.replicas` (e.g. 3)
   - increase server CPU limit (serialization/compression is CPU-heavy)

Rationale: the UI bottleneck is often the API response path.

#### Concrete values sketch (managed default)

Put these in `sources/values/common/argocd.yaml` under `server:` / `configs.params:` (numbers are starting points; tune to cluster size and user concurrency):

```yaml
server:
  replicas: 3
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 1Gi

configs:
  params:
    server.enable.gzip: "true"
```

### Priority 2: Reduce churn and smooth processing bursts (controller cache)

1. **Set cluster-cache batching explicitly** (even if defaults are “on”, make it intentional and tuneable):
   - `controller.cluster.cache.batch.events.processing: "true"`
   - `controller.cluster.cache.events.processing.interval: "250ms"` (tune based on observed jitter vs staleness)

2. **Be careful overriding ignoreResourceUpdates**:
   - The Helm chart ships default `resource.customizations.ignoreResourceUpdates.*` rules.
   - Overriding `resource.customizations.ignoreResourceUpdates.argoproj.io_Application` replaces the default list; if you need to add items, copy the chart defaults and extend them rather than replacing with a minimal set.

#### Concrete values sketch (batching + optional processors)

```yaml
configs:
  params:
    controller.cluster.cache.batch.events.processing: "true"
    controller.cluster.cache.events.processing.interval: "250ms"
    # Optional: only after increasing controller CPU limits/requests.
    # controller.status.processors: "30"
    # controller.operation.processors: "15"
```

### Priority 3: Prevent repo-server request storms (diff/refresh experience)

Add concurrency guardrails:

- `configs.params.reposerver.parallelism.limit: "<n>"`

This protects the system when many Applications refresh/diff simultaneously.

#### Concrete values sketch (repo-server parallelism)

```yaml
configs:
  params:
    reposerver.parallelism.limit: "5"
```

### Priority 4: Re-evaluate expensive comparisons if CPU is tight

Today we force server-side diff:

- `configs.params.controller.diff.server.side: true`

If controller CPU becomes the limiting factor, consider making this conditional (env/cluster override) and validating whether turning it off improves UI “freshness” under load.

---

## 4) Rollout plan (GitOps-safe)

1. Apply changes via Helm values in `sources/values/common/argocd.yaml` (and env overrides if needed).
2. Roll out `argocd-server` scaling + gzip first (low risk, directly targets UI).
3. Add cache batching + repo-server parallelism next (reduce jitter and protect against spikes).
4. Only then consider deeper behavior changes (diff mode, extra ignore rules, etc.).

---

## 5) What to measure

- `argocd-server` CPU during UI load and during `/api/v1/applications` list calls
- Redis CPU/latency
- controller CPU and reconcile queue delays
- repo-server CPU/memory and manifest generation latency
- UI “time to first usable list” as experienced by users
