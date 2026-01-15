Let’s make this concrete with a small but realistic setup:

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

* **App:** `nextjs-shop`
* **DB:** Postgres
* **Migration job:** runs Prisma / SQL migrations
* **Environments:** `dev → staging → prod`
* **Goal:**

  * Inside each environment: create DB → run migrations → deploy Next.js → expose via Ingress
  * Across environments: roll out change **dev first, then staging, then prod**, in a controlled way.

We’ll do it in a “more standard” Argo CD way:

* Use **sync waves (annotation)** for resource order *inside* each app.
* Use **RollingSync + labels** for cross-environment rollout.

---

## 1. Inside a single environment: one Argo CD Application

Imagine we have **three Argo CD Applications**, one per env:

* `nextjs-shop-dev`
* `nextjs-shop-staging`
* `nextjs-shop-prod`

Each Application points at the same repo but with env overlays (Kustomize/Helm/etc).
Focus on one (say `dev`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nextjs-shop-dev
  namespace: argocd
  labels:
    app: nextjs-shop
    env: dev
    rollout-phase: "10-dev"    # used later by RollingSync
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/nextjs-shop-infra
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: nextjs-shop-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Now, **inside `overlays/dev`** you have resources like:

* `postgres-deployment.yaml`
* `postgres-service.yaml`
* `migration-job.yaml`
* `nextjs-deployment.yaml`
* `nextjs-service.yaml`
* `ingress.yaml`

We’ll order them with **sync waves**:

### 1.1. Postgres first (wave 0)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  # normal Postgres deployment spec...
```

### 1.2. Postgres Service (still wave 0)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  # ClusterIP exposing postgres...
```

### 1.3. Data migration Job (wave 1)

This might run Prisma, Flyway, liquibase, or raw SQL.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nextjs-shop-migrate
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: your-registry/nextjs-shop-migrations:latest
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: url
          command: ["npm", "run", "prisma:migrate-deploy"]
```

### 1.4. Next.js deployment + service (wave 2)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-shop
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  template:
    spec:
      containers:
        - name: nextjs
          image: your-registry/nextjs-shop:sha-abcdef
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: url
```

Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nextjs-shop
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  # typical ClusterIP/LoadBalancer...
```

### 1.5. Ingress / Route (wave 3)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextjs-shop
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  rules:
    - host: dev.shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nextjs-shop
                port:
                  number: 80
```

**Result inside each environment:**

1. Wave 0 → Postgres Deployment + Service
2. Wave 1 → Migration Job
3. Wave 2 → Next.js Deployment + Service
4. Wave 3 → Ingress

That’s the **canonical** use of sync waves: resource-level ordering.

---

## 2. Across environments: ApplicationSet + RollingSync

Now we want to **roll changes dev → staging → prod** with control.

We’ll have an **ApplicationSet** that generates the three Applications and uses **RollingSync**:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nextjs-shop-all-envs
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            namespace: nextjs-shop-dev
            rolloutPhase: "10-dev"
          - env: staging
            namespace: nextjs-shop-staging
            rolloutPhase: "20-staging"
          - env: prod
            namespace: nextjs-shop-prod
            rolloutPhase: "30-prod"
  template:
    metadata:
      name: nextjs-shop-{{env}}
      labels:
        app: nextjs-shop
        env: "{{env}}"
        rollout-phase: "{{rolloutPhase}}"   # <- label used by RollingSync
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/nextjs-shop-infra
        targetRevision: main
        path: overlays/{{env}}
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy:
        # NB: with RollingSync, the ApplicationSet controller governs sync timing
        automated:
          prune: true
          selfHeal: true

  strategy:
    type: RollingSync
    rollingSync:
      maxUpdate: 1     # one environment at a time
      steps:
        - matchExpressions:
            - key: rollout-phase
              operator: In
              values: ["10-dev"]      # Step 1: dev
        - matchExpressions:
            - key: rollout-phase
              operator: In
              values: ["20-staging"]  # Step 2: staging
        - matchExpressions:
            - key: rollout-phase
              operator: In
              values: ["30-prod"]     # Step 3: prod
      deletionStrategy:
        # optional but nice: delete in reverse order (prod → staging → dev)
        deletionOrder: Reverse
```

### What happens when you push a change?

1. You commit a new Next.js image tag and new Prisma migration to `main`.
2. The **ApplicationSet** template sees the change, and Argo CD decides the three generated Applications need syncing.
3. Because the ApplicationSet uses **RollingSync**, it:

   * Looks at all Applications with label `app=nextjs-shop`.
   * Groups them by `rollout-phase`.
   * Applies them in the **step order**:

     1. `rollout-phase=10-dev` → sync `nextjs-shop-dev`
     2. Once dev is healthy, `rollout-phase=20-staging` → sync `nextjs-shop-staging`
     3. Once staging is healthy, `rollout-phase=30-prod` → sync `nextjs-shop-prod`
4. Within each Application, **sync waves take care of DB → migration → Next.js → Ingress**.

This gives you:

* **Safe DB migrations**: they always run **before** the new Next.js app is rolled out in each environment.
* **Safe env progression**: if migrations blow up in staging, prod is untouched.

---

## 3. How this handles a tricky migration

Say you’re adding a new column and backfilling data:

1. **Create migration** in your migrations repo/image:

   * Adds a nullable column.
   * Backfills data.
   * Does not yet rely on the new column in app code.

2. **Push migration + app change** to `main`:

   * The migration job in each env will run at **wave 1**.
   * Only after successful migration will Next.js deploy at **wave 2**.

3. **RollingSync** ensures this happens env-by-env:

   * dev first → verify manually if you want
   * then staging → maybe run smoke tests
   * then prod

If something goes wrong:

* Dev fails → rollout stops there; staging & prod Applications are not synced for that change.
* Staging fails → prod is not touched.
* The Next.js pods in an env are only updated **after** migrations succeed.

---

## 4. Mapping this back to your original question

You asked “what should we do to go more standard” and for an example with Next.js + Postgres + migrations moving across envs.

In this setup:

* **“Standard” sync waves (annotation)**

  * Used inside each Application: `argocd.argoproj.io/sync-wave: "0/1/2/3"`
  * Order: Postgres → migration Job → Next.js → Ingress.

* **“Standard” progressive sync (RollingSync)**

  * Used at ApplicationSet level with a semantic label: `rollout-phase: "10-dev" / "20-staging" / "30-prod"`.
  * Steps defined explicitly in the ApplicationSet, env by env.

* **No overloading of the term “wave” for labels.**

  * `wave` is now clearly the Argo CD **annotation** concept.
  * `rollout-phase` is your **cross-env group** concept.

If you’d like, next step I can:

* Sketch a small repo layout (folders for `overlays/dev`, `overlays/staging`, `overlays/prod`).
* Show how you’d wire a real migration tool (Prisma or Flyway) into that `Job` in a bit more detail.
