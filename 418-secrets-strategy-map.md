# 418 – Secrets strategy map & gaps

Purpose: single view of our current secrets posture, with pointers to the source docs and open gaps to close.

## Guiding Principle: Secret Origin Classification

> **Secrets are classified by their origin, which determines their authoritative source.**
> See [462-secrets-origin-classification.md](./462-secrets-origin-classification.md) for the full taxonomy.

| Category | Authority | Examples |
|----------|-----------|----------|
| **External/Third-Party** | Azure Key Vault | GHCR tokens, API keys (Anthropic, OpenAI), Azure DNS credentials |
| **Cluster-Managed (Operator)** | Kubernetes Operator | CNPG database credentials, Redis auth |
| **Cluster-Managed (Helm)** | Helm `randAlphaNum` | MinIO root, Grafana admin, bootstrap secrets |
| **Cluster-Managed (Service)** | Service → client-patcher → Vault | Keycloak OIDC client secrets (`platform-app`, `argocd`, `k8s-dashboard`) |

**Key rule:** Cluster-managed secrets must NOT be fetched from Azure KV—they are generated in-cluster. Vault may mirror them for audit but never drives them.

## Current pillars (source docs)
- Baseline stack and bootstrap: `README.md` — DevContainer/bootstrap seeds Argo repo creds, registry pull secrets, and relies on Vault + External Secrets to reconcile everything else post-apply.
- Database credentials authority: `backlog/412-cnpg-owned-postgres-greds.md` — CNPG owns Postgres roles/passwords and emits the Kubernetes Secrets; Vault may mirror via PushSecret but never writes back; rotation is driven by Secret changes.
- Secret origin classification: `backlog/462-secrets-origin-classification.md` — Distinguishes external (Azure KV authority) vs cluster-managed (operator/Helm/service authority) secrets; documents the Keycloak client secret flow.
- Keycloak client-patcher: `backlog/426-keycloak-config-map.md` §3.2 — Documents the client-patcher Job architecture, Vault policy paths, and per-environment secretExtraction configuration.
- Secrets layer orchestration: `backlog/339-helmfile-refactor.md` — Layer 15 (“secrets & certificates”) runs `vault`, `vault-secrets` bundles, `cert-manager`, and `external-secrets` with per-environment `platform/15-secrets.values.yaml` toggles and a `secrets-smoke` job.
- Vault bootstrap readiness: `backlog/413-vault-bootstrap-readiness.md` — rollout bands keep Vault core at 150 and bootstrap at 155; `/sys/health` is relaxed to avoid Argo deadlock; bootstrap CronJob runs every minute.
- Service test guardrails: `backlog/347-service-tests-refactorings.md` — per-service ExternalSecrets are authoritative; test harnesses must run `scripts/vault/ensure-local-secrets.py` and consume the per-service templates (no inline Secrets/Azure KV).
- Build-time registry secrets: `backlog/391-buildkit-dev-registry-secrets.md` — GitHub/GHCR tokens are build-only via BuildKit secret mounts; no ARG/ENV token baking; Tilt/CI pass `--secret id=...` flags.

## Open gaps to address
- Document runtime secrets map: add an environment-scoped inventory (which Secrets come from Vault vs CNPG vs generators) to reduce spelunking; link it from Layer 15 values and service READMEs.
- Rotation runbooks: write concise rotation flows per secret class (CNPG-owned DB users, Vault-sourced API keys, TLS certs) and where to trigger (Secret update vs Vault path vs cert-manager).
- Drift detection: define how we detect stale ExternalSecrets/PushSecrets and CNPG↔Postgres divergence (e.g., ESO status alerts, CNPG role/secret parity checks).
- Testing gates: expand CI/Tilt checks to fail fast when required Secret stores are missing (e.g., enforce `ensure-local-secrets.py` ran, block builds if BuildKit tokens absent rather than surfacing 401s).
- Mirroring policy: codify when we push CNPG-managed Secrets into Vault (namespaces/paths, retention) and ensure apps treat Vault as read-only; add examples in the CNPG doc.
- Legacy footprints: audit for remaining Vault-authored DB creds or inline Kubernetes Secrets in charts/tests and replace with CNPG-owned or per-service ExternalSecrets.

## Next steps
- [ ] Publish the environment-level secrets inventory and link it from Layer 15 values and service READMEs.
- [ ] Add rotation runbooks per secret class and surface them alongside the owning chart/ExternalSecret.
- [ ] Add automated drift/health checks (ESO status alerts, CNPG role/secret parity) and wire them into observability.
- [ ] Harden CI/Tilt prechecks for missing BuildKit tokens and unsatisfied `ensure-local-secrets.py` pre-reqs.
- [ ] Write a short mirroring policy for CNPG→Vault PushSecret usage and attach it to `backlog/412-cnpg-owned-postgres-greds.md`.
- [ ] Complete the audit to remove any remaining Vault-authored DB users/secrets or inline Secrets in charts/tests.

## Snapshot
- Latest inventory: `backlog/418-secrets-strategy-map-snapshot.md` (auto-generated).
- Regenerate with `python3 scripts/secrets_inventory.py` (writes Markdown with embedded JSON by default; use `--output` for a raw `.json`).
- Script lives at `scripts/secrets_inventory.py` and scans repo YAML for Secret/ExternalSecret/PushSecret/SealedSecret resources; template files that cannot parse record best-effort metadata and list a parse error.

> **Note (2025-12-03):** `scripts/secrets_inventory.py` was planned but not yet implemented.
> For manual audit, use: `grep -r "kind: Secret\|kind: ExternalSecret\|kind: PushSecret" sources/`
