# Local Secret Store Spike

**Created:** Nov 2025  
**Owner:** Platform DX / Infrastructure  
**Cross references:** backlog/338 (DevContainer bootstrap), backlog/348 (env/secret harmonization), backlog/355 (Layer 15 modularization), backlog/360 (secret guardrail enforcement/gap log)

## Context

Historically, local development clusters piggy-backed on Azure Key Vault (AKV) via the `azure-keyvault-secrets` Helm release. The bootstrap path ran `scripts/azure/sync-local-keyvault-secrets.sh`, pushing deterministic fixtures into a per-developer vault before External Secrets synced them into Kubernetes. That kept parity with staging/production but forced every developer workflow to authenticate against Azure and tolerate occasional throttling.

With the HashiCorp Vault rollout, local clusters now use the in-cluster `vault` chart and the `ameide-vault` `ClusterSecretStore`. This backlog captures the spike work that replaced the Azure dependency while preserving key names and workload templates.

## Status (Nov 2025)

- ✅ Local Helm values now point External Secrets at the in-cluster HashiCorp Vault store (`ClusterSecretStore/ameide-vault`) instead of the defunct `ameide-azure-kv`, so charts and smoke tests work offline.
- ✅ Keycloak, db-migrations, and other charts consume the same secret names after the switch; no template changes required.
- ⏳ Remaining work: streamline seeding tooling, document the new flow, and add validation helpers so developers notice missing Vault materialization early. Outstanding items align with backlog/360’s gap log (default secret validation, service-specific coverage, Layer 15 modularization).

## Goals

1. Reduce Azure coupling for local work while keeping chart templates and key names unchanged.
2. Provide a bootstrap path that works offline and finishes in under a minute.
3. Preserve the existing smoke tests and Helmfile story with minimal conditionals.
4. Document how secrets flow in local mode so teams can debug without Azure knowledge.

## Non-Goals

- Replacing Azure Key Vault in shared (staging/production) environments.
- Changing application code to read secrets differently.
- Shifting the source of truth for secret values away from infrastructure-owned repos.

## Approach

1. **Secret Store Abstraction**  
   - Introduce a `SecretProvider` interface (or Helmfile layer toggle) that chooses between AKV and a local store based on environment.
   - Ensure External Secrets still references the same `ClusterSecretStore` name (`ameide-vault`) by providing a local implementation that mimics AKV behavior.

2. **Local Backend Options**  
   - Adopt the existing HashiCorp Vault release (deployed in local clusters) as the default provider; keep file-backed alternatives on the backlog if Vault proves too heavy.
   - Prioritize deterministic, dependency-light tooling that can be vendored or prebuilt into the devcontainer.

3. **Sync Tooling**  
   - Replace `scripts/azure/sync-local-keyvault-secrets.sh` with a provider-agnostic bootstrapper that reads `scripts/azure/default-local-keyvault-secrets.json` and writes into Vault (or a future pluggable backend).  
   - Keep the existing presync hook in `infra/kubernetes/helmfiles/15-secrets-certificates.yaml` but branch by provider during the transition.

4. **Parity & Validation**  
   - Add a test helper that asserts the local store (Vault) exposes every key required by `infra/kubernetes/values/platform/azure-keyvault-secrets.yaml`.  
   - Ensure CI still runs against AKV so cloud paths remain covered.

## Workstreams

- [x] Design the provider abstraction (Helmfile values, Tilt args, or a CLI flag) and document configuration precedence.
- [x] Prototype at least two local backends and record trade-offs (HashiCorp Vault selected as default; file-backed option deferred).
- [ ] Build the provider-agnostic seeding script and update presync hook logic (feeds backlog/338 + backlog/360 default-secret validation).
- [ ] Update developer onboarding docs to describe the new bootstrap flow (coordinate with backlog/338 documentation refresh).
- [ ] Extend smoke tests or add a lightweight script that verifies key coverage in local mode (required to close backlog/360 gap #1).
- [ ] Plan the migration sequence (feature flag rollout, toggle default, rollback plan).

## Risks

- **Configuration drift:** Multiple stores may cause secret values to diverge; mitigate with a one-way sync from AKV fixtures and checksum validation.  
- **Complexity creep:** Too much abstraction could confuse contributors; keep environment switching explicit and well-documented.  
- **Security posture:** Local stores must remain git-ignored and scoped to developer machines; include guardrails in tooling.

## Open Questions

- Do we mandate a single local store (e.g., file-based) or allow multiple pluggable options?
- Should the store remain read-only fixtures, or can developers override secrets easily for testing?
- How do we surface validation errors when required keys are missing without Azure diagnostics?
- Can we reuse External Secrets Operator with a different provider, or is a native Kubernetes Secret sufficient locally?

## References

- `infra/kubernetes/helmfiles/15-secrets-certificates.yaml`
- `infra/kubernetes/scripts/azure/sync-local-keyvault-secrets.sh` (legacy; replaced by `scripts/vault/ensure-local-secrets.py` in devcontainer bootstrap)
- `scripts/azure/default-local-keyvault-secrets.json`
- `scripts/vault/ensure-local-secrets.py`
