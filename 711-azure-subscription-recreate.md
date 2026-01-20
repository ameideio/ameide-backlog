# Azure: recreate AKS in a new subscription (no DNS touch)

Goal: provision a fresh AKS + platform stack in a new Azure subscription via GitOps (Terraform in CI + ArgoCD), without changing DNS zones/records.

## Inputs

- Old subscription: `68ba6806-178c-41f1-84ec-d7a839799be1`
- New subscription: `e4652c74-2582-4efc-b605-9d9ed2210a2a`
- DNS zones must not be modified during the first bring-up.

## What “no DNS touch” means here

- Run Terraform with `disable_dns=true` in CI.
- Terraform will not create/update DNS zones/records.
- Runtime facts will not publish `dnsManagedIdentityClientId`, so cert-manager uses self-signed issuers (no Azure DNS-01).

## One-time prerequisites (OIDC)

1. Ensure the GitHub Actions OIDC identity (repo var `AZURE_CLIENT_ID`) has at least:
   - `Owner` and `User Access Administrator` on the **new** subscription `e4652c74-...`
2. Ensure it can read the “source” Key Vault configured in repo var `TF_ENV_SECRETS_KEYVAULT_NAME` (for secret sync).

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

## Notes

- Backup storage accounts are globally-unique by default: when no legacy account exists in the resource group, Terraform appends a subscription-derived suffix to the name (avoids collisions across subscriptions).
