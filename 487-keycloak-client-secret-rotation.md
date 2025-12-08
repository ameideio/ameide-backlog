# 487 – Keycloak OIDC Secret Rotation (Declarative Flow)

**Status**: Draft  
**Created**: 2025-12-08  
**Owner**: Platform/Auth  
**Related**: [426-keycloak-config-map.md](426-keycloak-config-map.md) · [462-secrets-origin-classification.md](462-secrets-origin-classification.md) · [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md) · [486-keycloak-admin-recovery.md](486-keycloak-admin-recovery.md) · [451-secrets-management.md](451-secrets-management.md) · [422-plausible-argocd-alignment.md](422-plausible-argocd-alignment.md)

---

## 1. Background

Keycloak client secrets (`platform-app`, `platform-app-master`, `argocd`, `k8s-dashboard`, future app clients) follow the **service-generated → client-patcher → Vault → ExternalSecret** pipeline captured in 426/462/485. Today the flow halts at Vault—operators still annotate ExternalSecrets (`force-refresh=$(date +%s)`) and restart Deployments manually (451, 486). This makes production recovery brittle and violates the “only Git changes the cluster” rule.

Pain points observed in Dec 2025:

- Vault has the canonical `platform-app-client-secret`, but `www-ameide-platform` pods kept the bootstrap placeholder until someone force-refreshed the ExternalSecret and restarted the Deployment.
- The runbooks in 426/486 explicitly tell operators to use `kubectl annotate externalsecret …` — this is not Argo-managed.
- There is no signal from client-patcher → Argo to tell dependent apps that secrets changed. The best we have is “rerun the realm app” and manually re-sync downstream apps.

We want a fully declarative, idempotent approach where a single Argo sync performs: “Keycloak generates secret → client-patcher writes Vault → ExternalSecret re-syncs → dependent Deployments roll.”

---

## 2. Goals

1. **Argo-only automation** – No external scripts or imperative annotations. Rotations happen because Argo syncs changed manifests.
2. **Idempotent secret propagation** – Every sync converges: ExternalSecrets reconcile, Deployments detect new secret versions via annotations/checksums, and pods restart exactly once when secrets change.
3. **Clear dependency graph** – client-patcher, ExternalSecret refresh, and downstream apps sit in ordered waves (per 447/387) so smoke tests see the correct state.
4. **Observability** – secret versions / hashes land in ConfigMaps or annotations so operators can diff and audit changes.

---

## 3. Proposed architecture

```
Keycloak (client secret) ──┐
                           ▼
platform-keycloak-realm (client-patcher: PreSync)
  ├─ writes secret value to Vault KV
  └─ emits rotation metadata ConfigMap (new)
            │
            ▼
platform-secret-rotation controller (new chart or Job)
  ├─ watches ConfigMap hash (GitOps managed)
  └─ patches ExternalSecret + dependent annotations
            │
            ▼
ExternalSecret (www-ameide-platform-auth-sync)
  ├─ `force-refresh` annotation managed in Git
  └─ status Ready → triggers secrets-smoke + app rollout
            │
            ▼
Deployment (www-ameide-platform)
  ├─ Pod template annotation includes rotation hash
  └─ rollout restart happens automatically during sync
```

### 3.1 Rotation metadata ConfigMap

- Add a template to `keycloak_realm` chart that writes the latest client secret digests following client-patcher completion.
- Example ConfigMap `keycloak-client-secret-versions` in `ameide-{env}` namespace:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: keycloak-client-secret-versions
  annotations:
    app.kubernetes.io/managed-by: keycloak-realm
  labels:
    rotation.ameide.io/component: keycloak
  data:
    platform-app: sha256:<digest>
    platform-app-master: sha256:<digest>
    argocd: sha256:<digest>
```

- Data source: client-patcher already retrieves each secret; we can compute `sha256` in the Job and `kubectl patch` the ConfigMap (or write via `kubectl apply -f -`). Because the ConfigMap content is deterministic per secret value, Argo diffing keeps us declarative.

### 3.2 ExternalSecret refresh

- ExternalSecrets stay pointed at the canonical Vault path (eventually `/active` in the blue/green layout) and rely on ESO’s `refreshInterval`/`refreshPolicy=Periodic` to pull updated values. No digest annotations are required—ESO simply syncs Vault → K8s Secret whenever the KV version changes.
- The rotation ConfigMap remains **telemetry only**. Argo ignores `.data` drift (diff customization) so the Job can patch digests without fighting Git. Smoke tests compare ConfigMap digests vs. the synced Secrets to catch any propagation stalls.
- If we ever need faster convergence, we can still toggle ESO’s `refreshPolicy: OnChange` and let Git bump a simple timestamp annotation, but the steady state should not require digests flowing back into Helm values.

### 3.3 Deployment rollout

- Pods that read the rotated Secret are annotated with `reloader.stakater.com/auto: "true"`. Stakater Reloader (deployed in §3.4) watches for Secret updates and issues rollouts automatically. This keeps Argo/Helm out of the restart path and lines up with the Git-only responsibility: we no longer add checksum annotations derived from runtime data.

### 3.4 Ordering & waves

- `platform-keycloak-realm` (wave 355) emits ConfigMap + secrets.
- `platform-secret-rotation-sync` (new component, wave 356) runs a lightweight Job or controller that reads ConfigMap and patches ExternalSecrets annotations. Alternatively, we skip this component and let the ExternalSecret template read the ConfigMap via downward API, but that’s clunky. Better: incorporate the `force-refresh` annotation into the chart template using the ConfigMap data (Argo manages both). When the ConfigMap changes, Argo updates the ExternalSecret manifest, which triggers the operator to re-sync.
- `platform-secrets-smoke` (wave 399) depends on ExternalSecret readiness; we add one more assertion that the `force-refresh` annotation matches the ConfigMap hash so we know the rotation propagated.
- `www-ameide-platform` (apps wave > 400) references the same ConfigMap hash for its Deployment annotation so rollouts follow automatically.

---

## 4. Implementation tasks

1. **Keycloak Realm chart**
   - Extend `client-patcher` script to compute `sha256` of each extracted secret (e.g., using `sha256sum <<< "$SECRET"`).
   - After writing to Vault, patch ConfigMap `keycloak-client-secret-versions` with the digest map. Template lives in repo so Argo owns metadata; data is updated by the Job.
   - Document this in backlog/426 §3.2.

2. **New shared values**
   - Add `_shared/platform/platform-secret-rotations.yaml` capturing which apps depend on which secret digests.

3. **ExternalSecret templates**
   - Ensure every ExternalSecret (www-ameide-platform, argocd, k8s-dashboard, plausible, pgadmin, backstage, …) references the new Vault KV layout (`…/<client>/<color>` with an `active` pointer once blue/green lands) and has a sane `refreshInterval`/`refreshPolicy`. No digest annotations—ESO simply polls Vault.

4. **Deployment restarts**
   - Annotate secret-consuming Deployments with `reloader.stakater.com/auto: "true"` (already in place for www-ameide-platform) and verify platform-reloader watches their namespace. No Helm checksum magic required.

5. **Automation for other apps**
   - Apply the same pattern to: `argocd`, `k8s-dashboard`, `plausible`, `backstage`, etc. Document in their backlogs (422, 477) that they now rely on rotation metadata.

6. **Runbooks**
   - Update 426, 486 to remove manual `kubectl annotate externalsecret` steps; replace with “bump the secret in Keycloak → sync platform-keycloak-realm → Argo propagates automatically.”

7. **Testing**
   - Add `secrets-smoke` assertions (per 362/360) that rotation digests match between ConfigMap and ExternalSecret annotation.

---

## 5. Implementation status (Dec 2025)

- ✅ `platform-keycloak-realm` now owns the rotation metadata: the chart renders `keycloak-client-secret-versions`, the client-patcher Job installs kubectl, records per-client SHA-256 digests, patches Vault, and merges those digests back into the ConfigMap via RBAC-scoped permissions.
- ✅ Dev overrides enable secret extraction + client reconciliation for every OIDC consumer (platform, argocd, k8s-dashboard, backstage, plausible, pgadmin) so Argo already emits the digests we need for downstream wiring.
- ✅ `platform-secrets-smoke` enforces that `www-ameide-platform-auth` is non-placeholder _and_ its digest matches the ConfigMap so regressions fail fast.
- ✅ A dedicated `platform-reloader` controller is deployed (wave 360) and `www-ameide-platform` carries the annotation, giving us automatic restarts when Secrets change.
- ✅ `www-ameide-platform` now relies on the pure Keycloak → Vault → ESO → Reloader path: ESO polls Vault, platform-reloader issues rollouts, and the digest ConfigMap stays observability-only.
- ⏳ Other apps (argocd, k8s-dashboard, plausible, pgadmin, backstage) still rely on the legacy “Vault only” path; they need the same annotation/checksum wiring once the shared values surface the digests.
- ⏳ Runbooks/scripts (`refresh-platform-auth-secret.sh`, 426/486) still describe the manual annotate/restart flow and must be updated once the declarative path is live.

---

## 5. Open questions

1. **ConfigMap writer permissions** – The client-patcher Job currently runs under `serviceAccountName: default`. We need to ensure it can `patch configmaps` in the namespace. Document RBAC update in backlog/485.
2. **Multiple environments** – Each environment manages its own ConfigMap; ApplicationSet must render environment-specific references.
3. **Future realm-per-tenant** – When we introduce realm-per-tenant (333), the client-patcher will produce many digests. The ConfigMap should be structured (e.g., `platform-app@realm`).

---

## 6. Deliverables & tracking

- [x] Extend client-patcher to write rotation ConfigMap (426/485 follow-up)
- [x] Update `www-ameide-platform` ExternalSecret + Deployment templates (this doc)
- [ ] Update `argocd`, `k8s-dashboard`, `plausible`, `backstage` charts
- [x] Add `secrets-smoke` validation (362 guardrail)
- [ ] Update operator runbooks (426, 486, 451)
- [ ] Close out manual script references once the automation lands
- [x] Deploy a namespace-wide reloader controller (`platform-reloader`) and annotate workloads (starting with `www-ameide-platform`) so pods restart automatically when ESO writes fresh secrets.

---

## 7. References

- [426-keycloak-config-map.md](426-keycloak-config-map.md) – Source of truth for Keycloak component wiring and client-patcher architecture.
- [462-secrets-origin-classification.md](462-secrets-origin-classification.md) – Secret authority taxonomy and existing fixes ensuring `AUTH_KEYCLOAK_SECRET` draws from `platform-app-client-secret`.
- [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md) – client-patcher idempotency, PreSync hook reasoning, and ensure-client logic.
- [486-keycloak-admin-recovery.md](486-keycloak-admin-recovery.md) – Highlights the manual steps (annotate ExternalSecret, restart apps) that this backlog aims to eliminate.
- [451-secrets-management.md](451-secrets-management.md) – Current operator commands for inspecting Vault and ExternalSecrets; reference for updating documentation once automation is in place.
- [422-plausible-argocd-alignment.md](422-plausible-argocd-alignment.md) – Environment-specific Plausible wiring (oauth2-proxy + Vault-backed Keycloak client) that now consumes the rotation/digest flow tracked here.

---

## 8. Vendor-aligned layering (Keycloak → Vault → ESO)

The long-term shape of this backlog needs to stay in lockstep with upstream guidance. We are treating this as a “reset the board” exercise with the following separation of responsibilities:

| Layer | Role | Notes / References |
|-------|------|--------------------|
| **Git / Argo CD** | Owns declarative configuration only (CRDs, CR specs, ExternalSecrets, Deployments, policies). | Argo manages manifests; runtime state/secret material is ignored via diff customization. |
| **Keycloak** | Authoritative source of clients + rotation policy. | Use the official Keycloak Operator for the instance (per [keycloak.org/operator/installation](https://www.keycloak.org/operator/installation)). Clients are declared via realm/client CRDs once upstream exposes them, or through our `client-patcher` controller until then. Client secret rotation leverages Keycloak’s native dual-secret feature when GA ([docs](https://www.keycloak.org/docs/latest/server_admin/index.html)). Bootstrap admin credentials are used exactly once; automation relies on a service account client with scoped `realm-management` roles ([docs](https://www.keycloak.org/docs/25.0.6/securing_apps/index.html)). |
| **Vault** | Canonical store for secret material and rotation history. | KV v2 paths follow a blue/green layout (`.../blue`, `.../green`, `.../active`) so we can atomically flip which secret is active, per HashiCorp’s static secret rotation guidance ([developer.hashicorp.com/well-architected-framework/...](https://developer.hashicorp.com/well-architected-framework/secure-systems/secrets/rotate-secrets)). Vault auth uses Kubernetes auth with scoped roles; no long-lived root tokens stored in K8s (per [HashiCorp production hardening](https://developer.hashicorp.com/vault/docs/concepts/production-hardening)). |
| **External Secrets Operator** | Copies Vault → K8s Secret. | `refreshPolicy` is either `Periodic` (short interval) or `OnChange` when driven by annotations from Git. ESO has no bespoke rotation logic; it simply syncs the value ([external-secrets.io](https://external-secrets.io/latest/api/externalsecret/)). |
| **Controllers (e.g., Stakater Reloader)** | Restart workloads that cannot hot-reload secrets. | Pods that mount secrets as volumes can reload in place; env var consumers are restarted by a controller watching Secrets/ConfigMaps. This keeps Argo out of the restart path. |
| **Workloads** | Consume secrets via environment variables or mounted files. | Deployments include hashes/annotations referencing the rotation metadata so that pod templates change whenever a new digest is published. |

### 8.1 Rotation flow (future state)

1. Operator or controller updates the client’s CR (or Keycloak regenerates the secret). Keycloak now exposes two secrets (main + secondary) per the vendor feature.
2. `client-patcher` (or successor) reads both secrets, writes them to Vault’s blue/green paths, and updates the `active` pointer.
3. ESO notices the Vault change (periodic poll or `force-sync` annotation written by Argo) and updates the Kubernetes Secret.
4. Reloader/controller patches the Deployment so new pods roll and pick up the secret.
5. After a grace period, Keycloak drops the old secret; Vault tombstones the retired colour.

### 8.2 Changes implied by the feedback

- **Bootstrap admin** – `bootstrap-admin user|service` should run only while Keycloak nodes are stopped (per [Bootstrapping and recovering an admin account](https://www.keycloak.org/server/bootstrap-admin-recovery)). We will bake that into the recovery runbook and future automation.
- **Vault access** – Replace the stored root token (`vault-auto-credentials`) with short-lived tokens issued via the Kubernetes auth method, scoped to the precise KV paths. This work will track under backlog/486 and this document.
- **Manual scripts** – Scripts like `scripts/refresh-platform-auth-secret.sh` are being removed; Argo + controllers will own refresh signalling end-to-end following the architecture above.
