---
title: "713 – Seeding Contract (Failfast, Deterministic, Self-Healing)"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-21
---

# 713 – Seeding Contract (Failfast, Deterministic, Self-Healing)

## Purpose

Define a single, platform-wide contract for anything we call “seeding” so that:

- a fresh cluster (or DB reset) converges to a usable state without manual UI steps,
- failures are visible and actionable (not silent fallback),
- re-runs are deterministic (no “it ran once and disappeared”),
- GitOps remains the only mutation path in managed environments.

This contract applies to **secrets**, **infra-derived runtime facts**, and **application database bootstrap state**.

## Cross-references

- Secrets architecture: `backlog/451-secrets-management.md`
- Secret authority taxonomy: `backlog/462-secrets-origin-classification.md`
- Coder first-user requirement (/setup): `backlog/626-ameide-devcontainer-service-coder.md`
- Cluster recreate -> Coder DB reset incident: `backlog/677-coder-dev-templates-disappeared-after-cluster-recreate.md`
- Workspaces WI + credential fan-out: `backlog/712-coder-workspaces-tasks-azure-workload-identity.md`
- Codex auth slots + fan-out: `backlog/613-codex-auth-json-secret.md`

## Definitions (what “seeding” is)

We treat “seeding” as any workflow that materializes **required runtime state** that is *not* committed to Git:

1. **External/third-party secrets** (authority: Azure Key Vault)  
   Example: `ghcr-token`, `openai-api-key`.
2. **Cluster-managed secrets** (authority: K8s/Helm/operator)  
   Example: CNPG-owned DB passwords, Helm-generated admin passwords.
3. **Service-generated secrets** (authority: the service)  
   Example: Keycloak client secrets generated in Keycloak.
4. **Application DB bootstrap state** (authority: application DB)  
   Example: “first user exists” for Coder, “admin user exists” for GitLab, schema exists for Temporal.

## Contract (non-negotiables)

### C1) Failfast on required inputs

If a seeding unit depends on a prerequisite, it must fail clearly and early when missing:

- missing secret input,
- missing runtime fact (issuer URL, resource group, etc.),
- missing network / auth injection (e.g. `AZURE_FEDERATED_TOKEN_FILE`).

Do not silently fall back to placeholders in managed clusters.

### C2) Idempotent, safe re-run

Every seeding unit must be safe to run repeatedly:

- “already exists” is success,
- drift correction is explicit and deterministic (no delete/recreate unless justified),
- secrets are never printed to logs.

### C3) Self-healing after resets

If the seeded state can disappear due to expected lifecycle events (cluster recreate, PVC reclaim, DB restore), it must be re-created automatically.

Preferred mechanism: **reconciler CronJob** (see Patterns).

### C4) Observable evidence

A seeding unit must leave evidence for at least a short window:

- Job/CronJob status, and logs that show *what* failed (without secret material).
- Avoid patterns that instantly delete success/failure evidence.

### C5) Clean wiring, minimal moving parts

- No optional “fallback modes” that make behavior ambiguous.
- Prefer environment overlays to enable/disable whole seeders (dev-only vs all envs).
- Keep secrets out of Terraform source and out of template parameters; secrets come from the secrets pipeline.

## Standard patterns (choose one)

### P1) Reconciler CronJob (preferred)

Use when the seeded state must exist continuously and can be lost.

Characteristics:

- runs on a short interval,
- exits 0 if state is already correct,
- bounded internal retry for dependency readiness,
- `concurrencyPolicy: Forbid`,
- `backoffLimit: 0` (internal retry loops are the retry),
- minimal job history (1 success, few failures).

Examples in repo:

- AKV→AKV curated sync: `foundation-vault-bootstrap` `*-keyvault-sync` CronJob
- Workspace federated credential reconciler: `foundation-vault-bootstrap` `*-coder-ws-fic` CronJob

### P2) ArgoCD hook Job with explicit rerun knob (secondary)

Use only when a one-time action is needed at install/upgrade time.

Requirements:

- include a **required** `runId` or checksum annotation so Git can force a rerun deterministically,
- do not hide failures (keep failed jobs),
- keep TTL long enough for humans to inspect.

Examples in repo:

- `platform/dev-data` seed job requires `job.runId`
- GitLab bootstrap includes `ameide.io/run-id`

### P3) Operator-managed (no custom seeding)

Use when the operator is the authority (CNPG, cert-manager). Any “seeding” is expressed as CRs and the operator reconciles.

### P4) Extraction job (service-generated secrets)

Use when a service is the authority and we need to publish derived secrets:

Keycloak client secrets: Keycloak → extractor Job → Vault → ESO → K8s Secret.

## “Done means” checklist for any new seeding unit

- It is **idempotent** (safe to run repeatedly).
- It is **failfast** (missing prerequisites produce a clear failure).
- It is **self-healing** if state can disappear.
- It has **observable evidence** for debugging.
- It fits the authority model in `backlog/462-secrets-origin-classification.md`.

## Applying this contract to Coder (/setup)

Problem class: **Application DB bootstrap state**.

Contract alignment:

- The “first admin user exists” must be ensured by a **reconciler CronJob** (P1), because Coder DB can be reset on cluster recreate (see 677).
- The bootstrap user inputs (`coder-bootstrap-admin-*`) remain sourced from the secrets pipeline (Vault/AKV), not from template parameters.
- Coder associates a login type per user. In SSO-only deployments, the seeding unit must ensure the seeded admin user’s `login_type` is `oidc` (not `password`), otherwise Keycloak login for that identity fails with “Incorrect login type”.
