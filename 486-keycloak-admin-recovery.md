# 486 – Keycloak Admin Recovery

**Status**: Implemented (GitOps-first posture)  
**Created**: 2025-12-07  
**Last Updated**: 2026-01-18  
**Priority**: P0 (Blocks Keycloak realm operations)  
**Related**: [426-keycloak-config-map.md](426-keycloak-config-map.md), [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md), [462-secrets-origin-classification.md](462-secrets-origin-classification.md), [487-keycloak-client-secret-rotation.md](487-keycloak-client-secret-rotation.md)

---

## 1. When to use this

Use this runbook when the `platform-keycloak-realm` `client-patcher` hook Job cannot authenticate to the Keycloak Admin API (commonly surfaced as `invalid_grant` / `401 Unauthorized` / `403 Forbidden`). This blocks:

- OIDC client reconciliation (485)
- secret extraction to Vault (487)
- downstream ESO → Kubernetes Secret updates

---

## 2. As-running auth model

`client-patcher` authenticates using:

1. **Primary**: a master-realm service-account client credential from `clientPatcher.loginSecretRef` (typically `Secret/keycloak-admin-sa`).
2. **Fallback**: bootstrap admin authentication (break-glass) to avoid deadlocks when the service-account path is broken.

### Vendor behavior (bootstrap admin is one-time)

Per Keycloak vendor docs, bootstrap admin credentials are only used when the master realm is created. After that point, changing `KC_BOOTSTRAP_ADMIN_*` or `Secret/keycloak-bootstrap-admin` does not update the stored admin credentials.

---

## 3. GitOps mitigation path (shared AKS clusters)

1. **Check Argo application health** (read-only):
   - `{env}-platform-keycloak-realm` hook status and recent hook logs
   - `{env}-platform-secrets-smoke` hook status (this is the enforcement signal)
2. **Force a clean client-patcher run** (Git):
   - bump `clientPatcher.runId` in `sources/values/env/{env}/platform/platform-keycloak-realm.yaml`
   - sync `{env}-platform-keycloak-realm` in Argo CD
3. **Wait for convergence**:
   - `platform-secrets-smoke` must succeed (it validates “no placeholders” + digest match for Keycloak-rotated secrets)
4. **Confirm downstream pickup**:
   - secret consumers restart via Stakater Reloader (no manual rollout restart)

---

## 4. If both auth paths fail (break-glass boundary)

If both service-account auth and fallback auth fail, treat this as an incident.

- **Cloud / shared clusters**: do not run ad-hoc `kubectl exec` or `kubectl apply` as a “fix”. Use a GitOps-owned recovery hook Job (or an approved CI workflow) and record the remediation as a Git change.
- **Local exception**: for a disposable local cluster, you may use vendor `kc.sh bootstrap-admin …` commands for debugging, but do not carry this pattern into cloud operations.

---

## 5. Prevention

- Keep the master management client ID aligned with Keycloak defaults (`realm-management`) and ensure the automation client has the required realm-management roles.
- Keep `clientPatcher.runId` as the deterministic re-run knob (Git-only).
- Keep `platform-secrets-smoke` gating enabled so placeholder secrets cannot silently persist.

---

## 6. References

- [Keycloak Bootstrap Admin Recovery](https://www.keycloak.org/server/bootstrap-admin-recovery)
- [Keycloak All Config](https://www.keycloak.org/server/all-config)
- [RHBK Operator Guide](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.2/html-single/operator_guide/)
