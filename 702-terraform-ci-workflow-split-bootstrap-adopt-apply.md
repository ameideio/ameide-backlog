---
title: "702 – Terraform CI workflow split: bootstrap vs adopt(import) vs apply"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-19
suite: "terraform"
source_of_truth: false
---

# 702 – Terraform CI workflow split: bootstrap vs adopt(import) vs apply

## Problem

The Azure “apply” workflow currently performs multiple responsibilities:
- backend bootstrap (tfstate storage)
- import/adopt logic (for break-glass-created resources)
- apply
- post-apply sync (Key Vault + runtime facts)

This increases complexity and makes the “normal” apply path harder to reason about once the system is converged.

## Contract (target state)

Day-2 operation should be:
- `plan` → review → `apply` (no hidden migrations).

Migrations/adoption should be explicit:
- separate “adopt/import” workflow runs until plan is clean.

## Proposed workflows

1) **bootstrap-tf-backend-azure** (one-time / rare)
   - ensure backend RG/SA/container
   - assign backend RBAC

2) **terraform-azure-adopt** (manual / break-glass)
   - `terraform init`
   - import known resources (cluster, identities, nodepools, dns record sets)
   - run `terraform plan` and fail if drift remains (unless explicitly allowed)

3) **terraform-azure-apply** (normal)
   - `terraform init`
   - `terraform plan` → `terraform apply`
   - publish outputs/runtime facts
   - no imports

## Implementation notes (current repo)

- The “single-writer” day-2 path is `plan` → review → `apply`, with no import/adoption logic in the apply workflow.
- The break-glass path is explicit: run `terraform-azure-adopt` until plan is clean, then return to normal apply.
- Enabled environments are derived from `config/clusters/azure.yaml` and shared across CI scripts via `infra/scripts/enabled-azure-envs.sh` to avoid drift.

## Acceptance criteria

- Normal apply contains no “import_if_exists” logic.
- Adoption workflow produces a no-op plan as its exit criteria.
- The repo documents which workflow is used for which scenario (normal vs incident recovery).
