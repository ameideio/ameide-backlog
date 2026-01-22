# 445: ArgoCD Namespace Isolation

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

## Status: Implemented

## Summary

ArgoCD runs in its own isolated `argocd` namespace with dedicated infrastructure resources, completely independent of application environment namespaces (`ameide-{env}`). This ensures ArgoCD remains operational even when environment namespaces are scaled down or unhealthy.

## Problem

Previously, ArgoCD TLS certificates depended on environment-scoped cert-manager/resources. This created several issues:

1. **Circular dependency**: ArgoCD manages environment deployments, but depended on those environments for its own TLS
2. **Availability risk**: If dev namespace was scaled down or unhealthy, ArgoCD lost TLS capability
3. **Blast radius**: Environment issues could impact cluster-wide GitOps operations

## Architecture

### Namespace Isolation Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    argocd namespace                           │   │
│  │  (Cluster-scoped, always running)                            │   │
│  │                                                               │   │
│  │  ┌─────────────┐  ┌──────────────────┐  ┌─────────────────┐  │   │
│  │  │   ArgoCD    │  │ Cluster Gateway  │  │ argocd-tls App   │  │   │
│  │  │ components  │  │ (Gateway/Routes) │  │ (Certificate)    │  │   │
│  │  └─────────────┘  └──────────────────┘  └─────────────────┘  │   │
│  │                                                               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              │ Manages via GitOps                    │
│                              ▼                                       │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐    │
│  │   ameide-dev     │ │  ameide-staging  │ │   ameide-prod    │    │
│  │                  │ │                  │ │                  │    │
│  │  apps, data,     │ │  apps, data,     │ │  apps, data,     │    │
│  │  platform...     │ │  platform...     │ │  platform...     │    │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                 cert-manager namespace                        │   │
│  │            (single install per cluster)                        │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Cert-manager (single install per cluster)

cert-manager is deployed **once per cluster** as `cluster-cert-manager` in the `cert-manager` namespace (vendor-supported topology). ArgoCD TLS is provided by the `argocd-tls` Application, which creates a `ClusterIssuer` + `Certificate` and relies on `cluster-cert-manager` to reconcile them.

## ArgoCD Namespace Contents

### ArgoCD-managed manifests (`argocd/`)

```
argocd/
├── bootstrap-app.yaml       # Applied by bootstrap.sh, syncs argocd/ directory
├── kustomization.yaml       # Defines what ArgoCD manages
├── argocd-tls.yaml          # ClusterIssuer + Certificate for argocd.ameide.io
├── tls/                     # Helm chart for TLS resources
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── clusterissuer.yaml
│       └── certificate.yaml
├── projects/                # ArgoCD AppProjects
├── applicationsets/         # Root ApplicationSets
└── repos/                   # Repository credentials

sources/charts/cluster/
└── gateway/                 # Cluster-scoped gateway chart
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── gatewayclass.yaml
        ├── envoyproxy.yaml
        ├── gateway.yaml
        ├── httproute-argocd.yaml
        └── redirect-gateway.yaml
```

### Kubernetes Resources in `argocd` Namespace

| Resource | Name | Purpose |
|----------|------|---------|
| Deployment | `argocd-server` | ArgoCD API/UI server |
| Deployment | `argocd-repo-server` | Git repository operations |
| Deployment | `argocd-application-controller` | Reconciliation controller |
| Deployment | `argocd-applicationset-controller` | ApplicationSet controller |
| Deployment | `argocd-dex-server` | SSO/OIDC authentication |
| Deployment | `argocd-redis` | Caching layer |
| Deployment | `argocd-notifications-controller` | Notifications |
| Service | `argocd-server` | ClusterIP (exposed via Gateway API) |
| Secret | `argocd-server-tls` | TLS certificate secret ArgoCD server expects |
| Gateway/HTTPRoute | (cluster-gateway) | Exposes `argocd.ameide.io` via Gateway API |

## Value Flow for TLS

```
Terraform outputs
       │
       ▼
sync-globals.sh cluster
       │
       ▼
sources/values/cluster/globals.yaml
       │
       │  azure:
       │    subscriptionId: "..."
       │    resourceGroup: "Ameide"
       │    dnsZone: "ameide.io"
       │    dnsZoneResourceGroup: "Ameide"
       │    dnsManagedIdentityClientId: "..."
       │
       ▼
argocd/tls/ Helm chart
       │
       ▼
ClusterIssuer (letsencrypt-argocd)
Certificate (argocd-server → argocd-ameide-io-tls secret)
```

## Sync Waves

ArgoCD uses sync waves to ensure correct deployment order:

| Wave | Application | Resources |
|------|-------------|-----------|
| -1 | `argocd-cert-manager` | cert-manager Deployment, CRDs, Webhook |
| 0 | `argocd-tls` | ClusterIssuer, Certificate |
| 1 | `cluster-gateway` | GatewayClass, EnvoyProxy, Gateway, HTTPRoute |
| (default) | Projects, ApplicationSets | AppProjects, ApplicationSets |

## DNS Configuration

| Domain | Type | Target | IP Address | Purpose |
|--------|------|--------|------------|---------|
| `argocd.ameide.io` | A | Cluster Gateway | 20.160.216.7 | ArgoCD UI/API |
| `*.dev.ameide.io` | A | Dev Gateway | 40.68.113.216 | Dev environment apps |
| `*.staging.ameide.io` | A | Staging Gateway | 108.142.228.7 | Staging environment apps |
| `*.ameide.io` | A | Prod Gateway | 4.180.130.190 | Production apps |

## Workload Identity

The ArgoCD cert-manager uses Azure Workload Identity to authenticate with Azure DNS:

```yaml
serviceAccount:
  annotations:
    azure.workload.identity/client-id: "<dns-managed-identity-client-id>"

podLabels:
  azure.workload.identity/use: "true"
```

This allows cert-manager to create DNS-01 challenge records in the `ameide.io` zone for Let's Encrypt validation.

## Operational Considerations

### Scaling Environments to Zero

When environments are scaled down (e.g., dev at night):
- Environment cert-manager pods stop
- Environment certificates remain valid (secrets persist)
- **ArgoCD continues operating normally** with its own cert-manager

### Disaster Recovery

If ArgoCD namespace is deleted:
1. Re-run `bootstrap/argocd-bootstrap.sh --install-argo --apply-root-apps`
2. ArgoCD self-heals by syncing `argocd-config` Application
3. Cert-manager and TLS are restored automatically

### Monitoring

Key metrics to monitor:
- `argocd` namespace pod health
- Certificate expiration (`cert-manager_certificate_expiration_timestamp_seconds`)
- ArgoCD application sync status

## Related Documents

- [438-cert-manager-dns01-azure-workload-identity.md](438-cert-manager-dns01-azure-workload-identity.md) - Cert-manager architecture
- [434-unified-environment-naming.md](434-unified-environment-naming.md) - DNS zone structure
- [442-environment-isolation.md](442-environment-isolation.md) - Environment isolation patterns
- [444-terraform.md](444-terraform.md) - Terraform infrastructure

## System Node Pool Tolerations

The ArgoCD cert-manager requires tolerations to run on system node pools that have `CriticalAddonsOnly` taints:

```yaml
# argocd/cert-manager.yaml
tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
    effect: NoSchedule
```

This is applied to:
- `cert-manager` controller
- `cert-manager-webhook`
- `cert-manager-cainjector`

## Implementation Commits

- `feat: add cluster-scoped ArgoCD TLS with Helm chart pattern`
- `feat: add isolated cert-manager for ArgoCD namespace`
- `chore: sync cluster globals from terraform outputs`
- `feat: add cluster-gateway with Envoy for argocd.ameide.io`
- `fix: align certificate secret name to argocd-ameide-io-tls`
- `fix: add system node tolerations to argocd cert-manager`
