# 445: ArgoCD Namespace Isolation

## Status: Implemented

## Summary

ArgoCD runs in its own isolated `argocd` namespace with dedicated infrastructure resources, completely independent of application environment namespaces (dev/staging/production). This ensures ArgoCD remains operational even when all environment namespaces are scaled down or unavailable.

## Problem

Previously, ArgoCD TLS certificates depended on cert-manager deployed in environment namespaces (e.g., `ameide-dev`). This created several issues:

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
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │   │
│  │  │   ArgoCD    │  │ cert-manager│  │ ClusterIssuer +     │  │   │
│  │  │   Server    │  │  (isolated) │  │ Certificate         │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │   │
│  │                                                               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              │ Manages via GitOps                    │
│                              ▼                                       │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐    │
│  │   ameide-dev     │ │  ameide-staging  │ │   ameide-prod    │    │
│  │                  │ │                  │ │                  │    │
│  │  cert-manager    │ │  cert-manager    │ │  cert-manager    │    │
│  │  (environment)   │ │  (environment)   │ │  (environment)   │    │
│  │                  │ │                  │ │                  │    │
│  │  apps, data,     │ │  apps, data,     │ │  apps, data,     │    │
│  │  platform...     │ │  platform...     │ │  platform...     │    │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Cert-Manager Instances

There are **multiple cert-manager instances** in the cluster, each with a specific purpose:

| Instance | Namespace | Purpose | DNS Zone | Managed By |
|----------|-----------|---------|----------|------------|
| `argocd-cert-manager` | `argocd` | ArgoCD TLS only | `ameide.io` (apex) | ArgoCD self-manages |
| `foundation-cert-manager` | `ameide-dev` | Dev environment certs | `dev.ameide.io` | ArgoCD ApplicationSet |
| `foundation-cert-manager` | `ameide-staging` | Staging environment certs | `staging.ameide.io` | ArgoCD ApplicationSet |
| `foundation-cert-manager` | `ameide-prod` | Production environment certs | `ameide.io` | ArgoCD ApplicationSet |

### Why Multiple Cert-Managers?

1. **Isolation**: Environment failures don't affect ArgoCD
2. **Namespace-scoped resources**: Each cert-manager manages Issuers and Certificates in its namespace
3. **Independent scaling**: Environments can scale to zero without affecting cluster operations
4. **Cleaner RBAC**: Each cert-manager only needs permissions for its namespace's DNS zone

## ArgoCD Namespace Contents

### Applications (self-managed via `argocd-config`)

```
argocd/
├── bootstrap-app.yaml       # Applied by bootstrap.sh, syncs argocd/ directory
├── kustomization.yaml       # Defines what ArgoCD manages
├── cert-manager.yaml        # Deploys cert-manager in argocd namespace
├── argocd-tls.yaml          # ClusterIssuer + Certificate for argocd.ameide.io
├── tls/                     # Helm chart for TLS resources
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── clusterissuer.yaml
│       └── certificate.yaml
├── applications/
│   └── cluster-gateway.yaml # Envoy Gateway for argocd.ameide.io
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
| Deployment | `cert-manager` | TLS certificate management |
| Deployment | `cert-manager-webhook` | Cert-manager admission webhook |
| Deployment | `cert-manager-cainjector` | CA bundle injection |
| Service | `argocd-server` | LoadBalancer with static IP |
              | Secret | `argocd-ameide-io-tls` | TLS certificate for HTTPS |
              | ClusterIssuer | `letsencrypt-argocd` | Let's Encrypt for argocd.ameide.io |
              | Certificate | `argocd-server` | TLS cert for argocd.ameide.io |
              | GatewayClass | `envoy-cluster` | Envoy Gateway controller class |
              | Gateway | `cluster` | HTTPS gateway for argocd.ameide.io |
              | Gateway | `cluster-redirect` | HTTP→HTTPS redirect gateway |
              | HTTPRoute | `argocd` | Routes traffic to argocd-server |
              | HTTPRoute | `http-to-https-redirect` | 301 redirect from HTTP |
              | EnvoyProxy | `cluster-proxy-config` | LoadBalancer with static IP |

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
1. Re-run `bootstrap/bootstrap.sh --install-argo --apply-root-apps`
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

## Implementation Commits

- `feat: add cluster-scoped ArgoCD TLS with Helm chart pattern`
- `feat: add isolated cert-manager for ArgoCD namespace`
- `chore: sync cluster globals from terraform outputs`
- `feat: add cluster-gateway with Envoy for argocd.ameide.io`
- `fix: align certificate secret name to argocd-ameide-io-tls`
