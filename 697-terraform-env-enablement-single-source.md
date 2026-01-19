---
title: "697 – Terraform env enablement: single source of truth"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-19
suite: "gitops"
source_of_truth: false
---

# 697 – Terraform env enablement: single source of truth

## Problem

ArgoCD can disable environments via `config/clusters/azure.yaml` (`enabled: "false"`), but Terraform can still provision
“disabled” environment infrastructure if Terraform uses its own hard-coded environment list.

This creates split-brain:
- ArgoCD: env disabled (no apps)
- Terraform: env infra still exists (cost + maintenance)

## Contract (target state)

- `config/clusters/azure.yaml` is the single source of truth for which envs are enabled.
- Terraform provisions per-environment Azure resources only for enabled envs.
- Terraform keeps stable Azure resource names by keying off namespace suffix:
  - `dev`, `staging`, `prod`
  - values env `production` maps to Terraform key `prod` (namespace is `ameide-prod`)

## Implementation (current repo)

- Terraform derives enabled env keys from `config/clusters/azure.yaml`:
  - `infra/terraform/azure/main.tf` (`local.tf_env_keys_enabled`)
  - `staging` disabled -> `local.tf_env_keys_enabled = ["dev","prod"]`
- Terraform apply workflow avoids importing resources for disabled envs:
  - `.github/workflows/terraform-azure-apply.yaml` (imports gated by enabled env list)

## Follow-ups (recommended)

- Make DNS child zone creation match env enablement (if desired), instead of treating DNS zone inventory as independent.
- Decide an explicit AKS upgrade/drift model for the shared cluster (auto-upgrade channel vs pinned versions).

