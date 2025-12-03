# 438 – cert-manager DNS-01 with Azure Workload Identity

**Updated**: 2025-12-03

> **Related backlogs:**
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) — Environment naming, domain matrix, and certificate architecture
> - [386-envoy-gateway-cert-tls-docs.md](386-envoy-gateway-cert-tls-docs.md) — TLS strategy for Envoy Gateway and cert-manager PKI
> - [418-secrets-strategy-map.md](418-secrets-strategy-map.md) — secrets posture overview (cert-manager is part of Layer 15)
> - [436-envoy-gateway-observability.md](436-envoy-gateway-observability.md) — Envoy Gateway telemetry using `EnvoyProxy` resource

## Overview

This document describes the configuration and troubleshooting for cert-manager DNS-01 challenges using Azure DNS and Azure Workload Identity. This setup enables automated TLS certificate issuance for `*.dev.ameide.io` (and other child zones) via Let's Encrypt without storing Azure credentials in Kubernetes Secrets.

### Domain and Certificate Matrix

Per [434](434-unified-environment-naming.md#appendix-certificate--gateway-architecture):

| Environment | DNS Zone | ClusterIssuer | Certificate |
|-------------|----------|---------------|-------------|
| dev | `dev.ameide.io` | `letsencrypt-dev-dns01` | `ameide-wildcard-dev` |
| staging | `staging.ameide.io` | `letsencrypt-staging-dns01` | `ameide-wildcard-staging` |
| production | `ameide.io` | `letsencrypt-prod-dns01` | `ameide-wildcard-prod` |
| ArgoCD (cluster) | `ameide.io` | `letsencrypt-apex-dns01` | `argocd-ameide-io` |

**Key decisions:**
- Production uses apex domain (`ameide.io`) directly — there is NO `prod.ameide.io`
- ArgoCD uses apex issuer for `argocd.ameide.io` (cluster-level, not env-specific)
- Each environment has its own DNS zone and ClusterIssuer

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Azure Subscription                              │
│                                                                          │
│  ┌─────────────────────┐      ┌─────────────────────────────────────┐  │
│  │   Managed Identity   │      │           Azure DNS                  │  │
│  │  (ameide-dns-mi)     │─────▶│  Parent: ameide.io                  │  │
│  │                       │      │  Child:  dev.ameide.io              │  │
│  │  Federated Cred:     │      │          staging.ameide.io          │  │
│  │  system:serviceaccount│      │                                     │  │
│  │  :cert-manager:       │      │  Role: DNS Zone Contributor         │  │
│  │  foundation-cert-mgr  │      └─────────────────────────────────────┘  │
│  └─────────────────────┘                                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
              │ OIDC Token Exchange
              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              AKS Cluster                                 │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      cert-manager namespace                       │   │
│  │                                                                   │   │
│  │  ServiceAccount: foundation-cert-manager                         │   │
│  │    annotations:                                                   │   │
│  │      azure.workload.identity/client-id: <dns-mi-client-id>       │   │
│  │      azure.workload.identity/tenant-id: <tenant-id>              │   │
│  │    labels:                                                        │   │
│  │      azure.workload.identity/use: "true"                         │   │
│  │                                                                   │   │
│  │  Pod: foundation-cert-manager                                     │   │
│  │    labels:                                                        │   │
│  │      azure.workload.identity/use: "true"                         │   │
│  │    args:                                                          │   │
│  │      --dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53        │   │
│  │      --dns01-recursive-nameservers-only                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        ameide namespace                           │   │
│  │                                                                   │   │
│  │  ClusterIssuer: letsencrypt-dev-dns01                            │   │
│  │    solver:                                                        │   │
│  │      dns01:                                                       │   │
│  │        azureDNS:                                                  │   │
│  │          hostedZoneName: dev.ameide.io                           │   │
│  │          resourceGroupName: Ameide                                │   │
│  │          subscriptionID: <sub-id>                                 │   │
│  │          environment: AzurePublicCloud                            │   │
│  │          managedIdentity:                                         │   │
│  │            clientID: <dns-mi-client-id>                          │   │
│  │                                                                   │   │
│  │  Certificate: ameide-wildcard-dev                                 │   │
│  │    dnsNames: ["dev.ameide.io", "*.dev.ameide.io"]                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Bicep Infrastructure

### DNS Identity Module

The DNS managed identity and role assignments are defined in:
- `bicep/managed-application/modules/dnsIdentity.bicep`

Key resources:
1. **User-Assigned Managed Identity** (`ameide-dns-mi`)
2. **Federated Identity Credential** for cert-manager service account
3. **DNS Zone Contributor role** on parent zone (`ameide.io`)
4. **DNS Zone Contributor role** on each child zone (`dev.ameide.io`, `staging.ameide.io`)
5. **Child zone creation** via `dnsChildZone.bicep` module
6. **NS delegation records** from parent to child zones

### Critical Configuration Points

```bicep
// main.bicep
var certManagerServiceAccountSubject = 'system:serviceaccount:cert-manager:foundation-cert-manager'

module dnsIdentity './modules/dnsIdentity.bicep' = {
  params: {
    enableFederatedIdentity: enableDnsFederatedIdentity  // Must be true
    oidcIssuer: aks.outputs.oidcIssuer
    certManagerServiceAccount: certManagerServiceAccountSubject
    childZones: dnsChildZones  // ['dev.ameide.io', 'staging.ameide.io']
  }
}
```

The federated identity credential binds:
- **Issuer**: AKS OIDC issuer URL
- **Subject**: `system:serviceaccount:cert-manager:foundation-cert-manager`
- **Audience**: `api://AzureADTokenExchange`

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
# Use well-known recursive nameservers for reliable propagation checks
dns01RecursiveNameservers: "8.8.8.8:53,1.1.1.1:53"
dns01RecursiveNameserversOnly: true

# Azure Workload Identity labels
serviceAccount:
  labels:
    azure.workload.identity/use: "true"

podLabels:
  azure.workload.identity/use: "true"

cainjector:
  podLabels:
    azure.workload.identity/use: "true"

webhook:
  podLabels:
    azure.workload.identity/use: "true"

startupapicheck:
  podLabels:
    azure.workload.identity/use: "true"
```

### Per-environment values (dev)

File: `sources/values/dev/foundation/foundation-cert-manager.yaml`

```yaml
# Azure Workload Identity annotations for DNS-01 validation
serviceAccount:
  annotations:
    azure.workload.identity/client-id: "<dns-mi-client-id>"
    azure.workload.identity/tenant-id: "<tenant-id>"
```

**Important**: The `client-id` must match the DNS managed identity, not the AKS managed identity.

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
kubectl logs deployment/foundation-cert-manager -n cert-manager | grep "configured acme dns01 nameservers"
# Should show: nameservers=["8.8.8.8:53","1.1.1.1:53"]
```

#### 2. 401 Unauthorized from Azure AD

**Symptom:**
```
Error presenting challenge: authentication failed:
POST https://login.microsoftonline.com/<tenant>/oauth2/v2.0/token
RESPONSE 401 Unauthorized
```

**Causes:**
- Federated identity credential not created or misconfigured
- Service account subject mismatch
- Missing `azure.workload.identity/use: "true"` labels/annotations

**Diagnosis:**
```bash
# Check federated credentials on the managed identity
az identity federated-credential list \
  --identity-name ameide-dns-mi \
  --resource-group Ameide

# Verify service account annotations
kubectl get sa foundation-cert-manager -n cert-manager -o yaml
```

**Fix:** Ensure Bicep parameter `enableDnsFederatedIdentity: true` and redeploy.

#### 3. 403 Forbidden on Azure DNS

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

# Check cert-manager logs
kubectl logs deployment/foundation-cert-manager -n cert-manager --tail=50

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
   kubectl rollout restart deployment/foundation-cert-manager -n cert-manager
   ```

## File Reference

| Purpose | Path |
|---------|------|
| DNS identity Bicep | `bicep/managed-application/modules/dnsIdentity.bicep` |
| DNS role assignment | `bicep/managed-application/modules/dnsIdentityRoleAssignment.bicep` |
| Child zone creation | `bicep/managed-application/modules/dnsChildZone.bicep` |
| NS delegation | `bicep/managed-application/modules/dnsNsDelegation.bicep` |
| Main Bicep | `bicep/managed-application/main.bicep` |
| Shared cert-manager values | `sources/values/_shared/foundation/foundation-cert-manager.yaml` |
| Dev cert-manager values | `sources/values/dev/foundation/foundation-cert-manager.yaml` |
| Platform cert-manager config | `sources/values/dev/platform/platform-cert-manager-config.yaml` |

## Changelog

- **2025-12-03**: Initial documentation after debugging DNS-01 propagation check failures and 403 errors on child zones.
