# backlog/362-v2 – Unified Secret Guardrails & Vault Lifecycle

**Created:** Mar 2026 (refresh)  
**Owner:** Platform DX / Infrastructure  
**Supersedes:** `backlog/362-unified-secret-guardrails.md` (kept for history)
**Cross-references:**
- [462-secrets-origin-classification.md](./462-secrets-origin-classification.md) – Secret origin taxonomy and classification
- [412-cnpg-owned-postgres-greds.md](./412-cnpg-owned-postgres-greds.md) – CNPG credential ownership
- [451-secrets-management.md](./451-secrets-management.md) – Azure KV → Vault → K8s flow for external secrets
- [418-secrets-strategy-map.md](./418-secrets-strategy-map.md) – Strategy overview

---

## Why v2?
The original backlog consolidated work from 348/355/360, but real-world lessons (Vault cron bootstrap, vendor guidance, prod vs. dev needs) left the doc cluttered. This v2 keeps the hard requirements—service-owned secrets, a thin foundation boundary for shared primitives, and guardrail automation—while documenting the two bootstrap modes:

1. **Dev / CI / Ephemeral clusters** – Use the GitOps-managed `foundation-vault-bootstrap` CronJob to auto-init/unseal Vault, seed fixtures, and keep Tilt runs deterministic.
2. **Production / regulated clusters** – Follow vendor guidance: auto-unseal via cloud KMS/HSM or managed HSM, run a one-time, audited init, store recovery keys/root token outside K8s, and limit automation to lower-privilege configuration (auth mounts, policies, fixtures).

The doc also calls out vendor recommendations (HashiCorp and Big Bang) so reviewers know why production bootstrap differs.

---

## Guiding Principles
1. **Service-Scoped Ownership** – Each Helm chart (or namespace bootstrap chart) defines and documents its ExternalSecrets. Foundation-level components (waves 15‑21 in the ApplicationSet) only handle shared primitives such as the `ClusterSecretStore`, Vault wiring, and any explicitly approved shared bundles.
2. **Zero Inline Secrets** – Repository manifests never embed credentials. Charts require `existingSecret` references or define their own ExternalSecrets; `createSecret` toggles stay off.
3. **No Local Fallbacks** – Even for dev/Tilt overlays, do not disable ExternalSecrets or inject ad-hoc fixture Secrets. Every environment (including `local`) must reconcile the same Vault-managed secret objects so drift is caught early and parity stays tight.
4. **Fail Fast Everywhere** – db-migrations, bootstrap jobs, and smoke tests validate that every referenced secret exposes the required keys before connecting to databases or APIs.
5. **Bootstrap Split (Dev vs Prod)**  
   - *Dev / CI*: `foundation-vault-bootstrap` CronJob may auto-init/unseal and seed fixtures for convenience; credentials live in the `vault-auto-credentials` Secret.  
   - *Prod*: Vault is init/unsealed via out-of-band ceremony or KMS/HSM auto-unseal. Recovery keys/root token live outside Kubernetes. The GitOps job uses scoped tokens only to configure auth, mounts, and fixtures.
6. **Configuration as Code** – Vault auth methods, policies, and fixture maps are declared via GitOps (Helm values, CronJob ConfigMap). No manual `vault write` drift.
7. **Automated Enforcement (CI enabled)** – Helm lint/template + `scripts/infra/validate-hardened-charts.sh` run in CI (`.github/workflows/ci-helm.yml`) and must also run in Tilt/locally before merge.
8. **Shared Argo Health Source** – Argo health customizations live in `gitops/ameide-gitops/sources/values/common/argocd.yaml`; environment overlays stay empty so fail-fast gating remains consistent across clusters.

---

## Objectives
1. **Close Remaining Service Gaps** – Keep the service tracker current; ensure every chart consumes secrets via `existingSecret` or chart-owned ExternalSecret.
2. **Foundation Secret Boundary** – Keep the shared boundary minimal (vault-secret-store + any vetted cross-cutting bundles). Optional services must ship their own ExternalSecrets directly inside the service chart instead of relying on a monolithic layer.
3. **Vault Bootstrap Strategy** –  
   - Document the dev CronJob flow (idempotent, fixture-driven).  
   - Define the production bootstrap plan: auto-unseal via KMS/HSM, manual init ceremony, external storage for recovery keys, short-lived bootstrap tokens.
4. **Unified Fixture Story** – Keep the CronJob fixture ConfigMap (and per-environment overrides) aligned with every ExternalSecret key, including integration suites.
5. **Guardrail Automation** – Keep `scripts/infra/bootstrap-db-migrations-secret.sh` enforcing secrets before Flyway, `secrets-smoke` waiting for ExternalSecrets, and `scripts/validate-hardened-charts.sh` enforced via CI/Tilt.
6. **Documentation & Runbooks** – Every chart README describes secret ownership, rotation, and how to trigger guardrails (CronJob job, `kubectl annotate externalsecret …`).

---

## Workstreams
### 1. Service Alignment (tracker reused from v1)
| Service | Status | Notes |
| --- | --- | --- |
| agents | ✅ (Nov 2025) | Guardrails + README done. |
| agents-runtime | ✅ (Nov 2025) | Same as above. |
| db-migrations | ✅ (Mar 2026) | Data-plane bundle + bootstrap script enforce prerequisites. |
| graph | ✅ (Mar 2026) | Chart requires `graph-db-credentials`; README updated. |
| platform | ✅ (Apr 2026) | Chart requires `platform-db-credentials`; README updated. |
| transformation | ✅ (Mar 2026) | Integration secrets + rotation doc in place. |
| threads | ✅ (Mar 2026) | Vault-only credentials + rotation doc. |
| workflows-runtime | ✅ (Mar 2026) | README describes rotation + guardrails. |
| inference / inference-gateway / www surface / etc. | ✅ (Mar-Apr 2026) | Each README references Vault store + CronJob flow. |
| pgadmin | ✅ (Nov 2025 refresh) | Chart uses the same Vault-managed ExternalSecret + bootstrap guardrails as every other service; no local overrides or inline credentials remain. |
| minio | ✅ (Nov 2025) | Chart-owned ExternalSecrets pull root creds + service-user configs from `ameide-vault`; provisioning job now waits on `minio-service-users` instead of inline secrets. |
| workrequests-runner | ✅ (Dec 2025) | Service-owned ExternalSecrets templates: `workrequests-github-token` (`ghcr-token`), `workrequests-domain-token`, `workrequests-minio-credentials`; Vault fixtures: `workrequests-domain-token`, `workrequests-minio-access-key`, `workrequests-minio-secret-key`. Note: `local`/`dev` overlays currently set `secrets.enabled=false` for the workbench scaffolding; enable when we want Vault-synced creds in-cluster. |
| Remaining (list new services here) | ⏳ | Triage new services as they onboard. |

### 2. Foundation Secret Boundary
- Document which foundation components still manage shared secrets (currently `foundation-vault-secret-store` in GitOps) and keep that list minimal.
- Capture any shared bundles that must stick around (e.g., integration/test helpers) and migrate them into explicit `foundation/secrets/<name>` components when needed.
- Ensure every other secret is sourced via a service-owned ExternalSecret template (OAuth proxies, Grafana, pgAdmin, etc.); when gaps are discovered, update the owning chart instead of re-introducing a monolithic layer.
- Remove obsolete alias resources (done) and keep this backlog as the approval gate for any new shared bundle.
- **Action – ClickHouse auth bundle:** ✅ `foundation-vault-secrets-observability` now renders via `foundation/common/raw-manifests` and Argo gates `data-clickhouse` with a preflight secret check. Future work: convert the precheck RBAC/Job into Argo hooks everywhere so the Application stays green after the hook finishes.
- **Action – Temporal DB credentials (Nov 2025):** superseded. Temporal DB/visibility secrets are now CNPG-managed; the Vault-authored bundle was removed. Runbooks should note CNPG as the authority and treat any Vault copy as a mirror only.
- **Action – Platform DB bundles (Nov 2025):** Added `foundation-vault-secrets-platform` so all platform DB credentials (`platform`, `agents`, `agents-runtime`, `graph`, `threads`, `transformation`, `workflows`) originate from Layer 15. Disable the duplicate ExternalSecrets in the service charts (pgAdmin, Keycloak, Langfuse bootstrap, platform-postgres-clusters) so ownership conflicts disappear.
- **Action – Keycloak secrets (Nov 2026):** `foundation-vault-secrets-platform` now provisions the Keycloak DB credentials, bootstrap admin, master admin, and platform client secrets for every environment—including local—so the chart simply references existing Secrets and local overlays no longer need to re-enable chart-managed ExternalSecrets or override the `ClusterSecretStore`. The Keycloak admin service-account credentials (`keycloak-admin-sa-client-id` / `...-secret`) are also stored in Vault and synced via `ExternalSecret/keycloak-admin-sa-sync`, keeping the client-patcher service-account auth fully declarative.
- **Action – Tempo S3 credentials (Nov 2026):** The Tempo ExternalSecret now sources the MinIO access key/secret via direct `data.remoteRef` entries, removing inline template braces that previously made Argo fail fast while keeping the object-store credentials in Vault.

### 3. Vault Bootstrap
#### 3a. Dev / CI / Ephemeral
- `foundation-vault-bootstrap` CronJob:
  - Waits for Vault pod, runs `vault operator init/unseal` if needed, stores root + unseal key in `vault-auto-credentials` Secret (dev only).
  - Enables `secret/` kv-v2, configures K8s auth, writes fixture map from values (shared + per-env overrides).
  - Seeds deterministic secrets referenced by the foundation boundary (vault-secret-store + any approved shared bundles) and integration suites.
  - Can be triggered manually via `kubectl -n vault create job … --from=cronjob/foundation-vault-bootstrap-vault-bootstrap`.
#### 3b. Production / Regulated
- **Auto-Unseal via KMS/HSM:** configure Vault `seal` stanza to use cloud KMS or managed HSM. No unseal keys reside in K8s.
- **One-time Init Ceremony:** run `vault operator init` from a secure environment; distribute Shamir recovery keys/root token via approved channels; revoke root after bootstrap.
- **Scoped Automation:** GitOps job (or Terraform) uses a limited bootstrap token (provisioned via KMS secrets manager or out-of-band) to enable auth mounts, policies, and fixtures; no long-lived root token stored in the cluster.
- **Action Item:** design + implement this prod flow, then mark CronJob as dev-only in prod overlays (disable auto-init/unseal, rely on KMS). Track progress here.

### 4. Automation & Guardrails
- Keep `scripts/infra/bootstrap-db-migrations-secret.sh` enforced via Tilt resource `infra:db-secrets-bootstrap`.
- `secrets-smoke` waits up to 5 minutes (configurable env) for ExternalSecrets.
- `scripts/validate-hardened-charts.sh` runs in CI via `.github/workflows/ci-helm.yml` and remains required locally + Tilt for fast feedback.

#### CLI Verification (Implemented)

The core repository enforces a lightweight secret guardrail as part of Phase 0 of the repo-wide front door:

```bash
ameide test
```

What it enforces today:
- Forbids `kind: SealedSecret`
- Scans YAML/JSON manifests under core + GitOps roots and fails if a Kubernetes `Secret` contains inline credential-like keys (e.g., `password`, `token`, `clientSecret`, `privateKey`) with literal values (allows Helm templating `{{ ... }}` and env placeholders like `${VAR}`)

What it does not enforce yet (still required by this backlog’s target state):
- ExternalSecret/Vault wiring correctness (store refs, cross-namespace policies)
- Secret reference resolution (every `existingSecret` exists and exposes required keys)
- Naming/ownership policies beyond the inline-secret guardrail

### 5. Documentation & Communication
- Each service README includes:
  - Secret ownership (which ExternalSecret / existingSecret).
  - Rotation steps (update Vault key + annotate ExternalSecret + CronJob note).
  - Guardrail references (db-migrations, `infra:db-secrets-bootstrap`, CronJob job).
- Old backlog (v1) now states “See v2 for updates”.

### Targeted remediation – data-clickhouse
- **Secret provenance:** ✅ `foundation/secrets/vault-secrets-observability` ships via `foundation/common/raw-manifests`, and a Sync hook Job now blocks `data-clickhouse` until `clickhouse-auth` exists. Next step: convert the helper RBAC objects into hooks (or a dedicated component) so they cleanly disappear after success.
- **Value layering:** Provide per-environment overrides (for dev/staging/prod) and stop applying the `local` overlay globally; otherwise managed clusters inherit the `local-path` storage class and indefinitely Pending PVCs. ApplicationSets should render `_shared → <env> → local` where `<env>` is optional.
- **Sync options:** The `component.yaml` already lists `SkipDryRunOnMissingResource=true`, but the `data-services` ApplicationSet drops component-level `syncOptions`. Update the template to merge them so the ClickHouse Application can skip dry-run on custom resources while the Altinity CRDs roll out.

---

## Risks & Mitigations
| Risk | Mitigation |
| --- | --- |
| Wave ordering regressions when optional bundles flip | Maintain ApplicationSet wave ordering (waves 15‑21) for foundation secrets + operators, and run infra health-check after changes. |
| ClickHouse cluster loops because required secret missing | Ship the `vault-secrets-observability` component ahead of `data-clickhouse`, then re-run the `data-plane-core` smoke test so Argo refuses to sync ClickHouse until the Bundle is Healthy. |
| Dev CronJob pattern used in production | Document in this backlog + Argo values that auto-init/unseal + storing root/unseal keys in K8s is dev-only. Production overlays must disable CronJob init and rely on KMS auto-unseal & external key storage. |
| Vault fixture drift | Treat the CronJob ConfigMap as the single source; PRs that add ExternalSecrets must add fixture entries and note it in this backlog. |
| CI noise when new secrets appear | CI now templates the hardened charts; keep CronJob fixture entries + environment overrides current before merging to avoid failures, and continue running `scripts/validate-hardened-charts.sh` locally/Tilt. |
| Root token misuse | Production plan mandates revoking root after bootstrap and using scoped tokens; track implementation as part of Workstream 3b. |

Vendor references: HashiCorp Kubernetes guide, seal best practices, production hardening docs, and Big Bang’s warning that auto-init jobs storing root/unseal keys in a K8s Secret are dev-only.

---

## Flow (Dev / CI)
```
foundation-vault-bootstrap CronJob
    ↓ (init/unseal in dev, seeds fixtures)
Vault kv/secret (platform-db-url, www-ameide-auth-secret, etc.)
    ↓ (ClusterSecretStore ameide-vault)
Foundation secret primitives (vault-secret-store + any approved shared bundles)
    ↓
Namespace Secrets (platform-db-credentials, agents-db-secret, …)
    ↓
Service charts via existingSecret / extraSecretRefs
    ↓
db-migrations guardrail + runtime Pods
```

## Flow (Production target)
```
One-time init (secure context) + KMS auto-unseal
    ↓
Vault unseals via KMS; recovery keys + root token stored outside K8s
    ↓
GitOps/Terraform bootstrap job (scoped token):
  - enable auth mounts
  - configure policies
  - write fixture data
    ↓
Same foundation boundary + per-chart flows as dev (ClusterSecretStore, ExternalSecrets, guardrails)
```

---

## Natural Next Steps
1. **Mirror CronJob fixtures to production-safe tooling** – maintain the shared fixture YAML so both dev CronJob and future Terraform flows consume the same map.
2. **Implement production bootstrap plan** – design KMS seal stanza, init ceremony, secret storage, and GitOps token rotation. Track tasks under Workstream 3b.
3. **Keep CI guardrails green** and extend coverage as new services/onboarding flows land.

---

## Historical Note
`backlog/362-unified-secret-guardrails.md` remains for audit trail (covers Nov 2025 work). New work must reference this v2 document.
