> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Secrets Ownership Refactor

**Created:** Nov 2025  
**Owner:** Platform DX / Infrastructure

## Motivation

The legacy `azure-keyvault-secrets` chart owned a monolithic list of ExternalSecrets. Every change to application credentials required editing that shared manifest, making the platform layer a bottleneck and inflating Helmfile diffs. Even after migrating to the Vault-backed `vault-secrets` bundle, the ownership problem persists—we want application teams to own their wiring while platform keeps the shared `ClusterSecretStore` and RBAC aligned.

## Goals

1. **Platform provides access, not inventory** – maintain the `ClusterSecretStore`, RBAC, and service account wiring, but stop declaring per-service ExternalSecret resources in the platform layer.
2. **Workloads own their secrets** – each chart (or the values file for that chart) defines the ExternalSecret objects it needs. This keeps the configuration close to the consumer and lets teams iterate without touching platform code.
3. **Zero runtime scripting** – continue to rely on declarative manifests (Helmfile/Tilt/Argo). No ad‑hoc jobs or scripts that create secrets at runtime.
4. **Consistent GitOps flow** – Helmfile sync and `helmfile test` continue to work layer by layer; CI/Tilt still gate on the smoke suites.

## Plan

1. **Retain the ClusterSecretStore only**
   - Update `infra/kubernetes/values/platform/vault-secrets.yaml` so the chart renders just the ClusterSecretStore definition plus any shared RBAC.
   - Remove the ExternalSecret entries from that chart (or split into a “examples” doc).
2. **Introduce workload-level ExternalSecrets**
   - For each service that currently relies on the shared manifest, add an ExternalSecret template or values-driven manifest inside that chart (e.g., `charts/platform/postgres_clusters`).
   - Ensure the manifests reference the existing store (`ameide-vault`) and are namespaced with the workload.
   - Use chart values to plug key names so we don’t hardcode them into templates.
3. **Document the pattern**
   - Update `infra/kubernetes/README.md` with a “How to consume Vault secrets” section.
   - Add a sample snippet to `docs/` showing the recommended ExternalSecret template.
4. **Migration path**
   - Create new ExternalSecret manifests for each workload before deleting the old centralized ones to avoid downtime.
   - Roll out environment by environment (local → staging → production) confirming Helm tests still pass.

## Tasks

- [ ] Update `charts/values/platform/vault-secrets.yaml` to remove per-secret definitions; keep only ClusterSecretStore + comments.
- [ ] Update `helmfiles/15-secrets-certificates.yaml` to ensure the simplified chart still deploys (Vault presync + ESO wiring).
- [ ] Inventory workloads that depend on the existing ExternalSecrets; create follow-up checklists for each (e.g., Postgres credentials, Plausible secrets).
- [ ] Add per-workload ExternalSecret templates/values.
  - [x] MinIO (`infra/kubernetes/values/infrastructure/minio.yaml:27`)
  - [x] Postgres clusters (`infra/kubernetes/charts/platform/postgres_clusters/templates/externalsecrets.yaml:1`)
  - [x] Plausible (`infra/kubernetes/charts/platform/plausible/templates/externalsecrets.yaml:1`)
  - [x] Prometheus / Alertmanager OAuth
  - [x] Platform app secrets (`platform-smoke`, etc.)
    - [x] platform (`infra/kubernetes/values/platform/platform.yaml`)
    - [x] graph (`infra/kubernetes/values/platform/graph.yaml`)
    - [x] inference (`infra/kubernetes/values/platform/inference.yaml`)
    - [x] agents / agents-runtime
    - [x] www frontends (`infra/kubernetes/values/platform/www-ameide*.yaml`)
    - [x] workflows (`infra/kubernetes/charts/platform/workflows/templates/externalsecret.yaml:1`)
    - [x] transformation (`infra/kubernetes/charts/platform/transformation/templates/externalsecret.yaml:1`)
    - [x] threads (`infra/kubernetes/charts/platform/threads/templates/externalsecret.yaml:1`)
    - [x] keycloak (moved ExternalSecret ownership into `charts/operators-config/keycloak_instance`; local overlay consumes Vault-sourced admin/master secrets)
  - [ ] Langfuse / LangGraph
    - [x] langfuse (`infra/kubernetes/values/platform/langfuse.yaml`)
    - [x] langfuse-bootstrap (`infra/kubernetes/charts/platform/langfuse-bootstrap/templates/externalsecrets.yaml:1`)
    - [x] runtime-agent-langgraph (retired; superseded by `apps-agents-runtime`)
- [ ] Extend smoke tests (where helpful) to verify the new per-workload secrets materialize.
- [ ] Update documentation and developer onboarding notes (Vault-first story).
- [ ] Remove the legacy `azure-keyvault-secrets` raw manifests once all workloads are migrated in hosted environments.
- [ ] Observe Flyway job using per-service secrets – helmfile selector run blocked (release not defined under current helmfiles)
- [ ] Re-run 40/45 data plane stack after cert-manager is healthy (pending)

## Open Questions

- Should we provide a shared helper template (e.g., `_externalsecret.tpl`) for consistent naming?  
  ✅ Added a scoped helper to `keycloak_instance` to render `remoteRef` blocks without duplicating YAML; evaluate lift-and-shift into a shared library once more charts need it.
- Do we want to enforce a naming scheme for Vault key paths so charts can infer key names (e.g., `service/secret-key`)?
- How do we track which workloads still rely on the centralized manifest during the migration? (Suggested: temporary checklist in this backlog entry, tick them off as we move each chart.)

## Lessons Learned (Nov 5 2025)

- Keeping database credentials in CNPG-owned secrets (`postgres-ameide-auth`) avoids double-encoding issues with ExternalSecrets; charts should reference the cluster-managed secret when it already exists instead of re-hydrating the same data.
- Supporting both map and string forms for `remoteRef` makes values files cleaner; helper templates prevent drift in optional fields like `property` and `decodingStrategy`.
- Clearing stuck Helm releases (`sh.helm.release.v1.*`) is still required when an install fails before annotations are applied—document the workaround in operator runbooks.
