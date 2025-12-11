# Azure AD SSO for Control Plane

## Context
- Strategic direction: every control-plane UI must authenticate with Azure AD across all environments (local, staging, production), while customer-facing or tenant-scoped applications continue to rely on Keycloak for federation and per-tenant directories.
- Kubernetes Dashboard already used Azure AD in production; local and staging clusters now consume the same oauth2-proxy configuration backed by Azure Key Vault so all environments behave consistently.
- Argo CD is delivered via Helmfile (OCI chart) with Azure AD OIDC, the admin account disabled, and RBAC mapped to the shared `ameide-aks-admins` Azure AD group adopted across environments.
- Grafana and the rest of the observability surface (Prometheus, Alertmanager, Loki, Tempo), Temporal Web, and pgAdmin are all fronted by oauth2-proxy (or native OIDC) with Microsoft Entra ID; every environment now sources credentials from Azure Key Vault.
- Secrets flow from Azure Key Vault across all environments. The manifest of expected keys lives in `charts/values/platform/azure-keyvault-secrets.yaml`, and local clusters read from the `ameidelocalkv` vault inside the `Ameide-Local` resource group. Environment overlays currently map all control-plane roles to the shared Azure AD group `ameide-aks-admins` until dedicated read-only groups are provisioned.
- Documentation, runbooks, and GitOps references have been updated to reflect the Azure-only access pattern and the retirement of the legacy Argo CD install script.

## Goals
- Enforce Azure AD OIDC on every control-plane surface in each environment, including local development clusters, with matching redirect URIs (`*.ameide.io`, `*.staging.ameide.io`, `*.dev.ameide.io`).
- Stand up a delegated `staging.ameide.io` DNS zone (Azure Public DNS) mirroring production records so staging control-plane traffic stays isolated while following the same Azure AD and TLS patterns.
- Keep tenant and end-user applications bound to Keycloak for identity, federation, and per-customer SSO across environments.
- Deliver reproducible deployments using Helmfile-managed charts and Azure Key Vault + ExternalSecret across every environment without ad-hoc manual edits.
- Document Azure AD app-registration requirements (redirect URIs, group claims, automation strategy) and map them cleanly to Helm values so teams can audit and re-create the configuration.

## Control-Plane Surface Inventory

| Service | Purpose | Production FQDN | Staging FQDN | Local FQDN (CoreDNS) | Azure AD Status |
| --- | --- | --- | --- | --- | --- |
| Argo CD | GitOps orchestration | `argocd.ameide.io` | `argocd.staging.ameide.io` | `argocd.dev.ameide.io` | Helmfile-managed (OCI) with Azure AD OIDC; `ameide-aks-admins` mapped as admin/read-only; awaiting Key Vault secret seeding |
| Grafana | Observability dashboards | `grafana.ameide.io` | `grafana.staging.ameide.io` | `grafana.dev.ameide.io` | Azure AD enabled via `grafana.ini`; secrets sourced from Azure Key Vault in every environment (local draws from `ameidelocalkv`) |
| Kubernetes Dashboard (oauth2-proxy) | Cluster UI | `k8s-dashboard.ameide.io` | `k8s-dashboard.staging.ameide.io` | `k8s-dashboard.dev.ameide.io` | Azure AD in all environments; Azure Key Vault wiring complete |
| Postgres Admin (pgAdmin) | Database administration | `pgadmin.ameide.io` | `pgadmin.staging.ameide.io` | `pgadmin.dev.ameide.io` | Azure AD-native Helm chart & Gateway routes committed; secrets pending |
| Temporal Web UI | Workflow visibility | `temporal.ameide.io` | `temporal.staging.ameide.io` | `temporal.dev.ameide.io` | oauth2-proxy config shipped with `ameide-aks-admins`; blocked on Azure AD client secret rotation |
| Alertmanager / Prometheus consoles | Alerting & metrics | `alertmanager.ameide.io`, `prometheus.ameide.io` | `alertmanager.staging.ameide.io`, `prometheus.staging.ameide.io` | `alertmanager.dev.ameide.io`, `prometheus.dev.ameide.io` | oauth2-proxy + Gateway + Key Vault entries enforced by `ameide-aks-admins`; rollout pending credential seeding |
| Loki / Tempo consoles | Logs & traces | `loki.ameide.io`, `tempo.ameide.io` | `loki.staging.ameide.io`, `tempo.staging.ameide.io` | `loki.dev.ameide.io`, `tempo.dev.ameide.io` | oauth2-proxy + Gateway routes committed with `ameide-aks-admins`; awaiting secret materialisation |
| Keycloak Admin Console | Tenant identity management | `auth.ameide.io` | `auth.staging.ameide.io` | `auth.dev.ameide.io` | Remains Keycloak-native (no Azure AD changes) |

_Note:_ Any new ingress in Azure must fit under the wildcards `*.ameide.io` and `*.staging.ameide.io`; CoreDNS already maps `*.dev.ameide.io` to the local Envoy gateway.

## Latest Progress
- [x] Promoted Argo CD to a Helmfile-managed OCI release with Azure AD OIDC (UI + CLI), disabled the legacy admin account, and updated GitOps/infra documentation.
- [x] Fronted Grafana, Prometheus, Alertmanager, Loki, Tempo, and Temporal Web with oauth2-proxy (Azure provider) across every environment and added the corresponding Gateway routes/TLS wiring.
- [x] Added a pgAdmin chart with native Azure AD configuration and integrated it into Helmfile, Gateway, and Key Vault manifests.
- [x] Expanded `azure-keyvault-secrets` to include all control-plane client IDs, secrets, and cookie keys, and aligned local development with the Ameide-Local Key Vault.
- [x] Standardised on the `ameide-aks-admins` Azure AD group for control-plane admin/read-only roles across local, staging, and production until bespoke groups are available, and made Helmfile fail fast when IDs are missing.
- [x] Deprecated the shell-based Argo CD installer and replaced docs/runbooks with the Azure-only workflows and Key Vault seeding guidance.

## Remaining Work

### Azure AD & Key Vault
- [ ] Audit the `ameide-aks-admins` membership (admin vs. read-only personas) and decide whether dedicated read-only groups are required for long-term least privilege.
- [ ] Finalise Azure AD app registrations for every control-plane surface (UI + CLI), capturing client IDs/secrets in automation-friendly form.
- [ ] Seed Azure Key Vault (`ameide`, `ameide-staging`) with the committed secret names (`argocd-oidc-*`, `grafana-azuread-*`, `*-oauth2-proxy-*`, `pgadmin-oidc-*`) and validate ExternalSecret reconciliation.

### Secret Management & Automation
- [ ] Automate Azure AD app registration lifecycle (creation, consent, rotation) via CLI/Terraform/Bicep.
- [ ] Document and test the rotation workflows (Key Vault update ➜ ExternalSecret refresh ➜ pod restart) for each oauth2-proxy/native client.
- [ ] Encrypt and commit final `*.sops.yaml` payloads once Azure secrets are issued for local development parity.

### Deployment Validation
- [ ] Run `helmfile -e staging diff/sync` once secrets and group IDs are in place; validate Azure AD login flows for Argo CD, Grafana, Prometheus, Alertmanager, Loki, Tempo, Temporal, and pgAdmin.
- [ ] Promote to production with a controlled change window and verify group-based RBAC plus logout flows.
- [ ] Capture a break-glass procedure (temporary Azure group membership, admin re-enable) in the operations runbook and socialise with on-call staff.

### Operational Guardrails
- [ ] Publish the Azure AD incident response flow (expired secret, group membership drift, outage) in `docs/operations/azure-ad-control-plane.md`.
- [ ] Add automated alerts that flag oauth2-proxy refresh token expiry or ExternalSecret reconciliation failures.
- [ ] Establish quarterly reviews to ensure Azure AD app registrations stay aligned with DNS/TLS records and Helm values.

### Documentation & DNS Follow-Up
- [ ] Complete the `staging.ameide.io` DNS delegation work and ensure cert-manager DNS01 automation is in place.
- [ ] Publish the finalised runbooks covering Azure AD access, secret rotation, and emergency procedures.

## Risks & Mitigations
| Risk | Impact | Mitigation |
| --- | --- | --- |
| Azure AD secret rotation missed | Control-plane UIs reject logins | Automate secret rotation via CLI/Terraform and schedule reminders; expose rotation runbook |
| Group membership drift (`ameide-aks-admins`) | Loss of access or privilege escalation | Add Azure AD audit alerts, document break-glass, plan for dedicated read-only groups |
| ExternalSecret lag after Key Vault update | Stale credentials in pods | Monitor ExternalSecret status, add readiness probes that surface expired tokens, document manual restart path |
| TLS/DNS drift between environments | Login redirects fail | Track DNS zones in IaC, gate Helmfile sync on explicit hostnames, add staging delegation SOP |

## Timeline (Target)
- **Week 1:** Finalise Azure AD registrations and seed Key Vaults (production, staging, Ameide-Local).
- **Week 2:** Validate Helmfile sync in staging, complete documentation, cut operations runbook.
- **Week 3:** Production rollout with change window, publish post-deployment validation, close out risk register.
