# 448: Cert-Manager Workload Identity Architecture

## Status: Superseded (Option A: single cert-manager per cluster)

**Update (2025-12-15)**:
- Cert-manager is now deployed once per cluster as `cluster-cert-manager` in the `cert-manager` namespace (see `backlog/519-gitops-fleet-policy-hardening.md` section **4.5**).
- The per-environment `foundation-cert-manager` + `foundation-cert-manager-wi` (“multi-identity per env”) topology is deprecated and removed from the GitOps component set.
- Azure Workload Identity for DNS-01 is now configured on the single cert-manager install (cluster-scoped), rather than via per-environment helper charts.

The remainder of this document is retained for historical context, but should not be treated as the current reference topology.

## Problem

Cert-manager needs Azure Workload Identity to authenticate with Azure DNS for ACME DNS-01 challenges. The client ID must be set as an annotation on the ServiceAccount.

**Challenges:**
1. cert-manager Helm chart has strict JSON schema validation - rejects unknown fields like `azure`
2. Using `excludeGlobals: true` means globals.yaml isn't available for templating
3. Hardcoding client IDs in values files breaks the Terraform → GitOps data flow

## Solution

Externalize ServiceAccount management following vendor recommendations:
- [cert-manager Best Practice](https://cert-manager.io/docs/installation/best-practice/)
- [Azure Workload Identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. Terraform creates managed identity                            │
│    └── outputs dns_identity_client_ids: {dev: "abc", ...}       │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. sync-globals.sh reads JSON, writes to globals.yaml            │
│    └── azure.dnsManagedIdentityClientId: "abc"                  │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. ArgoCD deploys foundation-cert-manager-wi (rolloutPhase 115)  │
│    └── Creates ServiceAccounts with WI annotation from globals  │
│    └── Chart: sources/charts/foundation/operators-config/cert-manager-wi
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. ArgoCD deploys foundation-cert-manager (rolloutPhase 120)     │
│    └── Uses externally-created SAs (serviceAccount.create=false)│
│    └── Chart: sources/charts/third_party/jetstack/cert-manager  │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│ 5. Azure WI Webhook injects credentials into cert-manager pods   │
│    └── AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_FEDERATED_TOKEN  │
└──────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Path | Purpose |
|-----------|------|---------|
| cert-manager-wi chart | `sources/charts/foundation/operators-config/cert-manager-wi/` | Creates SAs with WI annotations |
| cert-manager-wi component | `environments/_shared/components/foundation/operators-config/cert-manager-wi/` | ArgoCD app definition |
| cert-manager values | `sources/values/_shared/foundation/foundation-cert-manager.yaml` | `serviceAccount.create: false` |

### Helm Values Configuration

**cert-manager-wi chart** (our chart, accepts globals):
```yaml
azure:
  dnsManagedIdentityClientId: "{{ from globals.yaml }}"
```

**cert-manager chart** (third-party, strict schema):
```yaml
# Per-environment naming for complete RBAC isolation
fullnameOverride: cert-manager-dev  # cert-manager-staging, cert-manager-prod

serviceAccount:
  create: false  # Don't create, use external SA
  name: cert-manager-dev  # cert-manager-staging, cert-manager-prod
```

## Rollout Order

1. **Phase 115**: `foundation-cert-manager-wi` - Creates ServiceAccounts
2. **Phase 120**: `foundation-cert-manager` - Deploys cert-manager using those SAs

### Local (offline) environments

Local k3d clusters do not talk to Azure DNS or Let’s Encrypt, so the Workload Identity helper chart is disabled via `enabled: false` in `sources/values/env/local/foundation/foundation-cert-manager-wi.yaml`. In addition, Terraform forces the webhook onto the node network namespace to avoid Pod IP reachability issues (`hostNetwork: true`, `securePort: 10260`) within `sources/values/env/local/foundation/foundation-cert-manager.yaml`. The cert-manager release (`cert-manager-local`) still uses the same ServiceAccount names for RBAC parity, but the SAs are created directly by the upstream chart without Azure annotations. Cloud environments (dev/staging/prod) keep `enabled: true` so the helper continues to enforce the managed identity wiring before cert-manager starts.

## Verification

```bash
# Check SA has correct client ID from globals (per-environment)
kubectl get sa cert-manager-dev -n ameide-dev \
  -o jsonpath='{.metadata.annotations.azure\.workload\.identity/client-id}'
kubectl get sa cert-manager-staging -n ameide-staging \
  -o jsonpath='{.metadata.annotations.azure\.workload\.identity/client-id}'
kubectl get sa cert-manager-prod -n ameide-prod \
  -o jsonpath='{.metadata.annotations.azure\.workload\.identity/client-id}'

# Check pods have WI env vars injected (per-environment)
kubectl get pod -l app.kubernetes.io/instance=dev-foundation-cert-manager -n ameide-dev \
  -o jsonpath='{.items[0].spec.containers[0].env[*].name}'
# Should include: AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_FEDERATED_TOKEN_FILE
```

## DNS-01 Issuer Pattern

We use **namespaced Issuer** (not ClusterIssuer) for DNS-01 challenges with Azure Workload Identity due to our multi-identity architecture:

**Why Issuer instead of ClusterIssuer:**
- Each environment has its own cert-manager instance with isolated Workload Identity
- ClusterIssuer requires a single controller to process challenges from all environments
- That single controller's ServiceAccount would need access to all per-env managed identities
- Namespaced Issuer + `--issuer-ambient-credentials=true` allows each env's controller to use its own identity

**Configuration pattern:**
```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-dev-dns01
  namespace: ameide-dev
spec:
  acme:
    email: admin@ameide.io
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-dev-dns01-key
    solvers:
    - dns01:
        azureDNS:
          hostedZoneName: dev.ameide.io
          resourceGroupName: Ameide
          subscriptionID: "..."
          environment: AzurePublicCloud
          managedIdentity:
            clientID: "<MI client ID from globals.yaml>"
```

**Certificate resources** reference the Issuer:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
spec:
  issuerRef:
    kind: Issuer  # Namespace-scoped
    name: letsencrypt-dev-dns01
```

**Required controller args:**
```yaml
extraArgs:
  - --namespace=$(POD_NAMESPACE)  # Only watch own namespace
  - --issuer-ambient-credentials=true  # Enable WI for Issuers
```

## Per-Environment RBAC Isolation

Each environment uses a unique `fullnameOverride` to ensure complete RBAC isolation:

| Environment | fullnameOverride | ClusterRole/ClusterRoleBinding prefix |
|-------------|------------------|--------------------------------------|
| dev | `cert-manager-dev` | `cert-manager-dev-*` |
| staging | `cert-manager-staging` | `cert-manager-staging-*` |
| prod | `cert-manager-prod` | `cert-manager-prod-*` |

**Why per-environment naming:**
- Helm generates ClusterRoles and ClusterRoleBindings using `fullnameOverride`
- Multiple releases with the same name would overwrite each other's cluster-scoped resources
- The last ArgoCD sync "wins", leaving other environments' SAs without RBAC permissions
- Per-env naming creates unique cluster-scoped resources that don't conflict

**Federated credential subjects** (in Bicep/Terraform):
```
argocd:  system:serviceaccount:argocd:argocd-cert-manager
dev:     system:serviceaccount:ameide-dev:cert-manager-dev
staging: system:serviceaccount:ameide-staging:cert-manager-staging
prod:    system:serviceaccount:ameide-prod:cert-manager-prod
```

## Related

- [434: Unified Environment Naming](434-unified-environment-naming.md) - sync-globals.sh pattern
- [438: DNS-01 Azure Workload Identity](438-cert-manager-dns01-azure-workload-identity.md) - Detailed WI configuration
- [447: Third-Party Chart Tolerations](447-third-party-chart-tolerations.md) - excludeGlobals pattern

## Changelog

- **2025-12-05**: Added per-environment RBAC isolation section with `fullnameOverride` pattern and updated verification commands.
