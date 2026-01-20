---
title: "710 — GitLab API access tokens (Vault → ExternalSecret → Secret)"
status: implemented
owners:
  - platform
created: 2026-01-19
updated: 2026-01-20
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

**Implementation note (current platform posture)**

- The current GitOps implementation mints **dedicated service-user Personal Access Tokens** via `gitlab-rails runner` (toolbox) and constrains access via:
  - per-service users (non-admin),
  - minimal scopes,
  - group membership + access level,
  - mandatory expiry + rotation.
- This is vendor-compatible for API auth (`PRIVATE-TOKEN`) and avoids parking an admin PAT in-cluster. Moving to **group access tokens + self-rotate** is a future hardening step.

## 2. Least-privilege service posture

- Default to one token **per consuming service** rather than a shared “platform” token. Blast radius stays limited to the projects/groups that service requires.
- Scope each token to read-only / read+write privileges that align with the service role (Reporter, Developer, Maintainer) and only enable the minimum scopes (`read_api` vs. `api`, `write_repository` only when required, etc.). [2]
- Track the owning service and project/group so rotation stories remain auditable.

### Integration tests (repo creation)

For E2E/integration tests that create and delete GitLab projects (for example `backlog/704-v6-enterprise-repository-memory-e2e-implementation-plan.md`), use a dedicated token that is explicitly allowed to mutate GitLab:

- Prefer a **group access token** scoped to a dedicated group (e.g. `ameide-e2e`) rather than an admin PAT.
- Deliver it as a separate Kubernetes Secret (distinct `secretName`) so integration-test credentials do not collide with normal service credentials.
- Use `POST /projects` with `namespace_id=<group_id>` so test projects are created under the dedicated group (not under a personal namespace).
- Token needs `api` scope and sufficient permissions to create/delete projects in that group.
- Default posture: provision this writer token only in `local`/`dev` environments; do not ship a mutating test credential into `production` unless explicitly approved and justified.
- CI must verify the token works via an auth-required call (e.g., `GET ${GITLAB_API_URL}/user` using `PRIVATE-TOKEN`) before running the mutating flow.

## 3. GitOps delivery contract

- **Source of truth:** Vault KVv2 (`secret/gitlab/tokens/<env>/<service>`, key `value`).
- **Writer (standard platform tokens):** the GitLab workload owns minting/writing its standard service tokens (for example Backstage) and writes them into Vault KV. This keeps “token maintenance” inside the cluster/ArgoCD rather than Terraform/Bicep.
- **ArgoCD workload:** `foundation-gitlab-api-credentials` (new) renders a configurable set of `ExternalSecret` resources. Each entry:
  - lives in the consuming namespace,
  - points to Vault via `SecretStore/ameide-vault`,
  - writes a Kubernetes Secret named `gitlab-api-credentials` (overridable per consumer) with keys `GITLAB_API_URL` and `GITLAB_TOKEN`,
  - emits deterministic labels (`app.kubernetes.io/part-of: platform`, `ameide.io/layer: secrets-certificates`).
  - **must set `spec.target.creationPolicy: Owner`** so the Secret is created/owned by ESO.
  - **must set `spec.target.template.mergePolicy: Merge`** so templated keys (like `GITLAB_API_URL`) do not overwrite fetched keys (like `GITLAB_TOKEN`).
- **Consumption contract:** workloads mount the Secret and use:
  - `GITLAB_TOKEN` via the documented `PRIVATE-TOKEN: <token>` HTTP header for GitLab API calls; GitLab explicitly documents this header for PATs/project/group access tokens. [2]
  - `GITLAB_API_URL` for the cluster’s canonical REST endpoint (`https://gitlab.<env>.ameide.io/api/v4`).
- **Vault policy:** ESO roles (`ameide-vault` SecretStore) must be allowlisted for `secret/data/gitlab/tokens/<env>/*`.
- **No placeholders:** managed environments must not ship placeholder GitLab API tokens. Rollouts must fail if `Secret/gitlab-api-credentials` is missing or contains placeholder values, and GitLab smokes must verify token auth works (`GET /api/v4/user`).
- **Rotation:** GitLab enforces expirations; rotation requires coordinated consumer restart because most apps read tokens from env vars at startup. Automatic rotation is tracked work; do not revoke old tokens immediately without a cutover mechanism.

**Environment scoping**

- Baseline service tokens (e.g. Backstage) exist in all environments.
- The E2E/integration-test writer token exists in **dev/local only** and must not be provisioned in staging/production by default.

## 4. Operational runbook (token creation)

1. Ensure GitLab is synced and healthy in ArgoCD (`platform-gitlab` + `platform-gitlab-smoke`).
2. GitLab’s in-cluster token writer mints a least-privilege service token and writes it into Vault under `secret/gitlab/tokens/<env>/<service>` (key `value`, plus `token_id`/`expires_at`).
3. ESO materializes `Secret/gitlab-api-credentials` (or a consumer-specific secret name) in the consumer namespace.
4. Smokes verify the token is valid by calling `GET /api/v4/user` with `PRIVATE-TOKEN`.
5. For integration tests that mutate GitLab, use the dev/local-only writer secret (`gitlab-api-credentials-e2e`) and ensure tests create/cleanup their own projects (seedless).

## 5. Scope answers

- **“Is this already set?”** — Not previously. GitLab tokens were consumed ad-hoc (per-chart), without a declarative, cluster-wide contract.
- **“Can GitOps provide it?”** — Yes. Vault already supplies GitLab’s OIDC secrets via ExternalSecrets, so extending that pattern to API tokens keeps the source of truth consistent and reuses the existing ESO plumbing.

## 6. Status (2026-01-19) and next steps

- `foundation-gitlab-api-credentials` now renders per-service ExternalSecrets keyed off `gitlab/tokens/<env>/<service>` and produces `Secret/gitlab-api-credentials` (keys `GITLAB_API_URL`, `GITLAB_TOKEN`).
- Backstage consumes the shared Secret (`vaultKey: gitlab/tokens/<env>/backstage`), replacing the bespoke `backstage-gitlab-credentials` template.

**Next**

- Document the Vault policy updates and ensure rotation guidance is captured under `backlog/451-secrets-management.md`.
- Onboard additional consumers by adding them to the per-environment `foundation-gitlab-api-credentials` values files (and writing their Vault tokens).
- Treat shared “platform” tokens as exceptions; require explicit sign-off and documentation for any multi-service credential.
- For platform E2E/integration tests that must create repositories, provision a dedicated writer token (per service) with `api` scope and permissions to create/delete projects in the target namespace; tests must create and clean up their own GitLab projects rather than depending on pre-seeded data.

## References

[1]: https://external-secrets.io/latest/api/externalsecret/?utm_source=chatgpt.com "External Secrets Operator"
[2]: https://docs.gitlab.com/security/tokens/?utm_source=chatgpt.com "GitLab token overview"
[3]: https://docs.gitlab.com/user/permissions/?utm_source=chatgpt.com "GitLab roles and permissions"
[4]: https://docs.gitlab.com/user/group/settings/group_access_tokens/?utm_source=chatgpt.com "Group access tokens"
[5]: https://external-secrets.io/latest/provider/hashicorp-vault/?utm_source=chatgpt.com "External Secrets + HashiCorp Vault provider"
