> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/366 – dev.ameide.io Domain & HTTPS Alignment

**Created:** Nov 2025  
**Owner:** Platform DX / Auth  
**Status:** Draft  
**Motivation:** Playwright and integration jobs now run inside the cluster, but our repo, Helm values, and docs still hard-code `platform.dev.ameide.io` on port 8443 (deprecated). Envoy only exposes HTTPS on 443 internally, so anything that insists on the old port stalls. We need one canonical development hostname on a subdomain we control (`*.dev.ameide.io`) and the default HTTPS port everywhere so laptops, CI, and cluster workloads behave identically without clashing with Bonjour/mDNS.

---

## Objectives
1. **Canonical Hostname** – All developer-facing surfaces (docs, scripts, Helm overlays, Playwright env vars) use `https://platform.dev.ameide.io` (plus `auth.dev.ameide.io`, `api.dev.ameide.io`, etc.).
2. **Standard Port 443** – No component references port 8443; both pods and browsers speak TLS on 443. k3d NodePort stays hidden behind DNS/wildcard certs.
3. **Unified TLS & DNS** – CoreDNS rewrite + wildcard certificate cover `.dev.ameide.io`, so traffic inside the mesh and from laptops resolves the same endpoint.
4. **Authentication Alignment** – Keycloak realms, Auth.js config, and Playwright global setup use `.dev.ameide.io` URLs and share the new certificate bundle.
5. **Tooling Consistency** – Tilt buttons, integration packs, CI workflows, and READMEs reference the same base URL without hostAliases.

> **Why `dev.ameide.io`?** `.local` overlaps with OS-level mDNS/Bonjour and requires invasive workstation changes, while `.dev` (global TLD) enforces HSTS and trusted certificates. Using a delegated subdomain we already own lets cert-manager continue issuing self-signed (or mkcert) bundles, keeps split-horizon DNS simple (CoreDNS + optional dnsmasq), and avoids surprises from multicast DNS or Google-managed HSTS preload behavior.

---

## Workstreams
### 1. DNS + Certificates
- Extend `kube-system/coredns-custom` rewrite to include `*.dev.ameide.io` → Envoy service.
- Provision wildcard cert for `.dev.ameide.io` via cert-manager (self-signed for dev) and reference it from Envoy Gateway TLS listeners. Reissue the CA bundle and update trust docs/scripts.
- Document workstation DNS requirements (e.g., add `*.dev.ameide.io` to `/etc/hosts` or run dnsmasq/split-DNS pointing at the k3d node IP). No mDNS overrides required.

### 2. Gateway & Helm
- Update `gitops/ameide-gitops/sources/charts/apps/gateway/values.yaml` HTTPRoute hostnames to include `.dev.ameide.io` and ensure listeners terminate only on port 443.
- Remove leftover hostAliases/port-proxy YAML; rely on Gateway DNS rewrite instead.
- Verify k3d nodeport still maps host 8443/443 externally without leaking into manifests.

### 3. Application & Auth Config
- Change `AUTH_URL`, `NEXTAUTH_URL`, `NEXT_PUBLIC_PLATFORM_URL`, Keycloak `frontendUrl`, redirect URIs, and Vault secrets to `.dev.ameide.io` hostnames.
- Update Keycloak clients (realm configs + bootstrap scripts) so callback URLs accept `.dev.ameide.io`.
- Reissue Playwright storage state after the domain change.

### 4. Tooling, Docs, Automation
- Search/replace repo references to `platform.dev.ameide.io` on port 8443 (docs, scripts, README, backlog). Keep historical mention but flag as deprecated.
- Update Playwright/integration env files, `.env.test.example`, Tilt instructions, and onboarding docs to use `.dev.ameide.io`.
- Adjust CI workflows (GitHub Actions, Argo CD instructions) to default to the new hostname.

### 5. Validation & Rollout
- Run Playwright + integration packs end-to-end with `.dev.ameide.io` to confirm no timeouts.
- ⚠️ 2025-11-11 – Playwright auth-flow spec (`BASE_URL=E2E_CALLBACK_URL=https://platform.dev.ameide.io pnpm --filter www-ameide-platform test:e2e -- --project=chromium --workers=1 features/auth/__tests__/e2e/auth-flow.spec.ts`) could not complete because `/api/auth/providers` returned 404: the `www-ameide-platform` chart is pending install until `k3d-dev-reg:5000/ameide/www-ameide-platform:latest` is built/pushed. Re-run after the pod is healthy to regenerate storage state on the canonical 443 endpoint.
- Spot-check manual browser login on `https://platform.dev.ameide.io` (after trusting the new cert) to ensure cookies/auth state behave.
- Communicate migration steps to developers (new hosts entry or dnsmasq rule, re-login, repo search results).

---

## Progress – 2025-02-18
- ✅ **Docs & onboarding** – `README.md`, `infra/README.md`, and `infra/kubernetes/environments/README.md` now prescribe HTTPS-only access, document the `.devcontainer/export-cert.sh` trust flow, and point to `.devcontainer/manage-host-domains.sh` for dnsmasq automation.
- ✅ **k3d / Gateway plumbing** – `infra/docker/local/k3d.yaml`, `build/scripts/dev.sh`, `gitops/ameide-gitops/sources/values/local/apps/platform/gateway.yaml`, and the gateway chart README were updated to remove 8080/8443 listeners and describe the 80/443 mapping exclusively.
- ✅ **DNS + TLS configs** – Local CoreDNS values and the cert-manager chart comments reference only `*.dev.ameide.io`; backlog text now mandates single-SAN dev certificates.
- ✅ **Helm chart defaults / policies** – Gateway CORS defaults and inference-gateway `SecurityPolicy` derive HTTPS origins from `Values.domain`; Keycloak HTTPRoute templates inject `X-Forwarded-Port: 443`.
- ✅ **Tooling & tests** – Playwright global setup removed the 8443 rewrite shim, integration `playwright.yaml` dropped the port rewrite flag, telemetry scripts and `scripts/test-ssl.sh` probe the 443 endpoints, and the www SUMMARY file calls out the new standard + trust automation.
- 🔄 **Application/Auth config** – Keycloak realm values, vault secrets, and service env files still need a pass to guarantee no hard-coded `.ameide.test`/8080 strings remain (search results still show hundreds of hits under `services/.../SUMMARY.md` and backlog archives).
- 🔄 **Validation & rollout** – Need end-to-end Playwright/integration regression on the updated domain plus a developer comms plan covering the new DNS/trust helpers.

---

## Infrastructure Refactor Inventory
Every infrastructure surface that still references `*.dev.ameide.io` or host port 8443 must switch to `*.dev.ameide.io` on port 443. The list below captures all known touchpoints so we can track refactors methodically.

### Global Docs & Defaults
- `infra/README.md:74-182` – update service URL tables, Keycloak endpoints, and auth instructions to the new domain/port.
- `infra/kubernetes/environments/README.md:61` – change the documented `domain` example from `dev.ameide.io` to `dev.ameide.io` and call out the new trust/DNS steps.
- `infra/kubernetes/environments/local/values.yaml:5-38` – flip the canonical `domain` value and ensure nested URLs (websocket + platform URL) derive from it.

### DNS & Resolver Layer
- `infra/kubernetes/environments/local/platform/coredns-config.yaml:1-4` – rewrite comments and defaults to map `*.dev.ameide.io` to Envoy.
- `infra/kubernetes/charts/platform/coredns-config/templates/configmap.yaml:9-20` – add `*.dev.ameide.io` rewrites (keeping `.test` during rollout) and confirm `stopHosts`/hostRecords` cover both zones.
- `infra/kubernetes/charts/platform/coredns-config/README.md:3-112` – refresh examples (`nslookup`, troubleshooting) for the new domain.

### TLS & Certificate Management
- `gitops/ameide-gitops/sources/values/local/platform/platform-cert-manager-config.yaml:17-53` – regenerate the wildcard certificate with SANs for `*.dev.ameide.io`, `dev.ameide.io`, and retire `.dev.ameide.io` once migration ends.
- `infra/kubernetes/environments/local/cert-manager/README.md:21-52` – rewrite verification commands (`curl https://platform…`) and certificate tables to the new hostnames.
- `infra/kubernetes/environments/local/infrastructure/keycloak.yaml:4-16` & `infra/kubernetes/charts/operators-config/keycloak_instance/values.yaml:9-18` – set `hostname` and HTTPS port expectations to `auth.dev.ameide.io`.
- `infra/kubernetes/charts/operators-config/keycloak_realm/values.yaml:151-420` – update redirect URIs, web origins, CSP entries, and OAuth client settings that currently reference `https://platform.dev.ameide.io` on port 8443 and the legacy `k8s-dashboard.dev.ameide.io`/`argocd.dev.ameide.io` 8433 endpoints.
- `infra/kubernetes/values/infrastructure/grafana.yaml:1652`, `infra/kubernetes/values/infrastructure/pgadmin.yaml:31-43`, and `infra/kubernetes/charts/platform/pgadmin/values.yaml:60-79` – adjust `GF_SERVER_ROOT_URL`, OAuth redirect URIs, and allowed domains.

### Gateway, Envoy & Routing
- `gitops/ameide-gitops/sources/values/local/apps/platform/gateway.yaml:2-251` – convert listener hostnames, `redirect.httpsPort`, Plausible CORS origin, and every extra HTTPRoute host (ArgoCD, pgAdmin, temporal, prometheus, alertmanager, langfuse, loki, tempo, etc.).
- `infra/kubernetes/environments/local/platform/envoy-gateway.yaml:12-46` – ensure Gateway API resources and the gRPC hostname use `*.dev.ameide.io`.
- `gitops/ameide-gitops/sources/charts/apps/gateway/templates/httproute-keycloak.yaml:17-35` – update `hostnames` and injected `X-Forwarded-*` headers to the new domain/port.
- `gitops/ameide-gitops/sources/charts/apps/gateway/templates/cors-policy.yaml:28-34` – rebase default `Access-Control-Allow-Origin` values away from `dev.ameide.io:8080/8443`.
- `gitops/ameide-gitops/sources/charts/apps/gateway/README.md:80-239` – rewrite listener tables, curl samples, and troubleshooting commands.
- `infra/kubernetes/charts/platform/inference_gateway/templates/securitypolicy.yaml:16` – replace the `https://platform.dev.ameide.io` (port 8443) allowlist entry.

### Service Helm Values & Overlays
- `infra/kubernetes/environments/local/platform/www-ameide-platform.yaml:19-233` – change HTTPRoute hostnames, Auth.js URLs, Keycloak issuers, Plausible base URLs, and websocket/public API settings.
- `infra/kubernetes/environments/local/platform/www-ameide.yaml:15-128` – same updates for the marketing site (CORS origins, Auth URL, Plausible settings).
- `infra/kubernetes/environments/local/platform/langfuse.yaml:13-23` & `langfuse-bootstrap.yaml:3` – point Langfuse UI and seed email domains to `dev.ameide.io`.
- `infra/kubernetes/environments/local/platform/graph.yaml:20`, `platform/langfuse.yaml:13-23`, `platform/inference.yaml:2`, `platform/inference.yaml` comments, `platform/argocd.yaml:27`, `platform/plausible.yaml:2`, `platform/envoy-gateway.yaml:12-46` – update hostnames and comments.
- `infra/kubernetes/environments/local/external-services/gitlab.yaml:9-32` – reconfigure GitLab external URL/host entries.
- Base chart defaults: `infra/kubernetes/values/platform/www-ameide-platform.yaml:62-198`, `.../graph.yaml:52`, `.../inference.yaml:51`, `.../inference-gateway.yaml:49`, `.../threads.yaml:53`, `.../platform.yaml:49` – ensure shared values no longer mention `.dev.ameide.io`.
- `infra/kubernetes/charts/platform/www-ameide-platform/templates/configmap.yaml:59-82` and `README.md:21-105` – refresh sample env vars and issuer docs.
- `infra/kubernetes/charts/platform/graph/README.md:11-71` – update GRPCRoute hostnames and curl examples.

### Observability & OAuth Proxies
- `infra/kubernetes/environments/local/infrastructure/kubernetes-dashboard-oauth2-proxy.yaml:19-35`, `alertmanager-oauth2-proxy.yaml:21-37`, `prometheus-oauth2-proxy.yaml:25`, `temporal-oauth2-proxy.yaml:25` – change redirect URLs, cookie domains, and whitelist domains to `*.dev.ameide.io`.
- `infra/kubernetes/environments/local/platform/plausible.yaml:2` & `www-ameide*.yaml` – adjust base URLs (`https://plausible.dev.ameide.io`/`:443`).
- `infra/kubernetes/values/infrastructure/grafana.yaml:1652` – set Grafana’s root URL + OAuth configs to the new hostname.

### Integration & Test Infrastructure
- `infra/kubernetes/environments/local/integration-tests/e2e/playwright.yaml:4-6` and `infra/kubernetes/environments/staging/integration-tests/e2e/playwright.yaml:9-11` – update `BASE_URL` defaults and ensure staging/local values align after the rename.
- `infra/kubernetes/environments/local/integration-tests/envoy-proxy-service.yaml:4` – keep the direct Envoy proxy service as the only helper for in-cluster HTTPS access to `platform.dev.ameide.io:443` (legacy `port-proxy` Deployments/ConfigMaps have been removed).
- `infra/kubernetes/environments/local/platform/inference.yaml:2` (doc comment) – update the reference for the public inference endpoint.

### Scripts & GitOps Tooling
- `infra/kubernetes/scripts/test-auth.sh:9-58` – rewrite every curl target (`platform.dev.ameide.io`, `auth.dev.ameide.io` on 8443) to the new hostname/port and align validation logic.
- `infra/kubernetes/gitops/argocd/README.md:73` – change the example `argocd login` command to `argocd.dev.ameide.io`.

### Best-Practice Follow-Ups
- **Centralized domain config** – Audit Helm charts/overlays so all hostnames derive from `Values.domain` (or a single helper) instead of hard-coded strings. This prevents future migrations from repeating a repo-wide search/replace.
- **Single-SAN certificates** – Wildcard certificates should now target only `.dev.ameide.io` (plus the apex) to reinforce the canonical domain. Do not reintroduce `.dev.ameide.io` or `.ameide.test` SANs.
- **Trust automation** – Ship an automated CA installation path (DevContainer hook, mkcert export script, or tooling command) that installs the new `dev.ameide.io` root cert in browsers, system trust stores, and Playwright containers. Avoid manual “import this cert” steps.
- **DNS split-horizon** – Document and script a dnsmasq (or systemd-resolved) configuration that serves `dev.ameide.io` locally instead of relying solely on `/etc/hosts`. Owning the subdomain in a real DNS zone also prepares us for remote developer machines.
- **Port 443 parity** – Plan the host-layer change from 8443 to 443. Options: grant k3d bind permissions, front it with a reverse proxy, or add authbind/CAP_NET_BIND_SERVICE in the DevContainer. Track this explicitly so docs/tests stop referencing the old port.
- **Observability/OAuth cookie scope** – Ensure OAuth proxies (Grafana, Prometheus, Temporal, etc.) use environment-driven cookie domains and whitelist entries, and scope them to `.dev.ameide.io` using secure defaults.
- **Shared test env config** – Consolidate Playwright/integration `BASE_URL` definitions into a single shared config (env file or YAML) so domain changes are one edit. Update docs to point tests at the new variable instead of hard-coded URLs.
- **Rollback plan** – Add an operational playbook describing how to revert to `.dev.ameide.io` (DNS, cert-manager, Gateway values) if issues arise. Include communication + trust steps so on-call engineers can back out safely.

---

## Risks & Mitigations
- **Dual-Domain Drift:** Keep `.dev.ameide.io` alive until docs/tests confirm `.dev.ameide.io` parity, then remove.
- **Certificate Trust:** Provide helper script (mkcert/export from cert-manager) so developers can trust the new CA bundle.
- **Cache/Session Flush:** Changing domains logs everyone out; include cleanup instructions.

---

## Definition of Done
- No repo content references `platform.dev.ameide.io` on port 8443 for local workflows.
- `kubectl exec` and browsers both succeed against `https://platform.dev.ameide.io` with the same TLS cert.
- Playwright/global setup authenticates without hacks, and docs clearly describe the `.dev.ameide.io` + port 443 standard.
