# 448: Cert-Manager Workload Identity Architecture

## Status: Implemented

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
serviceAccount:
  create: false  # Don't create, use external SA
  name: foundation-cert-manager
```

## Rollout Order

1. **Phase 115**: `foundation-cert-manager-wi` - Creates ServiceAccounts
2. **Phase 120**: `foundation-cert-manager` - Deploys cert-manager using those SAs

## Verification

```bash
# Check SA has correct client ID from globals
kubectl get sa foundation-cert-manager -n ameide-dev \
  -o jsonpath='{.metadata.annotations.azure\.workload\.identity/client-id}'

# Check pods have WI env vars injected
kubectl get pod -l app.kubernetes.io/instance=foundation-cert-manager -n ameide-dev \
  -o jsonpath='{.items[0].spec.containers[0].env[*].name}'
# Should include: AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_FEDERATED_TOKEN_FILE
```

## Related

- [434: Unified Environment Naming](434-unified-environment-naming.md) - sync-globals.sh pattern
- [447: Third-Party Chart Tolerations](447-third-party-chart-tolerations.md) - excludeGlobals pattern
