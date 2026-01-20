# 713 - Migration remediations (AKS recreate + GitOps convergence)

This backlog captures the concrete failure modes observed during the **AKS subscription recreate** and the **remediations** needed to keep GitOps convergence deterministic while we transition toward the traffic-manager-first architecture.

## Scope

- Environment: `dev` (primary), plus any mirrored issues in `staging` / `production`
- Cluster: recreated AKS in new subscription (`e4652c74-2582-4efc-b605-9d9ed2210a2a`)
- GitOps: Terraform-in-CI + ArgoCD multi-source (runtime facts) + ExternalSecrets/Vault

Related:
- `backlog/711-azure-subscription-recreate.md`
- `backlog/712-traffic-manager-first-redesign.md`
- `backlog/438-cert-manager-dns01-azure-workload-identity.md`
- `backlog/448-cert-manager-workload-identity.md`
- `backlog/444-terraform-v2-az-sub-prereq.md`

## Issues → remediations

### A) ArgoCD can’t render target state (submodule ref missing)

**Symptom**
- ArgoCD repo-server fails to fetch `backlog/` submodule commit: `upload-pack: not our ref ...`

**Remediation**
- Ensure `ameideio/ameide-backlog` contains the referenced commit.
- Add a guardrail: CI check that validates submodule commit exists in the configured remote before merging.

**Acceptance**
- Argo apps no longer show `Failed to load target state` due to submodules.

---

### B) Keycloak realm bootstrap deadlocks (hooks ordering)

**Symptoms**
- `platform-keycloak-realm-client-patcher` job fails (403) and never writes OIDC client secrets to Vault.
- Downstream workloads fail due to missing secrets (Coder, GitLab, others).
- User-facing error: `Invalid parameter: redirect_uri` on auth flows when expected redirect URIs aren’t registered.

**Root cause**
- Bootstrap job that creates the automation client (e.g. `platform-app-master`) did not run early enough (incorrect hook semantics), so patcher fell back to low-privilege credentials.

**Remediation**
- Make “master bootstrap” an ArgoCD Sync hook that runs before the patcher (explicit sync waves).

**Acceptance**
- Realm client patcher completes.
- ExternalSecrets materialize:
  - `Secret/keycloak-realm-oidc-clients`
  - `Secret/gitlab-oidc-provider`
- Redirect URI errors stop for expected apps.

---

### C) GitLab stuck / 500 due to failed/blocked migrations

**Symptoms**
- `https://gitlab.dev.ameide.io/` returns `500`
- `platform-gitlab-webservice` / `sidekiq` stuck in init waiting for DB initialization/migrations

**Root cause**
- Migrations job fails and never initializes the DB schema.
- Specific failure observed: GitLab DB config validation (`ci` vs `main` decomposition) rejects a single-DB setup unless `ci` shares DB with `main`.

**Remediations**
- Force re-run of the migrations Job by changing the job-name suffix hash input (annotation bump).
- For dev single-DB: map `global.psql.ci.database` → `gitlabhq_production` so migrations/schema load can proceed.

**Acceptance**
- Migrations job succeeds; DB has tables.
- GitLab webservice becomes Ready and returns non-500.

---

### D) Vault bootstrap CronJob fails to seed required secrets from Azure Key Vault (AKV)

**Symptoms**
- `foundation-vault-bootstrap` fails: `missing required Key Vault secret: ghcr-username`

**Root cause (observed)**
- Azure Workload Identity token exchange fails:
  - `AADSTS700211: No matching federated identity record found for presented assertion issuer ...`
- Meaning: runtime facts point the CronJob to a managed identity that does not have a federated credential matching the recreated cluster’s OIDC issuer.

**Remediations**
- Ensure runtime facts for each environment point to the correct:
  - `azure.keyVaultUri`
  - `azure.vaultBootstrapIdentityClientId`
- Add diagnostics (non-secret) to vault-bootstrap logs for token/secret fetch failures.
- Add CI validation: compare AKS `oidc_issuer_url` against federated credentials on the identity.

**Acceptance**
- `foundation-vault-bootstrap` succeeds continuously.
- Required secrets are written into Vault KV and consumed by ExternalSecrets reliably.

## Follow-ups (architecture alignment)

- Ensure “no DNS touch” bring-up (711) remains possible even while remediation work lands.
- Track public ingress decoupling per `backlog/712-traffic-manager-first-redesign.md` so cluster bootstrap does not depend on DNS-01 / wildcard TLS readiness.

