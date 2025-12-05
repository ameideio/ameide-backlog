# 438 – cert-manager DNS-01 with Azure Workload Identity

**Updated**: 2025-12-05

> **Related backlogs:**
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) — Environment naming, domain matrix, and certificate architecture
> - [386-envoy-gateway-cert-tls-docs.md](386-envoy-gateway-cert-tls-docs.md) — TLS strategy for Envoy Gateway and cert-manager PKI
> - [418-secrets-strategy-map.md](418-secrets-strategy-map.md) — secrets posture overview (cert-manager is part of Layer 15)
> - [436-envoy-gateway-observability.md](436-envoy-gateway-observability.md) — Envoy Gateway telemetry using `EnvoyProxy` resource
> - [448-cert-manager-workload-identity.md](448-cert-manager-workload-identity.md) — ServiceAccount externalization and multi-identity architecture

## Overview

This document describes the configuration and troubleshooting for cert-manager DNS-01 challenges using Azure DNS and Azure Workload Identity. This setup enables automated TLS certificate issuance for `*.dev.ameide.io` (and other child zones) via Let's Encrypt without storing Azure credentials in Kubernetes Secrets.

### Domain and Certificate Matrix

Per [434](434-unified-environment-naming.md#appendix-certificate--gateway-architecture):

| Environment | DNS Zone | Issuer | Certificate |
|-------------|----------|--------|-------------|
| dev | `dev.ameide.io` | `letsencrypt-dev-dns01` (Issuer) | `ameide-wildcard-dev` |
| staging | `staging.ameide.io` | `letsencrypt-staging-dns01` (Issuer) | `ameide-wildcard-staging` |
| production | `ameide.io` | `letsencrypt-prod-dns01` (Issuer) | `ameide-wildcard-prod` |
| ArgoCD (cluster) | `ameide.io` | `letsencrypt-argocd` (ClusterIssuer) | `argocd-ameide-io` |

**Key decisions:**
- Production uses apex domain (`ameide.io`) directly — there is NO `prod.ameide.io`
- ArgoCD uses ClusterIssuer for `argocd.ameide.io` (cluster-level, processed by argocd-cert-manager)
- **Environment issuers use namespaced Issuer** (not ClusterIssuer) for multi-identity architecture
- Each environment has its own cert-manager instance with isolated Workload Identity (see [448](448-cert-manager-workload-identity.md))

## Architecture

### Multi-Identity Per-Environment Architecture

Each environment (dev, staging, prod) has its own:
1. **Managed Identity** in Azure (e.g., `ameide-dns-dev-mi`, `ameide-dns-staging-mi`, `ameide-dns-prod-mi`)
2. **Federated Credential** for its ServiceAccount (e.g., `system:serviceaccount:ameide-dev:cert-manager-dev`)
3. **cert-manager instance** in its namespace (namespace-scoped with `--namespace=$(POD_NAMESPACE)`)
4. **Namespaced Issuer** (not ClusterIssuer) for DNS-01 challenges
5. **Unique ClusterRoles/ClusterRoleBindings** via per-env `fullnameOverride` (e.g., `cert-manager-dev`)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Azure Subscription                              │
│                                                                          │
│  ┌─────────────────────┐      ┌─────────────────────────────────────┐  │
│  │  Per-Env Managed ID  │      │           Azure DNS                  │  │
│  │  ameide-dns-dev-mi   │─────▶│  Zone: dev.ameide.io                │  │
│  │  ameide-dns-staging  │─────▶│  Zone: staging.ameide.io            │  │
│  │  ameide-dns-prod-mi  │─────▶│  Zone: ameide.io                    │  │
│  │                       │      │                                     │  │
│  │  Federated Creds:    │      │  Role: DNS Zone Contributor          │  │
│  │  Per SA/namespace    │      │  (per identity per zone)             │  │
│  └─────────────────────┘      └─────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                │ OIDC Token Exchange (per environment)
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              AKS Cluster                                 │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      ameide-dev namespace                         │   │
│  │                                                                   │   │
│  │  ServiceAccount: cert-manager-dev                                │   │
│  │    annotations:                                                   │   │
│  │      azure.workload.identity/client-id: <dev-mi-client-id>       │   │
│  │    labels:                                                        │   │
│  │      azure.workload.identity/use: "true"                         │   │
│  │                                                                   │   │
│  │  Pod: cert-manager-dev                                            │   │
│  │    args:                                                          │   │
│  │      --namespace=$(POD_NAMESPACE)  # Only watches own namespace  │   │
│  │      --issuer-ambient-credentials=true  # Enable WI for Issuers  │   │
│  │      --dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53        │   │
│  │                                                                   │   │
│  │  Issuer: letsencrypt-dev-dns01  (namespace-scoped)               │   │
│  │    solver:                                                        │   │
│  │      dns01:                                                       │   │
│  │        azureDNS:                                                  │   │
│  │          hostedZoneName: dev.ameide.io                           │   │
│  │          managedIdentity:                                         │   │
│  │            clientID: <dev-mi-client-id>                          │   │
│  │                                                                   │   │
│  │  Certificate: ameide-wildcard-dev                                 │   │
│  │    issuerRef:                                                     │   │
│  │      kind: Issuer  # Not ClusterIssuer                           │   │
│  │      name: letsencrypt-dev-dns01                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  (Similar for ameide-staging and ameide-prod namespaces)                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Why Issuer Instead of ClusterIssuer

Per [448](448-cert-manager-workload-identity.md), we use namespaced Issuers because:
- Each environment has its own cert-manager instance with isolated Workload Identity
- ClusterIssuer requires a single controller to process challenges from all environments
- That single controller's ServiceAccount would need access to all per-env managed identities
- Namespaced Issuer + `--issuer-ambient-credentials=true` allows each env's controller to use its own identity

## Bicep/Terraform Infrastructure

### Per-Environment DNS Identity Module

Each environment has its own managed identity with federated credentials. Defined in:
- `infra/bicep/managed-application/modules/dnsIdentity.bicep` (creates 4 identities)
- `infra/terraform/azure/main.tf` (outputs `dns_identity_client_ids`)

Key resources per environment:
1. **User-Assigned Managed Identity** (e.g., `ameide-dns-dev-mi`, `ameide-dns-prod-mi`)
2. **Federated Identity Credential** for that environment's ServiceAccount
3. **DNS Zone Contributor role** on the environment's DNS zone

### Critical Configuration Points

```bicep
// dnsIdentity.bicep creates 4 identities with per-env SA names:
var dnsIdentities = [
  { name: 'ameide-dns-argocd-mi', zone: 'ameide.io', namespace: 'argocd', sa: 'argocd-cert-manager' }
  { name: 'ameide-dns-dev-mi', zone: 'dev.ameide.io', namespace: 'ameide-dev', sa: 'cert-manager-dev' }
  { name: 'ameide-dns-staging-mi', zone: 'staging.ameide.io', namespace: 'ameide-staging', sa: 'cert-manager-staging' }
  { name: 'ameide-dns-prod-mi', zone: 'ameide.io', namespace: 'ameide-prod', sa: 'cert-manager-prod' }
]
```

Each federated identity credential binds:
- **Issuer**: AKS OIDC issuer URL
- **Subject**: `system:serviceaccount:<namespace>:cert-manager-<env>` (per-env naming for RBAC isolation)
- **Audience**: `api://AzureADTokenExchange`

### Data Flow

```
Terraform → sync-globals.sh → globals.yaml → cert-manager-wi chart → ServiceAccount annotation
```

The client ID flows from infrastructure to Kubernetes via:
1. Terraform outputs `dns_identity_client_ids` map
2. `sync-globals.sh` writes to `sources/values/<env>/globals.yaml`
3. `foundation-cert-manager-wi` chart reads `azure.dnsManagedIdentityClientId` from globals
4. Creates ServiceAccount with `azure.workload.identity/client-id` annotation

## Helm Values Configuration

### Shared values (all environments)

File: `sources/values/_shared/foundation/foundation-cert-manager.yaml`

```yaml
# CRDs installed separately by foundation-crds-cert-manager
installCRDs: false
crds:
  enabled: false
  keep: true

# DNS-01 challenge propagation check configuration
dns01RecursiveNameservers: "8.8.8.8:53,1.1.1.1:53"
dns01RecursiveNameserversOnly: true

# Namespace-scoped watching for environment isolation
extraArgs:
  - --namespace=$(POD_NAMESPACE)
  - --enable-certificate-owner-ref=true
  - --issuer-ambient-credentials=true  # Required for WI with Issuers

# Azure Workload Identity labels
podLabels:
  azure.workload.identity/use: "true"

# ServiceAccounts managed externally by cert-manager-wi chart
# Names set per-environment: cert-manager-dev, cert-manager-staging, cert-manager-prod
```

### Per-environment values (dev example)

File: `sources/values/dev/foundation/foundation-cert-manager.yaml`

```yaml
# Per-environment naming for complete RBAC isolation
# Each environment gets unique ClusterRoles and ClusterRoleBindings
fullnameOverride: cert-manager-dev

# ServiceAccount matches the fullnameOverride for RBAC binding
serviceAccount:
  create: false
  name: cert-manager-dev
```

### ServiceAccount Management

ServiceAccounts are managed by the `foundation-cert-manager-wi` chart (see [448](448-cert-manager-workload-identity.md)):
- Reads client ID from `globals.yaml` (`azure.dnsManagedIdentityClientId`)
- Creates ServiceAccount with proper WI annotations
- Deployed at rolloutPhase 115 (before cert-manager at 120)

### Per-environment values (dev)

File: `sources/values/dev/platform/platform-cert-manager-config.yaml`

```yaml
certManager:
  enabled: true
  issuers:
    letsencrypt:
      dns01:
        enabled: true
        kind: Issuer  # Not ClusterIssuer
        name: letsencrypt-dev-dns01
        provider:
          azureDNS:
            hostedZoneName: "{{ .Values.azure.dnsZone }}"
            managedIdentity:
              clientID: "{{ .Values.azure.dnsManagedIdentityClientId }}"

  certificates:
    - name: ameide-wildcard-dev
      issuer: letsencrypt-dev-dns01
      issuerKind: Issuer  # Match the Issuer kind
```

## Troubleshooting

### Common Issues

#### 1. DNS propagation check fails continuously

**Symptom:**
```
propagation check failed: DNS record for "dev.ameide.io" not yet propagated
```

**Cause:** cert-manager's default DNS check uses local/authoritative nameservers which may not reflect recent changes quickly.

**Fix:** Add recursive DNS nameservers to cert-manager:
```yaml
dns01RecursiveNameservers: "8.8.8.8:53,1.1.1.1:53"
dns01RecursiveNameserversOnly: true
```

Verify the setting is active:
```bash
# Per-environment deployments (use appropriate namespace)
kubectl logs deployment/cert-manager-dev -n ameide-dev | grep "configured acme dns01 nameservers"
kubectl logs deployment/cert-manager-staging -n ameide-staging | grep "configured acme dns01 nameservers"
kubectl logs deployment/cert-manager-prod -n ameide-prod | grep "configured acme dns01 nameservers"
# Should show: nameservers=["8.8.8.8:53","1.1.1.1:53"]
```

#### 2. 400 Bad Request from Azure AD Token Exchange

**Symptom:**
```
authentication failed:
POST https://login.microsoftonline.com/<tenant>/oauth2/v2.0/token
RESPONSE 400 Bad Request
```

**Causes:**
- Federated identity credential doesn't exist for this namespace/ServiceAccount combination
- Subject mismatch between Kubernetes SA and Azure federated credential

**Diagnosis:**
```bash
# Check federated credentials on the per-env managed identity
az identity federated-credential list \
  --identity-name ameide-dns-prod-mi \
  --resource-group Ameide

# The subject should be: system:serviceaccount:ameide-prod:cert-manager-prod
# Verify service account annotations
kubectl get sa cert-manager-prod -n ameide-prod -o yaml

# Check WI env vars are injected into the pod
kubectl get pod -l app.kubernetes.io/instance=cert-manager-prod -n ameide-prod \
  -o jsonpath='{.items[0].spec.containers[0].env}' | jq .
```

**Fix:** Create the federated credential in Azure for the correct namespace:
```bash
az identity federated-credential create \
  --name "ameide-prod-cert-manager" \
  --identity-name "ameide-dns-prod-mi" \
  --resource-group "Ameide" \
  --issuer "<aks-oidc-issuer-url>" \
  --subject "system:serviceaccount:ameide-prod:cert-manager-prod" \
  --audience "api://AzureADTokenExchange"
```

#### 3. 401 Unauthorized from Azure AD

**Symptom:**
```
Error presenting challenge: authentication failed:
POST https://login.microsoftonline.com/<tenant>/oauth2/v2.0/token
RESPONSE 401 Unauthorized
```

**Causes:**
- Federated identity credential expired or revoked
- AKS OIDC issuer URL changed
- Missing `azure.workload.identity/use: "true"` labels/annotations

**Diagnosis:**
```bash
# Check federated credentials on the managed identity
az identity federated-credential list \
  --identity-name ameide-dns-dev-mi \
  --resource-group Ameide

# Verify service account annotations
kubectl get sa cert-manager-dev -n ameide-dev -o yaml
```

**Fix:** Verify federated credential configuration matches the AKS OIDC issuer.

#### 4. 403 Forbidden on Azure DNS

**Symptom:**
```
Error presenting challenge: request error:
GET https://management.azure.com/.../dnsZones/dev.ameide.io/TXT/_acme-challenge
RESPONSE 403 Forbidden
ERROR CODE: AuthorizationFailed
```

**Causes:**
- DNS Zone Contributor role not assigned on the child zone
- Role assigned only on parent zone, not child zones

**Diagnosis:**
```bash
# Check role assignments on the child zone
az role assignment list \
  --scope "/subscriptions/<sub>/resourceGroups/Ameide/providers/Microsoft.Network/dnsZones/dev.ameide.io" \
  --query "[?principalType=='ServicePrincipal'].{role:roleDefinitionName, principal:principalId}"
```

**Fix:** The `dnsIdentity.bicep` module includes `childZoneRoleAssignment` for each child zone. Verify it deployed:
```bash
az deployment operation group list \
  --resource-group Ameide \
  --name dnsIdentity \
  --query "[?contains(properties.targetResource.resourceName, 'dns-identity-access')].{name:properties.targetResource.resourceName, status:properties.provisioningState}"
```

#### 4. ClusterIssuer shows "No ClientID found"

**Symptom:**
```
No ClientID found: attempting to authenticate with ambient credentials
```

**Cause:** The ClusterIssuer doesn't have `managedIdentity.clientID` set.

**Fix:** Update the ClusterIssuer spec:
```yaml
spec:
  acme:
    solvers:
    - dns01:
        azureDNS:
          managedIdentity:
            clientID: "<dns-mi-client-id>"  # Must be present
```

### Verification Commands

```bash
# Check certificate status
kubectl get certificate -n ameide
kubectl describe certificate ameide-wildcard-dev -n ameide

# Check challenges
kubectl get challenges -A
kubectl describe challenge <name> -n ameide

# Check cert-manager logs (per-environment)
kubectl logs deployment/cert-manager-dev -n ameide-dev --tail=50
kubectl logs deployment/cert-manager-staging -n ameide-staging --tail=50
kubectl logs deployment/cert-manager-prod -n ameide-prod --tail=50

# Verify DNS record exists
dig @8.8.8.8 TXT _acme-challenge.dev.ameide.io +short

# Test Azure DNS access from inside cluster
kubectl run dns-test --rm -it --image=mcr.microsoft.com/azure-cli -- \
  az login --federated-token "$(cat /var/run/secrets/azure/tokens/azure-identity-token)" \
  --service-principal -u <client-id> -t <tenant-id>
```

### Recovery Procedure

If certificate issuance is stuck:

1. **Delete the stuck certificate** (cert-manager will recreate):
   ```bash
   kubectl delete certificate ameide-wildcard-dev -n ameide
   ```

2. **Wait for new challenge** and monitor:
   ```bash
   kubectl get challenges -A -w
   ```

3. **Check for errors**:
   ```bash
   kubectl describe challenge <new-challenge-name> -n ameide
   ```

4. **Force cert-manager restart** if configuration changed:
   ```bash
   # Restart per-environment deployments
   kubectl rollout restart deployment/cert-manager-dev -n ameide-dev
   kubectl rollout restart deployment/cert-manager-staging -n ameide-staging
   kubectl rollout restart deployment/cert-manager-prod -n ameide-prod
   ```

## File Reference

| Purpose | Path |
|---------|------|
| DNS identity Bicep | `infra/bicep/managed-application/modules/dnsIdentity.bicep` |
| DNS role assignment | `infra/bicep/managed-application/modules/dnsIdentityRoleAssignment.bicep` |
| Child zone creation | `infra/bicep/managed-application/modules/dnsChildZone.bicep` |
| NS delegation | `infra/bicep/managed-application/modules/dnsNsDelegation.bicep` |
| Main Bicep | `infra/bicep/managed-application/main.bicep` |
| Shared cert-manager values | `sources/values/_shared/foundation/foundation-cert-manager.yaml` |
| Dev cert-manager values | `sources/values/dev/foundation/foundation-cert-manager.yaml` |
| Platform cert-manager config | `sources/values/dev/platform/platform-cert-manager-config.yaml` |

## Changelog

- **2025-12-05**: Updated for per-environment cert-manager naming (`cert-manager-dev`, `cert-manager-staging`, `cert-manager-prod`) for complete RBAC isolation. Federated credential subjects updated in Bicep/Terraform.
- **2025-12-03**: Initial documentation after debugging DNS-01 propagation check failures and 403 errors on child zones.
