> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# HashiCorp Vault Migration

**Created:** Nov 2025  
**Owner:** Platform DX / Infrastructure

## Context

Vault now ships as part of Helm layer 15, replacing the Azure Key Vault-backed `azure-keyvault-secrets` bundle. The `foundation-vault-core` release runs Vault (Raft HA for hosted envs, standalone for local), while the new `foundation-vault-bootstrap` chart deploys a CronJob that initializes/unseals Vault, configures Kubernetes auth, and seeds deterministic fixtures so External Secrets can reconcile without any out-of-band scripts. Documentation updates cover operator workflows and runbooks, but TLS hardening, production bootstrap automation, and Azure resource teardown are still pending.

## Goals

1. Provision Vault clusters per environment with highly-available storage (Raft) and TLS enabled.
2. Replace the Azure-backed `ClusterSecretStore` with a Vault-backed configuration for ESO while retaining identical secret keys for workloads.
3. Offer deterministic bootstrap paths (CI, dev) that load baseline secrets and create required policies/roles.
4. Design rotation, backup, and disaster recovery workflows that match or exceed current SLAs.
5. Document operational runbooks (initialization, unseal, upgrades, auth bootstrap) and integrate them with platform onboarding.

## Non-Goals

- Changing application-level secret consumers or key naming conventions.
- Introducing dynamic secret engines in phase one (optional follow-up once baseline parity lands).
- Deprecating ESO; it remains the abstraction for workloads to consume Vault data.

## Approach

1. **Vault Deployment**
   - ✅ Vendored the HashiCorp Vault Helm chart under `infra/kubernetes/charts/third_party/hashicorp/vault/<version>/vault-<version>.tgz` and added a dedicated `vault` release within layer 15.
   - ✅ Per-environment overrides toggle Raft HA (staging/production) and dev mode locally.
   - ⏳ Follow-up: enable TLS certificates once bootstrap automation is in place.

2. **Bootstrap & Auth**
 - ✅ `foundation-vault-bootstrap` CronJob automates initialization, unseal, Kubernetes auth wiring, and fixture seeding for every environment (local + shared) so Argo owns the entire lifecycle.
 - ✅ Auth/policy guidance captured in `docs/operations/vault.md`; the bootstrap job configures the `kubernetes` auth mount and grants `system:auth-delegator` to the ESO service account so `ClusterSecretStore` validates without manual intervention.
  - ✅ Devcontainer bootstrap (`.devcontainer/postCreateCommand.sh`) now stops after installing Argo; Vault readiness is handled entirely by the GitOps components.
   - ⏳ Production bootstrap automation (TLS, auto-unseal storage, secret owners) still needs implementation.

3. **ESO Integration**
   - ✅ `vault-secrets` bundle replaces the old Azure manifests, rendering the `ameide-vault` `ClusterSecretStore`.
   - ✅ Every chart/values file now references `ameide-vault` while retaining key names.
   - ✅ Smoke tests assert Vault availability and critical secret materialisation.
   - ✅ Platform/product-service charts now default `remoteRef.property=value`, so ESO can read KV v2 entries without Azure-specific assumptions.
   - ⏳ Product-service ExternalSecrets are still rendered by `vault-secrets`; move ownership into the individual Helm charts (Layer 50) so future syncs don't hit Helm adoption errors.

4. **Migration Plan**
   - ✅ Local environment migrated (ESO reads from Vault and smoke tests pass); hosted envs ready once bootstrap runbook executed.
   - ⏳ Plan dual-write/verification for staging & production cutover (coordinate with backlog #340).

5. **Operations & Observability**
   - ✅ Vault operations runbook published (`docs/operations/vault.md`).
   - ✅ Prometheus ServiceMonitor enabled via Helm values.
   - ⏳ Automate Raft snapshot backups and confirm alerting coverage.

## Workstreams

- [x] Add Vault Helm release with environment-specific values (replica counts, storage classes, TLS roadmap).
- [ ] Enable TLS for Vault (cert-manager issuance, env overrides, readiness gates).
- [ ] Author production bootstrap automation (init, unseal, policy seeding) and supporting documentation.
- [x] Replace Azure-specific scripts (`scripts/azure/*`) with Vault-focused tooling.
- [x] Update ESO configuration (`ClusterSecretStore` + references) across charts and helmfiles.
- [x] Validate smoke tests and layer selectors against the new store (local, CI templates).
- [x] Harden Kubernetes auth wiring (mount path, auth-delegator RBAC, fixture refresh).
- [ ] Migrate staging/production secrets to Vault and retire Azure Key Vault resources/permissions.
- [ ] Implement automated Raft snapshot backups and retention policy.
- [ ] Re-home application ExternalSecrets that `vault-secrets` currently owns (e.g., `platform-db-credentials-sync`, `graph-db-credentials-sync`, LangGraph Runtime) so Layer 50 charts can manage them directly without ownership conflicts during Helm sync.

## Risks

- **Operational complexity:** Vault introduces unseal key management and Raft maintenance; mitigate via automation and clear runbooks.
- **Migration downtime:** Secret drift during cutover could break workloads; plan dual reads and automated verification scripts.
- **Security posture:** Root tokens/unseal keys must be handled securely; integrate with existing secrets handling policies.

## Open Questions

- Where should unseal keys live (e.g., password vault, cloud KMS, Shamir split across SRE team)?
- Do we adopt auto-unseal (Azure Key Vault, HSM, or cloud KMS) for production?
- What cadence/process do we use for secret rotation and audit log retention post-migration?
- How do we validate dual-write/diff tooling during staging → production cutover?

## References

- `infra/kubernetes/helmfiles/15-secrets-certificates.yaml`
- `infra/kubernetes/values/operators/vault.yaml`
- `infra/kubernetes/values/platform/vault-secrets.yaml`
- `scripts/vault/ensure-local-secrets.py`
- `docs/operations/vault.md`
