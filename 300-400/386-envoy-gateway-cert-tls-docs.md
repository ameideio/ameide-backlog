## Envoy Gateway TLS & Cert Management (dev) — Vendor-aligned shape with cert-manager present

> **Status:** Active for shared AKS plus Terraform-managed local clusters. Cloud environments pull wildcard certificates from Let’s Encrypt DNS-01 Issuers (see [438](../438-cert-manager-dns01-azure-workload-identity.md)), while local/offline clusters rely on the self-signed Issuer stack maintained in [444-terraform.md](../444-terraform.md#local-target-safeguards). All flows still defer to cert-manager; certgen remains enabled everywhere.

> **Related**: See [436-envoy-gateway-observability.md](436-envoy-gateway-observability.md) for telemetry configuration using `EnvoyProxy` resource, [417-envoy-route-tracking.md](417-envoy-route-tracking.md) for route inventory, and [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) for dual ApplicationSet architecture.
>
> **Updated 2025-12-16**: Envoy Gateway control-plane runs cluster-shared in `argocd`; EnvoyProxy resources are per-environment in `ameide-{env}`; Envoy data-plane Deployments/Services are created by the controller and live in `argocd`. Control-plane TLS for xDS is managed by Envoy Gateway `certgen` (secrets in `argocd`); external TLS for hostnames is managed by cert-manager (secrets in the environment namespace).

### Current model (vendor-aligned, deterministic)
- **Single cert-manager install per cluster** (`cluster-cert-manager` in `cert-manager` namespace) mints external TLS certificates (wildcards or per-host) and injects webhook CA bundles cluster-wide.
- **Envoy Gateway internal/control-plane TLS (xDS)** is owned by **Envoy Gateway `certgen`** (secrets are created in the controller namespace `argocd`).
- **External TLS (user-facing hostnames)** is owned by **cert-manager** (secrets live in the environment namespace and are referenced by Gateway listeners/routes).

### Namespaces and ownership
- **Controller**: `Deployment/envoy-gateway` in `argocd`
- **Per-env config**: `EnvoyProxy` + `Gateway` + `HTTPRoute` in `ameide-{env}`
- **Data-plane (generated)**: `Deployment/Service` created by Envoy Gateway controller in `argocd` (labeled with the owning Gateway namespace/name)

### Control-plane TLS (xDS) via certgen (current)
- Secrets live in `argocd` (created by the Envoy Gateway chart/job):
  - `Secret/envoy-gateway-ca`
  - `Secret/envoy-gateway`
  - `Secret/envoy`
  - (optional) `Secret/envoy-rate-limit` if rate-limit is enabled
- This keeps the control-plane convergent without requiring cross-app ordering between cert-manager config and the controller install.

### External TLS via cert-manager (current)
- Wildcard cert(s) are issued by cert-manager and referenced by Gateway listeners.
- Local/offline: wildcard is issued by a self-signed CA in-cluster.
- Cloud: wildcard is issued via Let’s Encrypt DNS-01 (Azure Workload Identity).

### Key file paths
- Envoy Gateway shared values: `sources/charts/shared-values/infrastructure/envoy-gateway.yaml`
- Local external TLS + ArgoCD TLS: `sources/values/env/local/platform/platform-cert-manager-config.yaml`
- Per-env Gateway/HTTPRoute config: `sources/values/env/<env>/platform/platform-gateway.yaml`
- EnvoyProxy per-env overrides: see [436-envoy-gateway-observability.md](436-envoy-gateway-observability.md)

## Detailed implementation steps (dev)

### 1) Control-plane PKI (cert-manager)
- Files: `sources/values/env/dev/platform/platform-cert-manager-config.yaml`
  - Issuers:
    - `self-signed-root` (local root CA for dev)
    - `ameidet-ca-issuer` (Issuer from self-signed root)
    - `envoy-gateway-ca-issuer` (Issuer from `envoy-gateway-ca` secret)
  - Certificates (namespace `ameide`):
    - `envoy-gateway-ca` → secret `envoy-gateway-ca` (isCA: true; usages: cert sign/crl sign; SANs: `envoy-gateway-ca.ameide.svc`, `envoy-gateway-ca.ameide.svc.cluster.local`)
    - `envoy-gateway-server` → secret `envoy-gateway` (SANs: `envoy-gateway`, `envoy-gateway.ameide`, `envoy-gateway.ameide.svc`, `envoy-gateway.ameide.svc.cluster.local`; usages: server/client auth)
    - `envoy-proxy-server` → secret `envoy` (SANs: `envoy`, `envoy.ameide`, `envoy.ameide.svc`, `envoy.ameide.svc.cluster.local`; usages: server/client auth)
    - `envoy-ratelimit-server` → secret `envoy-rate-limit` (SANs: `envoy-rate-limit`, `envoy-rate-limit.ameide`, `envoy-rate-limit.ameide.svc`, `envoy-rate-limit.ameide.svc.cluster.local`; usages: server/client auth)
  - Argo ordering per `backlog/387-argocd-waves-v2.md` Platform band:
    - 310: `platform-envoy-crds` (CRDs first)
    - 320: `platform-envoy-gateway` (operator, with `skipCrds: true`)
    - 330: `platform-cert-manager-config` (issues certs as secrets)
    - 340: `platform-gateway` (Gateway CRs that reference certs)
    - This ordering ensures: CRDs exist before operator starts; operator is running before Gateway CRs are applied; TLS secrets exist before Gateway references them. The cert-manager operator runs in Foundation (120), so Certificate CRs at 330 are reconciled immediately.
  - Namespace: all Issuers/Certificates live in `ameide` (matches EG install namespace).
- Orphan ignores for generated secrets: see `environments/dev/components/platform/control-plane/cert-manager-config/component.yaml`.

### 2) Certgen behavior
- Kept enabled in shared values: `sources/charts/shared-values/infrastructure/envoy-gateway.yaml`.
- Effect: certgen job/RBAC are plain Argo-managed resources (vendor Helm hooks removed) and reconcile like any other workload; certgen is the authoritative source of Envoy Gateway internal/xDS TLS secrets in `argocd`.

### 3) Envoy Gateway CRDs and operator wiring
Per `backlog/387-argocd-waves-v2.md`, CRDs are split from the operator chart:
- **CRDs (310)**: `environments/dev/components/platform/control-plane/envoy-crds/component.yaml`
  - Chart: `sources/charts/third_party/oci/docker.io/envoyproxy/gateway-crds-helm`
  - Installs the Envoy Gateway-specific CRDs (EnvoyProxy, BackendTrafficPolicy, etc.) ahead of the operator. Gateway API CRDs come from the dedicated cluster-scoped `crds-gateway-api` component, which vendors `sources/charts/foundation/common/raw-manifests/files/gateway-api-standard-install.yaml` via `sources/values/_shared/foundation/foundation-crds-gateway.yaml`. The shared toggle file `sources/values/_shared/cluster/platform-envoy-crds.yaml` intentionally leaves `crds.gatewayAPI.enabled=false` so SSA ownership stays split cleanly between the two components.
  - `ignoreDifferences` for CRD annotations (Argo vs controller metadata churn)
- **Operator (320)**: `environments/dev/components/platform/control-plane/envoy-gateway/component.yaml`
  - Chart: `sources/charts/third_party/oci/docker.io/envoyproxy/gateway-helm/1.6.0-nocreds`
  - `skipCrds: true` (CRDs handled by platform-envoy-crds at 310)
  - Namespace: `argocd`
  - Value files (in order):
    1) `sources/charts/shared-values/infrastructure/envoy-gateway.yaml`
    2) `sources/values/env/local/platform/infrastructure/envoy-gateway.yaml` (forces the controller Service to `ClusterIP` and strips cloud-specific annotations so local clusters stay internal-only; Azure clusters keep their load balancer settings in `_shared`)
  - Sync options: `CreateNamespace=true`, `RespectIgnoreDifferences=true`, `ServerSideApply=true`, `Replace=true`

### 4) External TLS (Gateways)
- Gateway chart: `sources/charts/apps/gateway`
- TLS wiring: listeners reference secret `ameide-wildcard-tls` already issued by cert-manager via platform cert-manager config.
- Optional enhancement: add `cert-manager.io/issuer`/`cluster-issuer` annotations on Gateway resources if preferring per-Gateway Certificates over the pre-created wildcard.

### 5) Namespacing considerations
- Current architecture:
  - Envoy Gateway operator: `argocd` namespace (cluster-shared)
  - EnvoyProxy resources: environment namespaces (`ameide-{env}`)
  - Gateway/HTTPRoute resources: environment namespaces (`ameide-dev`, `ameide-staging`, `ameide-prod`)
  - Control-plane TLS secrets: `argocd` (managed by Envoy Gateway certgen)
- See [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) for dual ApplicationSet architecture.

### 6) Argo/RollingSync hygiene for certgen
- Certgen resources are normal, non-hook resources; if migrating from an older hook-based release, delete any lingering `platform-envoy-gateway-gateway-helm-certgen*` RBAC/Jobs once, then re-sync so Argo reconciles the plain resources cleanly.

## Troubleshooting (dev) — xDS TLS & Gateway address

### Symptoms

- Envoy proxy pods (`envoy-ameide-*`) log repeated xDS errors like:
  - `DeltaAggregatedResources gRPC config stream to xds_cluster closed ... TLS_error: ... CERTIFICATE_VERIFY_FAILED`
- `Gateway` status:
  - `Accepted=True`
  - `Programmed=False` with `Reason=AddressNotAssigned` and message “No addresses have been assigned to the Gateway”.
- Envoy `LoadBalancer` Services have `EXTERNAL-IP: <pending>` or the Gateway has no `Addresses` entry.

### Expected healthy state

- TLS/xDS:
  - Secrets `envoy-gateway-ca`, `envoy-gateway`, and `envoy` exist in `argocd` with `Type: kubernetes.io/tls` and `ca.crt` populated (created/owned by certgen).
  - External TLS for hostnames is independent: the environment namespace has the listener TLS secret (e.g., `Secret/ameide-wildcard-tls` in `ameide-{env}`) Ready and referenced by the Gateway.
  - Envoy logs **do not** show `CERTIFICATE_VERIFY_FAILED` for `xds_cluster`.
- Gateway / address:
  - `kubectl describe gateway ameide -n ameide` shows:
    - `Accepted=True`
    - `Programmed=True` with a concrete address (e.g. `Addresses: IPAddress 172.18.0.5`).
  - Envoy `LoadBalancer` Services in `ameide` have a non-`<pending>` `EXTERNAL-IP` that matches the Gateway address.

### Runbook: repair xDS TLS via cert-manager (dev)

> This follows Envoy Gateway vendor docs for “Control Plane Authentication using custom certs”: cert-manager owns the root, intermediate, and leaf certs; Envoy Gateway and proxies consume the resulting Secrets.

1. **Confirm PKI objects exist and are Ready**
   - `kubectl get certificates.cert-manager.io -n ameide | egrep 'envoy-gateway|envoy-proxy|envoy-ratelimit'`
   - All three leaf certs should be `Ready=True`. If not, fix `platform-cert-manager-config` values first.

2. **Force cert-manager to re-issue leaf certs (no spec changes)**
   - In namespace `ameide`, delete the Secrets (not the Certificates):
     - `kubectl delete secret envoy envoy-gateway envoy-rate-limit -n ameide`
   - cert-manager will re-create them based on the existing `Certificate` specs:
     - `envoy-gateway-server` → secret `envoy-gateway`
     - `envoy-proxy-server` → secret `envoy`
     - `envoy-ratelimit-server` → secret `envoy-rate-limit`
   - Watch status:
     - `kubectl get certificates.cert-manager.io -n ameide envoy-gateway-server envoy-proxy-server envoy-ratelimit-server`
     - Wait until all are `Ready=True` again.

3. **Restart Envoy data plane so it picks up new certs**
   - Roll the proxy deployments:
     - `kubectl rollout restart deploy/envoy-ameide-ameide-95563066 -n ameide`
     - `kubectl rollout restart deploy/envoy-ameide-ameide-redirect-921c3ee8 -n ameide`
   - Verify pods are Ready:
     - `kubectl get pods -n ameide | grep envoy-ameide`
   - Re-check Envoy logs:
     - `kubectl logs -n ameide deploy/envoy-ameide-ameide-95563066 -c envoy --tail=80`
     - `CERTIFICATE_VERIFY_FAILED` messages for `xds_cluster` should disappear once the cert chain and trust bundle align.

### Runbook: repair Gateway address / LoadBalancer (dev)

> This follows the Gateway API spec: `Programmed=False` with `Reason=AddressNotAssigned` means the implementation could not assign an address. The usual cause in dev is a missing or misconfigured load balancer.

1. **Check Gateway and Service status**
   - `kubectl describe gateway ameide -n ameide`
     - If `Programmed=False` and `Reason=AddressNotAssigned`, there is no usable address backing the Gateway.
   - `kubectl get svc -n ameide | egrep 'envoy-ameide-ameide|envoy-ameide-ameide-redirect'`
     - If `EXTERNAL-IP` is `<pending>`, the cluster’s load balancer integration is not assigning an address.

2. **Fix dev cluster load balancer**
   - For k3d/k3s-based dev clusters:
     - Ensure the cluster is created with a working server load balancer (e.g., `k3d-<cluster>-serverlb`) that exposes ports 80/443.
     - Alternatively, install and configure a load balancer implementation such as MetalLB with an address pool.
   - Once configured correctly:
     - Envoy Services should show a concrete `EXTERNAL-IP` (e.g., `172.18.0.5`).
     - `Gateway/ameide` should transition to `Programmed=True` and gain an `Addresses` entry.

3. **End-to-end verification**
   - Confirm:
     - `kubectl describe gateway ameide -n ameide` → `Accepted=True`, `Programmed=True`, address assigned.
     - Envoy proxy pods are Running and not logging TLS failures for `xds_cluster`.
   - From a machine that can resolve `*.dev.ameide.io` to the Envoy address, hit a known host:
     - Example: `curl -kI https://platform.dev.ameide.io` (or use `--resolve platform.dev.ameide.io:443:<EXTERNAL-IP>` in dev).
   - You should see a 200/30x response from the appropriate backend route.

### Argo CD sync behaviour notes

- Envoy Gateway uses **server-side apply** for Gateway/HTTPRoute/GRPCRoute resources. In clusters where both Argo CD and Envoy Gateway apply these specs, Argo may show them as `OutOfSync` with messages like `... replaced` even when the rendered spec matches what is running.
- In practice this means Envoy Gateway appears as the field manager for some spec fields, and Argo interprets that as the resources being “replaced” even though there is no material drift from Git.
- To keep the `platform-gateway` Application green while still letting Envoy Gateway manage these resources, configure `ignoreDifferences` for the relevant Gateway/HTTPRoute/GRPCRoute kinds in the Argo application, targeting controller-managed metadata/managedFields (for example via `managedFieldsManagers` or specific metadata paths) so Argo ignores the server-side apply churn but still reports real spec drift. In dev this is wired via `environments/dev/components/platform/control-plane/gateway/component.yaml` using `managedFieldsManagers: [envoy-gateway]` for the Gateway API kinds.

### Key file paths summary

Rollout order follows `backlog/387-argocd-waves-v2.md` Platform band: CRDs (310) → Operators (320) → Secrets (330) → Configs (340)

| Phase | Component | File |
|-------|-----------|------|
| 310 | EG CRDs | `environments/dev/components/platform/control-plane/envoy-crds/component.yaml` |
| 320 | EG operator | `environments/dev/components/platform/control-plane/envoy-gateway/component.yaml` |
| 330 | cert-manager config (issues certs) | `environments/dev/components/platform/control-plane/cert-manager-config/component.yaml` |
| 340 | gateway (Gateway CRs) | `environments/dev/components/platform/control-plane/gateway/component.yaml` |

Values and overlays:
- Shared EG values (certgen on/no hooks): `sources/charts/shared-values/infrastructure/envoy-gateway.yaml`
- Dev overlay (nodeSelector + tolerations for the dev pool): `sources/values/env/dev/platform/platform-envoy-gateway.yaml`
- Cert-manager EG PKI (dev): `sources/values/env/dev/platform/platform-cert-manager-config.yaml`

All paths relative to `gitops/ameide-gitops/`.

## Local / Offline Considerations

- Terraform’s local overlay (`sources/values/env/local/platform/platform-cert-manager-config.yaml`) mirrors the same Issuer/Certificate names but sources from the in-cluster SelfSigned CA rather than Let’s Encrypt DNS-01.
- `sources/values/env/local/platform/infrastructure/envoy-gateway.yaml` pins the controller Service to `ClusterIP` (and leaves annotations empty) so k3d/k3s never tries to provision an Azure load balancer or cloud-specific resources while still consuming the shared Helm chart.
- `sources/values/env/local/platform/platform-gateway.yaml` carries the local-only EnvoyProxy overrides: telemetry host/service type swaps to `otel-collector.ameide-local.svc.cluster.local` with `ClusterIP`, backend service names line up with the non-AKS Helm releases, and local-only listener tweaks (e.g., disabling the redirect Gateway) stay scoped to `env/local`.
- Because cert-manager’s webhook runs in hostNetwork mode locally (see [448](../448-cert-manager-workload-identity.md#local-offline-environments)), the EG control-plane certificates reconcile even when Pod IPs aren’t routable inside k3d.
