# backlog/379 – Argo setup notes (devcontainer + prod posture)

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

## Requirements
- Single HTTPS entrypoint on 443 across dev/staging/prod; no HTTP path.
- Devcontainer must auto-run bootstrap with the HTTPS port-forward (`0.0.0.0:8443 -> 443`), no manual commands; host access should work via this forward (or the forwarded VS Code port).
- Argo server runs with `server.insecure=false` and an HTTPS `configs.cm.url` so cookies are Secure.
- SSO via Keycloak (Dex) stays primary; local admin only for dev bootstrap convenience.

## Current state (by layer)
- **Chart inputs in use** (`gitops/ameide-gitops/sources/values/common/argocd.yaml`): pins `argo-cd@9.1.3` (AppVersion v3.2.0), disables admin by default, adds Dex OIDC to Keycloak (`argocd` client) with `/argocd-admin` → admin and `/argocd-readonly` → readonly RBAC, CMP Helmfile sidecar + metrics, ClusterIP service + ingress scaffold.
- **Environment overlays via bootstrap** (`gitops/ameide-gitops/environments/*/argocd/values/argocd-values.yaml` + `ops/argocd-cmd-params-cm.yaml`): dev now aligns to HTTPS (`server.insecure: "false"`, `configs.cm.url: https://localhost:8443` with only 30443 NodePort); staging/prod keep `server.insecure: "false"` and include `configs.cm.url` + ingress hosts/TLS.
- **Devcontainer/local values** (`gitops/ameide-gitops/sources/values/env/local/apps/platform/argocd.yaml`): scales down, disables ingress, mounts `/workspace`; defined for HTTP/non-Secure cookie but is not read by the bootstrap Helm install (env overlay wins).
- **Production/staging values (source-only)** (`gitops/ameide-gitops/sources/values/{production,staging}/apps/platform/argocd.yaml`): set HTTPS URLs/ingress hosts and `server.insecure: "false"`; env overlays mirror these for bootstrap.
- **Replica footprint (dev/local)**: shared base values default to 2 replicas for `controller`, `repoServer`, and `applicationSet`. Dev/local overlays now pin `controller.replicas=1`, `repoServer.replicas=1`, and `applicationSet.replicas=1`, keeping all Argo components enabled while minimizing pod count (1x controller, 1x repo-server, 1x ApplicationSet controller, 1x server, 1x redis).
- **Access**: devcontainer post-create already runs bootstrap with `--port-forward`, so you land with a single `0.0.0.0:8443 -> 443` HTTPS forward without manual commands. Envoy/Gateway serves `https://argocd.dev.ameide.io` once deployed.
- **Cookie / URL**: `configs.cm.url` controls Secure vs non-Secure session cookie; bootstrap now sets HTTPS for dev via the env overlay (`https://localhost:8443`). The HTTP override in the local values file remains unused.
- **Auth/OIDC**: Dex OIDC to Keycloak (`argocd` client) with `/argocd-admin` → admin and `/argocd-readonly` → readonly is now in the active base; admin disabled by default (dev may override).
- **Secrets**: `vault-secrets-argocd` ExternalSecret template exists for admin hash/server secret/Dex client secret; Vault content not verified here.
- **Keycloak/Dex health (dev)**: Realm imports are flaky. `master-realm` import intermittently fails with missing `realm-management` role mappings; `ameide-realm` import logs “Referenced client scope … doesn’t exist” but reports success, yet the realm isn’t visible in `kcadm get realms`. Dex currently uses in-cluster issuer `http://keycloak.ameide.svc.cluster.local:8080/realms/ameide`; hostAliases/patches were needed to bring Dex up temporarily and should be removed once imports are stable.

## Implementation status
- Done: Argo chart pinned to 9.1.3, k3d exposes 8443, staging/prod env trees exist under `gitops/ameide-gitops/environments/{staging,production}/argocd/**`, bootstrap can start 8443 port-forward when flagged.
- Pending: verify Vault secrets (Dex client/admin hash) exist in staging/production, confirm Keycloak groups, and ensure bootstrap uses these overlays when installing.
- Pending: fix Keycloak imports via GitOps (ensure required client scopes/roles exist and realm import jobs succeed) so Dex no longer needs manual realm creation or hostAlias patches.

## Decision: align to 443 + HTTPS everywhere
- Policy: single entrypoint on 443/HTTPS across dev/staging/prod; drop dev-only HTTP and rely on Secure cookies.
- Bootstrap: keep `--port-forward` but serve the HTTPS forward only; favor auto-enabling it in dev so the UI is reachable without manual commands.
- Values: flip dev `server.insecure` to `"false"` and set dev `configs.cm.url` to the HTTPS endpoint (`https://localhost:8443` until Envoy hosts are ready).
- Overrides: keep `ARGOCD_PORT_FORWARD_ADDRESS` overrideable for inner-loop edge cases, but default to `0.0.0.0:8443 -> 443`.

## Gaps for production readiness
- Vault content for Argo secrets (admin hash/Dex client) not validated.

## Suggested actions
1. Re-run bootstrap with `--port-forward` to pick up the dev HTTPS overlay (`server.insecure=false`, `configs.cm.url=https://localhost:8443`) and confirm cookie behavior.
2. Make `--port-forward` default to the HTTPS path (no 8080) and consider auto-enabling it in the dev flow so UI access “just works.”
3. Verify Vault secrets for admin hash/server key/Dex client secret and document Keycloak group provisioning.
4. Stabilize Keycloak realm imports: fix `master-realm` client/role mappings, ensure client scopes (openid/profile/email/offline_access) exist for `ameide-realm`, and drop hostAlias/Dex patches once the realm endpoint is reachable via service DNS.

## Troubleshooting (dev)
- If Argo apps fail to render, check ApplicationSet controller logs for template errors. A recent failure was a bad indent on Helm `valueFiles` in the foundation ApplicationSet; fix in Git, restart the ApplicationSet controller (`kubectl -n argocd rollout restart deployment argocd-applicationset-controller`), then annotate the ApplicationSet with `argocd.argoproj.io/refresh=normal`.
- For recurring drift on server-default fields, add `ignoreDifferences` in component manifests (and ensure the ApplicationSet passes them through) instead of live patches. Keep `ServerSideDiff` disabled unless strictly needed for CRDs.
