# 487 – Keycloak OIDC Secret Rotation (Declarative Flow)

**Status**: Implemented  
**Created**: 2025-12-08  
**Last Updated**: 2026-01-18  
**Owner**: Platform/Auth  
**Related**: [426-keycloak-config-map.md](426-keycloak-config-map.md) · [462-secrets-origin-classification.md](462-secrets-origin-classification.md) · [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md) · [486-keycloak-admin-recovery.md](486-keycloak-admin-recovery.md) · [451-secrets-management.md](451-secrets-management.md) · [422-plausible-argocd-alignment.md](422-plausible-argocd-alignment.md)

---

## 1. Background

Keycloak client secrets (`platform-app`, `platform-app-master`, `argocd`, `kubernetes-dashboard`, `backstage`, `pgadmin`, `coder` (dev), …) follow the **service-generated → client-patcher → Vault → External Secrets Operator (ESO) → Kubernetes Secret → rollout** pipeline captured in 426/462/485.

This backlog was opened after incidents where the propagation chain was not converging end-to-end (Vault had a real secret, but Kubernetes still had placeholders), and operators “fixed it” by imperatively annotating ExternalSecrets or deleting hook Jobs. That violates the GitOps rule for shared cloud clusters.

As of 2026-01-18, the convergence chain is implemented and enforced via Argo hook Jobs + smoke tests:

1. `platform-keycloak-realm` runs `client-patcher` as an **Argo PreSync hook Job** to reconcile clients, extract/regenerate secrets, write Vault, and publish telemetry digests.
2. ESO syncs Vault → K8s Secrets on `refreshInterval`.
3. Stakater Reloader rolls workloads when Secrets change.
4. `platform-secrets-smoke` fails the sync if placeholders remain or if telemetry digests don’t match the materialized Secrets.

---

## 2. Goals

1. **Argo-only automation** – No imperative “fix the cluster” actions for OIDC secret issues.
2. **Deterministic convergence** – A Git change + Argo sync converges Keycloak → Vault → ESO → workloads.
3. **Clear enforcement** – `platform-secrets-smoke` gates sync on “no placeholders” + digest match.

---

## 3. Implemented architecture

```
Keycloak (authoritative secrets)
  │
  ▼
platform-keycloak-realm (client-patcher: Argo PreSync hook Job)
  ├─ ensures clients exist (485)
  ├─ extracts / regenerates secrets via Admin API
  ├─ writes secret values to Vault KV
  └─ patches ConfigMap/keycloak-client-secret-versions (sha256 digests)
            │
            ▼
External Secrets Operator (Vault → K8s Secret)
            │
            ▼
Deployments (Reloader-driven restarts on Secret change)
            │
            ▼
platform-secrets-smoke (Argo PostSync hook: validates placeholders + digests)
```

### 3.1 Rotation metadata ConfigMap

- `client-patcher` publishes SHA digests into `ConfigMap/keycloak-client-secret-versions`.
- The ConfigMap `.data` is **runtime-written telemetry**, not desired state. Argo CD ignores `.data` diffs for this ConfigMap so Git does not fight runtime output.
- `platform-secrets-smoke` compares telemetry digests to the materialized K8s Secrets to detect stuck ESO refreshes and placeholder leakage.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: keycloak-client-secret-versions
  data:
    platform-app: sha256:<digest>
    platform-app-master: sha256:<digest>
    argocd: sha256:<digest>
    kubernetes-dashboard: sha256:<digest>
    k8s-dashboard: sha256:<digest> # alias (same digest)
```

The dashboard key is `kubernetes-dashboard` (client ID), with a supported alias `k8s-dashboard` for compatibility with older naming.

### 3.2 ExternalSecret refresh

- ExternalSecrets rely on ESO’s `refreshInterval` to pull updated values from Vault.
- No imperative “force refresh” annotations are required as the standard mitigation path; `platform-secrets-smoke` includes retries to tolerate refresh latency.

### 3.3 Deployment rollout

- Pods that read the rotated Secret are annotated with `reloader.stakater.com/auto: "true"`. Stakater Reloader (deployed in §3.4) watches for Secret updates and issues rollouts automatically. This keeps Argo/Helm out of the restart path and lines up with the Git-only responsibility: we no longer add checksum annotations derived from runtime data.

### 3.4 Ordering & waves

- `platform-keycloak-realm` runs `client-patcher` as an Argo PreSync hook Job (ensures Keycloak/Vault/telemetry are updated before the rest of the sync).
- `platform-secrets-smoke` runs as an Argo PostSync hook Job and validates “no placeholders” + digest match.
- Secret consumers restart via Stakater Reloader on Secret change (no checksum annotations derived from runtime data).

---

## 4. Status

This secret propagation/rotation pipeline is now implemented and enforced by Argo hook Jobs (`client-patcher` + `platform-secrets-smoke`) and Stakater Reloader rollouts.

---

## 5. Mitigation path (GitOps, no manual cluster edits)

When an application reports `unauthorized_client` / “Invalid client credentials”:

1. Bump `clientPatcher.runId` for the affected environment in this repo.
2. Sync the environment’s `platform-keycloak-realm` Argo Application.
3. Wait for `platform-secrets-smoke` to succeed (it validates “no placeholders” + digest match).
4. Confirm the affected workload restarts via Stakater Reloader (no manual rollout restart).

---

## 6. References

- [426-keycloak-config-map.md](426-keycloak-config-map.md) – Keycloak component wiring and client-patcher architecture.
- [462-secrets-origin-classification.md](462-secrets-origin-classification.md) – Secret authority taxonomy.
- [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md) – client reconciliation + extraction model.
- [486-keycloak-admin-recovery.md](486-keycloak-admin-recovery.md) – Recovery posture and break-glass boundaries (GitOps-first).
- [451-secrets-management.md](451-secrets-management.md) – Vault/ESO architecture; GitOps-aligned troubleshooting guidance.

---

## 7. Vendor-aligned layering (Keycloak → Vault → ESO)

The long-term shape of this backlog needs to stay in lockstep with upstream guidance. We are treating this as a “reset the board” exercise with the following separation of responsibilities:

| Layer | Role | Notes / References |
|-------|------|--------------------|
| **Git / Argo CD** | Owns declarative configuration only (CRDs, CR specs, ExternalSecrets, Deployments, policies). | Argo manages manifests; runtime state/secret material is ignored via diff customization. |
| **Keycloak** | Authoritative source of clients + rotation policy. | Use the official Keycloak Operator for the instance (per [keycloak.org/operator/installation](https://www.keycloak.org/operator/installation)). Clients are declared via realm/client CRDs once upstream exposes them, or through our `client-patcher` controller until then. Client secret rotation leverages Keycloak’s native dual-secret feature when GA ([docs](https://www.keycloak.org/docs/latest/server_admin/index.html)). Bootstrap admin credentials are used exactly once; automation relies on a service account client with scoped `realm-management` roles ([docs](https://www.keycloak.org/docs/25.0.6/securing_apps/index.html)). |
| **Vault** | Canonical store for secret material and rotation history. | KV v2 paths follow a blue/green layout (`.../blue`, `.../green`, `.../active`) so we can atomically flip which secret is active, per HashiCorp’s static secret rotation guidance ([developer.hashicorp.com/well-architected-framework/...](https://developer.hashicorp.com/well-architected-framework/secure-systems/secrets/rotate-secrets)). Vault auth uses Kubernetes auth with scoped roles; no long-lived root tokens stored in K8s (per [HashiCorp production hardening](https://developer.hashicorp.com/vault/docs/concepts/production-hardening)). |
| **External Secrets Operator** | Copies Vault → K8s Secret. | `refreshPolicy` is either `Periodic` (short interval) or `OnChange` when driven by annotations from Git. ESO has no bespoke rotation logic; it simply syncs the value ([external-secrets.io](https://external-secrets.io/latest/api/externalsecret/)). |
| **Controllers (e.g., Stakater Reloader)** | Restart workloads that cannot hot-reload secrets. | Pods that mount secrets as volumes can reload in place; env var consumers are restarted by a controller watching Secrets/ConfigMaps. This keeps Argo out of the restart path. |
| **Workloads** | Consume secrets via environment variables or mounted files. | Deployments include hashes/annotations referencing the rotation metadata so that pod templates change whenever a new digest is published. |

### 7.1 Rotation flow (future state)

1. Operator or controller updates the client’s CR (or Keycloak regenerates the secret). Keycloak now exposes two secrets (main + secondary) per the vendor feature.
2. `client-patcher` (or successor) reads both secrets, writes them to Vault’s blue/green paths, and updates the `active` pointer.
3. ESO notices the Vault change (periodic poll or `force-sync` annotation written by Argo) and updates the Kubernetes Secret.
4. Reloader/controller patches the Deployment so new pods roll and pick up the secret.
5. After a grace period, Keycloak drops the old secret; Vault tombstones the retired colour.

### 7.2 Changes implied by the feedback

- **Bootstrap admin** – `bootstrap-admin user|service` should run only while Keycloak nodes are stopped (per [Bootstrapping and recovering an admin account](https://www.keycloak.org/server/bootstrap-admin-recovery)). We will bake that into the recovery runbook and future automation.
- **Vault access** – Replace the stored root token (`vault-auto-credentials`) with short-lived tokens issued via the Kubernetes auth method, scoped to the precise KV paths. This work will track under backlog/486 and this document.
- **Manual scripts** – Scripts like `scripts/refresh-platform-auth-secret.sh` are being removed; Argo + controllers will own refresh signalling end-to-end following the architecture above.
