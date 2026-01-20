# Azure: recreate AKS in a new subscription (no DNS touch)

Goal: provision a fresh AKS + platform stack in a new Azure subscription via GitOps (Terraform in CI + ArgoCD), without changing DNS zones/records.

Related: `backlog/712-traffic-manager-first-redesign.md` (target architecture where a persistent edge owns public DNS/TLS, so cluster recreate/canary no longer depends on DNS-01 during bootstrap).

## Status (2026-01-20)

This procedure has been executed successfully with no customer-facing DNS impact:

- Edge bootstrap (AFD Standard in DNS subscription, **no DNS/custom domain changes**):
  - Workflow: `Terraform Azure Edge Apply` run `21168440973`
  - Outputs published to runtime facts: `runtime-facts/edge/azure/globals.yaml` (Front Door ID + preview/prod `azurefd.net` hostnames + cert Key Vault)
- New cluster bootstrap (AKS in new subscription, **disable_dns=true**):
  - Workflow: `Terraform Azure Apply + Verify` run `21169178304`
  - Inputs: `subscription_id=e4652c74-2582-4efc-b605-9d9ed2210a2a`, `backend_resource_group=Ameide-TFState`, `backend_key=azure-e4652c74.tfstate`, `disable_dns=true`
- Origin hardening (new cluster):
  - NSG rules on port `8080` are in place in the node RG (`MC_Ameide_ameide_westeurope`) to allow `AzureFrontDoor.Backend` and deny `Internet`.

Notes:
- When `disable_dns=true`, the apply workflow intentionally skips “TLS certificates + SSO verification” steps. Use the read-only verification commands below during the no-DNS phase.
- As of this status, Front Door does **not** yet route to the new cluster (origin groups/routes are not created yet). Customer traffic remains on the existing path.

## Inputs

- Old subscription: `68ba6806-178c-41f1-84ec-d7a839799be1`
- New subscription: `e4652c74-2582-4efc-b605-9d9ed2210a2a`
- DNS zones must not be modified during the first bring-up.

## What “no DNS touch” means here

- Run Terraform with `disable_dns=true` in CI.
- Terraform will not create/update Azure DNS zones/records.
- Runtime facts will keep `dnsManagedIdentityClientId` empty, so cert-manager uses self-signed issuers (no Azure DNS-01).

Scope note: “DNS” here means *public* Azure DNS zones/records for `ameide.io` and its child zones. This procedure does not aim to preserve or cut over existing DNS, and it should not modify any existing DNS zones/records during initial bring-up.

## One-time prerequisites (OIDC + RBAC)

Assumptions:

- Workflows use GitHub Actions OIDC (`permissions: id-token: write`) with `azure/login` (no stored Azure secrets).

Minimum RBAC (prefer least privilege, scope to resource groups when possible):

1. On the **target** subscription `e4652c74-...`:
   - `Contributor` to create/update resources.
   - One of: `User Access Administrator` **or** `Role Based Access Control Administrator` at the scope where Terraform needs to create role assignments (Public IP / Key Vault / Managed Identity role bindings). If you grant `Owner`, do **not** also grant `User Access Administrator` (it’s redundant).
2. On the **Terraform state backend** (storage account or container in `backend_resource_group`, default `Ameide-TFState`):
   - Data-plane access via Entra ID: `Storage Blob Data Contributor` (container scope preferred).
3. Key Vault secret source for seeding:
   - The CI identity must be able to read secrets from the Key Vault named by repo var `TF_ENV_SECRETS_KEYVAULT_NAME`.

No-DNS guardrail (defense in depth):

- Ensure no principal used during initial bring-up (CI identity, cert-manager identity, any external-dns identity if enabled) has `DNS Zone Contributor` on the real public zones yet. This prevents DNS writes even if a chart/config slips in.

## 1) Smoke plan (already safe)

Run workflow `.github/workflows/terraform-azure-plan.yaml` (manual dispatch) with:

- `subscription_id`: `e4652c74-2582-4efc-b605-9d9ed2210a2a`
- `backend_resource_group`: `Ameide-TFState`
- `backend_key`: `azure-e4652c74.tfstate` (any unique key is fine)
- `disable_dns`: `true`

Expected:
- Plan shows only creates (new subscription), and **no** `azurerm_dns_*` resources.

## 2) Apply (provision cluster + bootstrap ArgoCD)

Run workflow `.github/workflows/terraform-azure-apply.yaml` with:

- `confirm`: `apply-azure`
- `subscription_id`: `e4652c74-2582-4efc-b605-9d9ed2210a2a`
- `backend_resource_group`: `Ameide-TFState`
- `backend_key`: `azure-e4652c74.tfstate`
- `disable_dns`: `true`

This will:
- Create the Azure resources in the new subscription.
- Seed/clone secrets into the new cluster Key Vault (from `TF_ENV_SECRETS_KEYVAULT_NAME`).
- Publish runtime facts.
- Bootstrap ArgoCD and root ApplicationSets.

## 3) Verify “operations” (read-only)

After apply:

- ArgoCD:
  - `kubectl -n argocd get pods`
  - `kubectl -n argocd get applications,argoprojects,applicationsets`
- Observability (Grafana/Loki/Tempo):
  - `kubectl -n observability get pods`
  - `kubectl -n observability get svc`
  - Optional local access (no DNS): `kubectl -n observability port-forward svc/grafana 3000:80`

## Next step (still no customer impact)

Wire Front Door **preview** endpoint to the **new cluster** origin (port `8080`, `/healthz`) so we can validate the new cluster via `*.azurefd.net` without touching DNS/custom domains.

Tracking: `backlog/712-traffic-manager-first-redesign.md`

Implementation note:
- This wiring is implemented in the edge Terraform root (`infra/terraform/azure-edge`) and executed via the `Terraform Azure Edge Plan/Apply` workflows, with the origin subscription passed as input (no manual Azure edits).

## Notes

- Backup storage accounts are globally-unique by default: when no legacy account exists in the resource group, Terraform appends a subscription-derived suffix to the name (avoids collisions across subscriptions).
