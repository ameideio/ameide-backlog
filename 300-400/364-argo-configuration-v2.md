# 364 – Argo Configuration (Hierarchy v2)

## Context

The first GitOps pass mirrored Helmfile “layers” (10/12/15/…​) directly into a single `argocd-apps.yaml`. After splitting Helmfile selectors to get better signal (e.g., `vault-core`, `secrets-temporal`), that monolith became impossible to review or own. We also blurred the line between:

1. **Shared platform primitives** (namespaces, CRDs, vault, gateways, observability core).
2. **Standard services** we run but don’t build (Keycloak, Temporal, Prometheus, etc.).
3. **First-party applications** under `/services`.

This rewrite defines a clean hierarchy and ownership model so each domain can evolve independently while Argo still enforces ordering.

## Target Layout

```
infra/kubernetes/gitops/argocd/
├── shared/                # cluster-scoped primitives
│   ├── bootstrap/
│   │   ├── namespace.yaml
│   │   └── smoke.yaml
│   ├── crds/
│   │   ├── gateway.yaml
│   │   └── observability.yaml
│   ├── secrets-core/
│   │   ├── vault-core.yaml
│   │   ├── cert-manager.yaml
│   │   └── external-secrets.yaml
│   ├── gateways/
│   └── observability-core/
├── services/              # third‑party workloads deployed by the platform team
│   ├── keycloak.yaml
│   ├── temporal.yaml
│   └── prometheus.yaml
├── applications/          # first‑party apps from /services/*
│   ├── platform.yaml
│   ├── repository.yaml
│   └── agents.yaml
└── applicationsets/
    ├── services.yaml      # optional generators (e.g., secret bundles per service)
    └── applications.yaml
```

*Guiding principles*
- **Shared = foundational**: only keep components every workload depends on (bootstrap, CRDs, vault/external-secrets, gateway config, logging/metrics base).  
- **Services = catalogued third parties**: anything under `infra/kubernetes/environments/<env>/platform/services/` gets a one-to-one Argo Application here.  
- **Applications = `/services` directory**: each repository service gets its own Application; ApplicationSets can template the common bits.  
- **Hooks stay local**: readiness checks (CRDs present, secret templates populated) belong to the service/application that needs them, not the shared layer.

## Application Spec Template

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shared-crds-gateway
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-93"
spec:
  project: ameide
  source:
    repoURL: https://github.com/ameideio/ameide.git
    targetRevision: main
    path: infra/kubernetes
    plugin:
      name: helmfile
      env:
        - name: HELMFILE_FILE
          value: helmfiles/12-crds.yaml
        - name: HELMFILE_ENV
          value: <env>
        - name: HELMFILE_SELECTOR
          value: component=gateway-api-crds
  destination:
    server: https://kubernetes.default.svc
    namespace: ameide
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Naming

- Prefix shared components with `shared-` (e.g., `shared-crds-gateway`, `shared-secrets-vault`).
- Services mirror their Helm chart name (`service-temporal`, `service-keycloak`).
- First-party apps mirror the repo service name (`app-platform`, `app-agents`).
- ApplicationSets live under `applicationsets/` and generate appropriately prefixed child apps.

## Dependency Strategy

1. **Sync Waves** – Shared primitives run in negative waves (bootstrap ≈ -95, CRDs ≈ -93, secrets ≈ -85). Services hover near 0, first-party apps > 0. Waves strictly control **creation order** inside the parent app; they do *not* guarantee the child Applications are Healthy yet.  
2. **Progressive Syncs for Cross-App Gates** – When groups of Applications must finish before others begin (e.g., secret bundles in batches, or all shared components before services), enable the ApplicationSet progressive sync controller and define `rollingSync` steps. The controller waits for child Applications to report Healthy before advancing to the next group.  
3. **Kubernetes-Native Hooks** – Each service/application ships PreSync/PostSync hook Jobs that wait on their prerequisites using `kubectl wait`/`kubectl get` (e.g., CRD Established, ExternalSecret present, Vault secret template committed). This keeps readiness logic close to the consumer without shelling out to the Argo CLI.  
4. **Optional Orchestrators** – For higher-order promotions (cross-cluster, multi-env), consider Argo Workflows/Events or Kargo; the hooks + progressive syncs cover the majority of in-cluster dependencies.

This mix ensures shared components stay lean while services/apps block until their actual dependencies emit Healthy status.

## Migration Steps

1. **Create folders** (`shared/`, `services/`, `applications/`, `applicationsets/`) under `infra/kubernetes/gitops/argocd/`.
2. **Move shared components** out of the monolithic `argocd-apps.yaml` into `shared/`. Start with:
   - `bootstrap/namespace.yaml`, `bootstrap/smoke.yaml`
   - `crds/gateway.yaml`, `crds/prometheus.yaml`
   - `secrets-core/vault.yaml`, `secrets-core/cert-manager.yaml`, `secrets-core/external-secrets.yaml`
   - Secret bundle ApplicationSet (`applicationsets/secret-bundles.yaml`)
3. **Map services**: for every chart under `infra/kubernetes/environments/<env>/platform/services/`, create `gitops/argocd/services/<name>.yaml` targeting that chart/Helmfile selector, or emit them from an ApplicationSet if the pattern is consistent.
4. **Map first-party apps**: relocate the existing `applications/*.yaml` files into `applications/` and ensure they reference the `/services/<name>` chart.
5. **Enable directory recursion** in each app-of-apps so nested folders get picked up:
   ```yaml
   spec:
     source:
       path: infra/kubernetes/gitops/argocd/shared
       directory:
         recurse: true
   ```
6. **Enable ApplicationSet progressive syncs** (controller flag/ConfigMap) and define rolling sync steps for bundles that must finish before others begin.
7. **Rewire Helmfile values** (or chart values) to include the new folder structure, replacing the giant `applications:` map.
8. **Document** the hierarchy and controller settings in `infra/kubernetes/gitops/argocd/README.md`.

## Validation Checklist

- [ ] `helmfile -e local sync --selector name=argocd-apps` recreates Argo CD + new Applications.
- [ ] `kubectl get app -n argocd` shows distinct `shared-*`, `service-*`, and `app-*` rows.
- [ ] Hook Jobs succeed (PreSync/PostSync) when CRDs/secrets/operators are ready.
- [ ] ApplicationSet progressive sync logs show each `rollingSync` step completing before the next.
- [ ] CI job runs `helmfile template --selector component=<name>` / `argocd app diff` for each folder.

## Open Questions

1. **Per-environment overrides** – Prefer ApplicationSets that emit one Application per environment to get distinct histories, RBAC, and promotions, but confirm we’re comfortable duplicating names/metrics.  
2. **Service catalog** – Decide whether third-party services are emitted via ApplicationSets (strong consistency) or explicit files (maximum clarity) when bespoke policies differ.  
3. **Lifecycle tooling** – Hooks + progressive syncs cover most in-cluster deps; only adopt external orchestrators when cross-cluster or promotion workflows demand it.

Answering these will let us finalize the move away from the “infra-10/12/15” mindset and keep GitOps artifacts aligned with how the platform is actually organized.
