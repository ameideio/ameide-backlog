# Azure Managed Application Offering

## Context
- We want to package the AMEIDE control-plane cluster as an Azure Marketplace Managed Application that tenants can deploy into their subscriptions while we retain day-2 management.
- All control-plane UIs (Argo CD, Grafana, Prometheus, Alertmanager, Loki, Tempo, Temporal Web, pgAdmin, Kubernetes Dashboard) now ship Azure AD configurations in Git; by default every environment points at the shared `ameide-aks-admins` group while dedicated read-only groups are evaluated, and rollout remains gated on supplying final client IDs/secrets per environment.
- External Secrets targets Azure Key Vault with per-environment overlays; secret keys for every control-plane client are declared, but production/staging vaults still need to be populated and automated rotation defined. Local development now reads from the Ameide-Local vault (`ameidelocalkv`). A helper script (`scripts/azure/seed-control-plane-secrets.sh`) can seed vault entries from a JSON payload when needed.
- Marketplace ingestion requires an ARM/Bicep deployment package (template + `createUiDefinition`) plus operations collateral; Terraform can help internally but is not a supported packaging format for the listing.

## Goals
- Ship a single Bicep-based Managed Application that can deploy either the staging or production footprint (parameterised `environment`).
- Move control-plane SSO (Argo CD, Grafana, Kubernetes Dashboard proxy, observability UIs) to Azure AD with per-app registrations and group-based RBAC.
- Store runtime secrets exclusively in Azure Key Vault for staging/production and surface them to the clusters via External Secrets with workload identity.
- Document and automate day-2 operations (patching, telemetry, backups, support) to meet Marketplace requirements.
- Produce partner-ready collateral (runbooks, SLA, support model, pricing inputs) accompanying the technical artefacts.

## Non-Goals
- Migrating tenant-facing applications away from Keycloak (they still use federated directories).
- Shipping a public multi-tenant PaaS offer; this scope is limited to managed clusters we operate for customers.
- Refactoring existing Terraform inventories beyond what is required for DNS or shared services.

## Current State Snapshot
- **Identity**
  - Argo CD now ships an Azure AD OIDC profile (`infra/kubernetes/values/common/argocd.yaml`) with per-environment overrides; `ameide-aks-admins` is wired everywhere as the interim admin/read-only group while the need for distinct read-only roles is assessed.
  - Grafana consumes `auth.azuread` settings and Prometheus / Alertmanager / Loki / Tempo are fronted by dedicated oauth2-proxy releases; secrets are sourced from Azure Key Vault across all environments (local uses `ameidelocalkv`).
  - Azure AD app registrations remain provisional—client IDs, secrets, and group assignments must be captured in automation or documentation before launch.
- **Secrets**
  - Global `azure-keyvault-secrets` chart now exposes client/ cookie secrets for Argo CD, Prometheus, Alertmanager, Loki, Tempo, Temporal, and pgAdmin; staging/production overlays set explicit vault names and workload identity is enabled.
  - Local development relies on the Ameide-Local Key Vault (`ameidelocalkv`) with the same secret names; automation is still required to populate customer Key Vault entries for managed environments.
- **Marketplace Packaging**
  - Initial Bicep scaffolding (`infra/bicep/managed-application/`) defines AKS, user-assigned identity, Key Vault, Log Analytics, and optional DNS plus a `createUiDefinition.json`; pipelines still need to compile and ship the artefacts.
  - CI/CD does not yet bundle marketplace-ready artefacts.

## Scope
- Environments: staging and production (customer Key Vaults) plus local development (Ameide-Local Key Vault).
- Resource coverage: AKS cluster, ACR, Azure Key Vault, DNS wiring, Log Analytics, and per-environment Managed Identities needed for runtime.
- Deliver Managed Application definition with tenant onboarding/offboarding automation.

## Deliverables
- Bicep template + `createUiDefinition.json` (tracked under `infra/bicep/managed-application/`).
- Azure AD app registration definitions (Terraform/Bicep module or documented CLI) with group mappings and secret rotation process.
- Updated Helm/Helmfile values for Argo CD, Grafana, Kubernetes Dashboard proxy, Prometheus/Alertmanager, Loki, Tempo to consume Azure AD + Key Vault secrets.
- GitOps updates for Argo CD App-of-Apps pointing to this graph.
- Operational documentation: runbooks, support workflows, SLA summary, compliance notes, cost model inputs.
  - Initial runbook captured in `docs/operations/managed-application-runbook.md`.
- CI/CD pipeline segment that bundles and publishes the Managed Application artefacts for Partner Center ingestion.

## Latest Progress
- [x] Switched Argo CD to a Helmfile-managed OCI release with Azure AD OIDC and documented the new bootstrap flow.
- [x] Introduced oauth2-proxy releases for Prometheus, Alertmanager, Loki, and Tempo with Azure AD wiring across local/staging/production.
- [x] Extended Azure Key Vault manifest to surface client credentials for all control-plane UIs and set explicit vault names per environment.
- [x] Added marketplace Bicep scaffold (modules, parameters, `createUiDefinition.json`) to unblock packaging automation.
- [x] Switched local Kubernetes Dashboard to Azure AD so every control-plane surface uses Microsoft Entra ID.
- [x] Created packaging helper (`infra/bicep/managed-application/package.sh`) and day-2 operations runbook (`docs/operations/managed-application-runbook.md`).
- [x] Adopted `ameide-aks-admins` as the default control-plane group across environment overlays and added a Key Vault seeding script for oauth2-proxy credentials.
- [ ] Publish automation to mint Azure AD app registrations and rotate secrets.
- [ ] Add CI packaging step that lints, what-ifs, and uploads Managed Application artefacts.
- [ ] Resolve remaining Bicep lint warnings (Key Vault identity metadata, AKS admin username parameterisation, optional DNS module outputs) before Partner Center submission.

## Execution Plan
### 1. Identity Alignment (Azure AD)
- [x] Capture Azure AD settings for every control-plane surface in Helm (redirect URIs, scopes, logout URLs, CLI flows).
- [ ] Automate Azure AD app registrations and ongoing group management (ensuring `ameide-aks-admins` membership is tracked and future read-only splits are supported) and feed the resulting object IDs into environment values.
- [ ] Document emergency access (temporary group membership, admin re-enable) and make it part of the operated runbooks.

### 2. Secret Management Hardening
- [x] Parameterise Key Vault settings and secret names across environments (Helmfile + ExternalSecrets).
- [ ] Seed staging/production vaults with the declared secrets and verify ExternalSecret reconciliation.
- [ ] Implement automated secret provisioning/rotation (CLI/Terraform/Bicep) for every control-plane client.

### 3. Managed Application Packaging
- [x] Scaffold `infra/bicep/managed-application/` (modules, `main.bicep`, parameters, `createUiDefinition.json`, packaging helper).
- [ ] Extend modules for customer networking options and document Managed Resource Group RBAC expectations.
- [ ] Add CI validation/publishing (Bicep lint/what-if, artefact bundling via `package.sh` for Partner Center submissions).
- [ ] Gate CI on zero Bicep warnings or capture justifiable suppressions for outstanding SDK gaps.

### 4. Operations & Compliance Readiness
- [x] Draft managed-application runbook (`docs/operations/managed-application-runbook.md`).
- [ ] Finalise monitoring/alerting hand-off (Log Analytics, Azure Monitor, ticketing integration) and codify it in the runbook.
- [ ] Define upgrade cadence & support collateral (SLAs/SLOs, pricing, data handling, escalation paths).
- [ ] Assemble the marketplace submission package (offer description, legal artefacts, telemetry statements).

### 5. Publishing & Partner Center Intake
- [ ] Stand up a packaging pipeline that versions the Managed Application artefacts, uploads them to storage, and generates the Partner Center submission bundle.
- [ ] Dry-run the Partner Center submission flow with a sandbox offer to validate metadata and policy requirements.
- [ ] Capture ongoing update cadence (e.g., monthly patch rollup) and align with Partner Center update process.

## Validation
- End-to-end dry run in a sandbox subscription deploying the Managed Application (both staging and production variants).
- Authentication journey tests for each control-plane app using Azure AD accounts and groups.
- Secret rotation drill (Key Vault -> External Secrets -> pod reload) in staging.
- Bicep CI pipeline must lint, run what-if, and attach artefacts before tagging a release.

## Dependencies
- Azure AD tenant admin to create app registrations, groups, and grant admin consent.
- Access to Partner Center with permissions to create Managed Application offers.
- Managed identity/role assignments for AKS and automation principals.

## Open Questions
- Where should Azure AD app registrations live in IaC (Terraform vs Bicep vs manual)? Need decision for long-term drift control.
- Do we bundle customer networking (VNet, subnet, private DNS) or assume pre-existing infrastructure provided as parameters?
- Is there a requirement for customer-managed keys or bring-your-own Key Vault that the template must support?
- Which telemetry stack (Azure Monitor, Application Insights, Grafana) must be surfaced to customers vs kept internal?

## Risks & Mitigations
| Risk | Impact | Mitigation |
| --- | --- | --- |
| Azure AD registrations drift outside IaC | Marketplace deployments fail auth | Adopt Terraform/Bicep module, automate secret rotation, document verification checklist |
| Key Vault secret seeding incomplete | Managed Application fails post-deploy | Include pre-flight validation script, add health probes, fail fast in Helmfile |
| Partner Center rejection due to artefact gaps | Launch delayed | Establish lint/what-if gates, run sandbox submission, maintain compliance checklist |
| Customer networking assumptions wrong | Deployment fails in customer subscription | Parameterise VNet/peering options, document prerequisites, include validation step |

## Timeline (Tentative)
- **Phase 1 (Weeks 1–2):** Finalise Azure AD automation, seed Key Vault, close identity deliverables.
- **Phase 2 (Weeks 3–4):** Harden Bicep modules, resolve lint warnings, integrate CI packaging.
- **Phase 3 (Weeks 5–6):** Sandbox deployment validation, Partner Center submission dry-run, publish operations collateral.
