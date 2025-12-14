# Vault bootstrap readiness and rollout bands

> **Cross-References (Deployment Architecture Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | Per-environment Vault deployment via ameide ApplicationSet |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Phase 150 (vault-core) → Phase 155 (vault-bootstrap) |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Chart at `sources/charts/foundation/vault-bootstrap/` |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | Client-patcher writes to Vault |
>
> **Related**:
> - [451-secrets-management.md](451-secrets-management.md) – End-to-end secrets flow
> - [452-vault-rbac-isolation.md](452-vault-rbac-isolation.md) – Per-environment Vault RBAC
> - [387-argocd-waves.md](387-argocd-waves.md) – Original wave/band semantics

Context: We need Vault to stay in the post-runtime band (155) while avoiding a deadlock where Argo CD waits for Vault to be Ready before it will apply the bootstrap Job that initializes/unseals it.

Decision:
- Keep `foundation-vault-core` in rollout-phase 150 (runtime) and `foundation-vault-bootstrap` in rollout-phase 155 (post-runtime), preserving the band semantics from backlog/387.
- Relax Vault readiness to treat sealed/uninitialized as HTTP 200 by using `/v1/sys/health?standbyok=true&perfstandbyok=true&sealedcode=200&uninitcode=200` so Argo CD can advance into 155 and run the bootstrap.
- Increase the bootstrap CronJob cadence to every minute (`*/1 * * * *`) to minimize time-to-unseal.

Rationale:
- HashiCorp explicitly documents overriding `/sys/health` codes in their Helm chart to report Ready even when sealed/uninitialized for environments that rely on automation to init/unseal.
- This breaks the RollingSync deadlock without collapsing the band model (runtime at *50, bootstrap at *55).

Notes and guardrails:
- “Ready” now means “listening and can be initialized/unsealed,” not “serving.” Communicate this to operators; keep liveness stricter/off so a permanently sealed Vault still surfaces as unhealthy in other signals.
- Long-term cleaner path is auto-unseal (KMS/HSM) with strict health codes restored and bootstrap limited to auth/config/fixtures.
- CronJob name: `foundation-vault-bootstrap-vault-bootstrap`; manual trigger (for immediate run) stays `kubectl -n vault create job vault-bootstrap-manual --from=cronjob/foundation-vault-bootstrap-vault-bootstrap`.
- Avoid runtime downloads inside bootstrap jobs (e.g., `curl https://dl.k8s.io/...` for `kubectl`); prefer an initContainer/tooling image so bootstraps remain reproducible and don’t fail due to transient TLS/DNS issues (tracked under 519).
