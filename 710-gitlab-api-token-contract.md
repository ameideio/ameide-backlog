---
title: "710 — GitLab API access tokens (Vault → ExternalSecret → Secret)"
status: implemented
owners:
  - platform
created: 2026-01-19
related:
  - 694-elements-gitlab-v6.md
  - 695-gitlab-configuration-gitops.md
  - 451-secrets-management.md
---

# 710 — GitLab API access tokens (Vault → ExternalSecret → Secret)

GitLab tokens have historically been consumed on an ad-hoc basis (e.g., Backstage originally shipped its own `ExternalSecret`). This backlog item records the vendor-aligned contract for issuing GitLab API tokens to cluster services and scopes the GitOps work needed to make it a first-class platform capability.

## 1. Token classes (vendor guidance)

- GitLab advises avoiding Personal Access Tokens (PATs) when a scoped alternative exists. Prefer (least to most access): pipeline job tokens → **project access tokens** → **group access tokens**. [2][3][4]
- Deploy tokens are explicitly not valid for the GitLab REST API. [2]
- Keep tokens non-admin; the platform enforces governance, and GitLab CE provides storage + CI primitives.

## 2. Least-privilege service posture

- Default to one token **per consuming service** rather than a shared “platform” token. Blast radius stays limited to the projects/groups that service requires.
- Scope each token to read-only / read+write privileges that align with the service role (Reporter, Developer, Maintainer) and only enable the minimum scopes (`read_api` vs. `api`, `write_repository` only when required, etc.). [2]
- Track the owning service and project/group so rotation stories remain auditable.

## 3. GitOps delivery contract

- **Source of truth:** Vault KVv2 (`secret/gitlab/tokens/<env>/<service>`, key `value`). In shared cloud environments, populate this key via the existing secret pipeline (AKV/local → `vault-bootstrap` → Vault) using `azure.keyVault.mappedSecrets` when the AKV secret name cannot match the Vault key.
- **ArgoCD workload:** `foundation-gitlab-api-credentials` (new) renders a configurable set of `ExternalSecret` resources. Each entry:
  - lives in the consuming namespace,
  - points to Vault via `SecretStore/ameide-vault`,
  - writes a Kubernetes Secret named `gitlab-api-credentials` (overridable) with keys `GITLAB_API_URL` and `GITLAB_TOKEN`,
  - emits deterministic labels (`app.kubernetes.io/part-of: platform`, `ameide.io/layer: secrets-certificates`).
- **Consumption contract:** workloads mount the Secret and use:
  - `GITLAB_TOKEN` via the documented `PRIVATE-TOKEN: <token>` HTTP header for GitLab API calls; GitLab explicitly documents this header for PATs/project/group access tokens. [2]
  - `GITLAB_API_URL` for the cluster’s canonical REST endpoint (`https://gitlab.<env>.ameide.io/api/v4`).
- **Vault policy:** ESO roles (`ameide-vault` SecretStore) must be allowlisted for `secret/data/gitlab/tokens/<env>/*`.

## 4. Operational runbook (token creation)

1. Create a project or group access token in GitLab (UI or API). [3][4]
2. Copy the token once; GitLab only shows it during creation. [4]
3. Store the token in the external secret source (preferred) and let `vault-bootstrap` map it into Vault:
   - Put a flat key like `backstage-gitlab-token` into AKV (or local seed file).
   - Configure `azure.keyVault.mappedSecrets: [{from: backstage-gitlab-token, to: gitlab/tokens/<env>/backstage}]` so `vault-bootstrap` writes `secret/gitlab/tokens/<env>/backstage` with `value: "<token>"`.
4. Allow ESO to refresh (default `refreshInterval: 1h`). Verify `ExternalSecret/<service>-gitlab-api-token` reports `Ready=True` and `Secret/gitlab-api-credentials` exists in the target namespace.
5. Configure the consuming workload to read `GITLAB_TOKEN`/`GITLAB_API_URL`. No in-cluster automation should mint tokens by default; automated issuance would require a privileged bootstrap identity and is tracked separately.

## 5. Scope answers

- **“Is this already set?”** — Not previously. GitLab tokens were consumed ad-hoc (per-chart), without a declarative, cluster-wide contract.
- **“Can GitOps provide it?”** — Yes. Vault already supplies GitLab’s OIDC secrets via ExternalSecrets, so extending that pattern to API tokens keeps the source of truth consistent and reuses the existing ESO plumbing.

## 6. Status (2026-01-19) and next steps

- `foundation-gitlab-api-credentials` now renders per-service ExternalSecrets keyed off `gitlab/tokens/<env>/<service>` and produces `Secret/gitlab-api-credentials` (keys `GITLAB_API_URL`, `GITLAB_TOKEN`).
- Backstage consumes the shared Secret (`vaultKey: gitlab/tokens/<env>/backstage`), replacing the bespoke `backstage-gitlab-credentials` template.

**Next**

- Document the Vault policy updates and add a rotation runbook entry under `backlog/451-secrets-management.md`.
- Onboard additional consumers by adding them to the per-environment `foundation-gitlab-api-credentials` values files (and writing their Vault tokens).
- Treat shared “platform” tokens as exceptions; require explicit sign-off and documentation for any multi-service credential.

## References

[1]: https://external-secrets.io/latest/api/externalsecret/?utm_source=chatgpt.com "External Secrets Operator"
[2]: https://docs.gitlab.com/security/tokens/?utm_source=chatgpt.com "GitLab token overview"
[3]: https://docs.gitlab.com/user/permissions/?utm_source=chatgpt.com "GitLab roles and permissions"
[4]: https://docs.gitlab.com/user/group/settings/group_access_tokens/?utm_source=chatgpt.com "Group access tokens"
[5]: https://external-secrets.io/latest/provider/hashicorp-vault/?utm_source=chatgpt.com "External Secrets + HashiCorp Vault provider"
