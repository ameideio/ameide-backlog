## Envoy Gateway TLS & Cert Management (dev) — Vendor-aligned shape with cert-manager present

### What we’re targeting (matches vendor guidance)
- Keep cert-manager as the CA/issuer and let it mint all EG control-plane and external TLS certs.
- Leave `certgen` **enabled** in values; certgen becomes a no-op when secrets already exist, keeping us on the supported path for OIDC/OAuth2 and other internals.
- Remove vendor Helm hooks so certgen resources are reconciled like any other Argo-managed workload (no hook lifecycle issues).

### Implementation shape (target)
- GitOps: EG control-plane TLS surfaced via cert-manager Issuers/Certificates in the platform cert-manager config chart (dev path below); certgen stays enabled with plain manifests (no hook semantics).
- Namespace: install remains in `ameide` (single namespace for EG + Gateway resources). If desired, can move to `envoy-gateway-system` later; DNS names/certs must be updated accordingly.
- Chart/values: `certgen.enabled=true`; certgen job/RBAC are plain resources (no Helm/Argo hooks) so they reconcile cleanly while cert-manager provides the real certs.

### Control-plane TLS via cert-manager (vendor doc “Control Plane Authentication using custom certs”)
- Issuers: `self-signed-root`, `ameidet-ca-issuer`, `envoy-gateway-ca-issuer` (namespaced to EG install namespace).
- Certificates/secrets: CA (`envoy-gateway-ca`), controller (`envoy-gateway`), proxy (`envoy`), ratelimit (`envoy-rate-limit`).
- DNS SANs cover service/FQDN forms within the install namespace.
- Certgen stays enabled; with these secrets present, certgen is a no-op but keeps EG on the supported path for OIDC/OAuth2 internals.

### External TLS via cert-manager
- Wildcard cert `ameide-wildcard-tls` issued by cert-manager and referenced by the Gateway chart.
- Option: add `cert-manager.io/issuer`/`cert-manager.io/cluster-issuer` annotations on Gateway resources if shifting to per-Gateway Certificates instead of a precreated wildcard.

### Paths and knobs (dev)
- EG shared values (certgen enabled, no hooks): `gitops/ameide-gitops/sources/charts/shared-values/infrastructure/envoy-gateway.yaml`
- EG dev overlay (currently empty): `gitops/ameide-gitops/sources/values/dev/platform/platform-envoy-gateway.yaml`
- Cert-manager control-plane PKI for EG: `gitops/ameide-gitops/sources/values/dev/platform/platform-cert-manager-config.yaml`
- Argo app / wave: see rolloutPhase values in component files (310 CRDs, 320 operator, 330 certs, 340 gateway)

## Detailed implementation steps (dev)

### 1) Control-plane PKI (cert-manager)
- Files: `sources/values/dev/platform/platform-cert-manager-config.yaml`
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
- Effect: certgen job/RBAC are plain Argo-managed resources (vendor Helm hooks removed) and will reconcile like any other workload; with cert-manager-issued secrets present, certgen is a no-op but stays on the supported path.

### 3) Envoy Gateway CRDs and operator wiring
Per `backlog/387-argocd-waves-v2.md`, CRDs are split from the operator chart:
- **CRDs (310)**: `environments/dev/components/platform/control-plane/envoy-crds/component.yaml`
  - Chart: `sources/charts/third_party/oci/docker.io/envoyproxy/gateway-crds-helm`
  - Deploys Gateway API and Envoy Gateway CRDs before the operator
  - `ignoreDifferences` for CRD annotations (Argo vs controller metadata churn)
- **Operator (320)**: `environments/dev/components/platform/control-plane/envoy-gateway/component.yaml`
  - Chart: `sources/charts/third_party/oci/docker.io/envoyproxy/gateway-helm/1.6.0-nocreds`
  - `skipCrds: true` (CRDs handled by platform-envoy-crds at 310)
  - Namespace: `ameide`
  - Value files (in order):
    1) `sources/charts/shared-values/infrastructure/envoy-gateway.yaml`
    2) `sources/values/local/platform/infrastructure/envoy-gateway.yaml` (empty placeholder)
  - Sync options: `CreateNamespace=true`, `RespectIgnoreDifferences=true`, `ServerSideApply=true`, `Replace=true`

### 4) External TLS (Gateways)
- Gateway chart: `sources/charts/apps/gateway`
- TLS wiring: listeners reference secret `ameide-wildcard-tls` already issued by cert-manager via platform cert-manager config.
- Optional enhancement: add `cert-manager.io/issuer`/`cluster-issuer` annotations on Gateway resources if preferring per-Gateway Certificates over the pre-created wildcard.

### 5) Namespacing considerations
- Current: everything in `ameide` (EG control-plane + Gateway resources + certs).
- If moving to vendor default `envoy-gateway-system`, update:
  - Cert-manager Issuers/Certificates namespaces and SANs.
  - Application destination namespace.
  - Gateway secrets references if they stay in a different namespace.

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
  - `Certificate` resources `envoy-gateway-ca`, `envoy-gateway-server`, `envoy-proxy-server`, and `envoy-ratelimit-server` in namespace `ameide` are `Ready=True`.
  - Secrets `envoy-gateway`, `envoy`, and `envoy-rate-limit` exist in `ameide` with `Type: kubernetes.io/tls` and `ca.crt` populated.
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
- Dev overlay (currently empty): `sources/values/dev/platform/platform-envoy-gateway.yaml`
- Cert-manager EG PKI (dev): `sources/values/dev/platform/platform-cert-manager-config.yaml`

All paths relative to `gitops/ameide-gitops/`.
