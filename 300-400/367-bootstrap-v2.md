# backlog/367 – Bootstrap v2

> **⚠️ BOOTSTRAP MOVED TO `ameideio/ameide-gitops` REPO**
>
> As part of [435-remote-first-development.md](435-remote-first-development.md), the bootstrap CLI
> and library have been moved to the `ameideio/ameide-gitops` repository:
> - `tools/bootstrap/bootstrap-v2.sh` → `ameide-gitops/bootstrap/bootstrap.sh`
> - `tools/bootstrap/lib/*` → `ameide-gitops/bootstrap/lib/*`
> - `infra/environments/*/bootstrap.yaml` → `ameide-gitops/bootstrap/configs/*.yaml`
>
> This document remains as historical design reference.

> **Related documents:**
> - [435-remote-first-development.md](435-remote-first-development.md) – **NEW**: Remote-first development architecture
> - [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md) – DevContainer bootstrap (superseded)
> - [432-devcontainer-modes-offline-online.md](432-devcontainer-modes-offline-online.md) – DevContainer modes (retired)
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) – Environment naming standardization

## Dual bootstrap responsibilities

We now operate **two complementary bootstrap flows**:

1. **GitOps / cluster bootstrap (this document, now housed in `ameideio/ameide-gitops`).**
   `ameide-gitops/bootstrap/bootstrap.sh` installs Argo CD, seeds repo/registry secrets, applies the RollingSync ApplicationSet, and validates cluster health for every environment (k3d, dev AKS, staging, prod). This is the script CI/CD and platform operators run after Bicep provisions an AKS cluster, and it remains the canonical path for converging GitOps resources.

2. **Developer bootstrap (lives in `ameideio/ameide`).**
   The application repo focuses on inner-loop ergonomics: `.devcontainer/postCreate.sh` invokes `tools/dev/bootstrap-contexts.sh`, which refreshes AKS credentials, sets predictable `kubectl` contexts, writes Telepresence defaults, and logs the `argocd` CLI into the shared control plane via a managed port-forward. This bootstrap never installs Argo or Helm charts; it simply prepares a DevContainer session to talk to the already-bootstrapped remote cluster.

Whenever the docs below mention `tools/bootstrap/bootstrap-v2.sh`, substitute the new location (`ameide-gitops/bootstrap/bootstrap.sh`). For developer onboarding instructions, see `backlog/491-auto-contexts.md` and `backlog/435-remote-first-development.md`.

## 1. Problem statement

The legacy `.devcontainer/bootstrap.sh` script was tightly coupled to k3d: it created the local registry and cluster, installed Argo CD, and applied the GitOps ApplicationSets in `gitops/ameide-gitops/environments/dev/argocd`. That flow worked well for the inner loop, but it could not be reused when the substrate came from Azure Bicep templates (`infra/bicep/managed-application/main.bicep`) or any other managed Kubernetes footprint. Cluster creation, identity wiring, and DNS / Key Vault / ACR provisioning already live in Bicep, yet operators still need a deterministic "post-cluster bootstrap" that installs Argo CD and the RollingSync ApplicationSets irrespective of where the cluster came from.

## 2. Goals

1. **Environment-agnostic bootstrap.** A single entry point should initialize Argo CD, repo credentials, image pull secrets, and the RollingSync ApplicationSet on any conformant cluster (k3d, AKS staging, AKS production, etc.).
2. **Plays nicely with Bicep outputs.** The bootstrap must accept the AKS credentials, OIDC issuer, Key Vault name, and DNS resource IDs emitted by the Bicep deployment so it can configure GitOps values, secrets, and annotations without manual edits.
3. **Composable workflow.** Keep the existing v1 script for devcontainer convenience, but add a v2 implementation that can run from CI/CD, a developer laptop, or automation (e.g., GitHub Actions) once a kubeconfig is available.
4. **Deterministic Argo setup.** Pin the Argo CD install to immutable content owned in our repos (Helm chart version or a checked-in manifest at a specific tag) and enforce RollingSync ApplicationSets (`foundation-dev`, `data-services-dev`, `platform-dev`, `apps-dev`) so every environment lands on the same structure described in `backlog/375-rolling-sync-wave-redesign.md`.

Non-goals:
- Provisioning Azure infrastructure (AKS, ACR, Key Vault) remains in Bicep.
- Handling per-product configuration drift outside of GitOps (e.g., editing workloads directly) remains out of scope.

## 3. Design

### 3.1 Layered responsibilities

| Layer | Owner | Deliverable |
| --- | --- | --- |
| **Substrate** | Bicep templates (`infra/bicep/managed-application`) or devcontainer k3d | AKS/k3d cluster, ACR, Key Vault, managed identities, DNS zone. |
| **Bootstrap v2** | New script (see §3.2) | Installs Argo CD, seeds secrets, applies GitOps ApplicationSets, waits for RollingSync progress. |
| **GitOps** | `gitops/ameide-gitops/**` | ApplicationSets + components encoding the wave layout from backlog/375. |

### 3.2 Interface & execution model

- Deliver a new CLI (e.g., `./tools/bootstrap-v2`) written in Bash initially for parity with v1. The CLI accepts:
  - `--kubeconfig` (path) or `--context` (name) to target an existing cluster.
  - `--values-root` to point at the environment-specific GitOps values (defaults to `gitops/ameide-gitops` tree).
  - Optional `--bicep-outputs file.json` to ingest AKS metadata (OIDC issuer, Key Vault name, DNS resource IDs).
- AppSet filtering flags were removed; bootstrap always applies the root `ameide`
  Application and lets its RollingSync ApplicationSet drive all components.
- When invoked inside the devcontainer, the CLI can auto-detect the `k3d-ameide` context and behave like today’s bootstrap; in AKS it simply relies on the kubeconfig produced by `az aks get-credentials`.
- `.devcontainer/postCreate.sh` should run the CLI directly with the defaults we rely on for local development (`--reset-k3d`, `--show-admin-password`, `--port-forward`) instead of maintaining a separate wrapper. For Azure rollouts, CI/CD (or an operator) runs the same CLI manually or via GitHub Actions after the Bicep deployment completes.

### 3.3 Workflow steps

1. **Cluster readiness check**  
   - Confirm the target context responds to `kubectl version` and wait for the `kube-system` core pods using `scripts/dev/wait-for-kube-api.sh` logic.

2. **Install / reconcile Argo CD**  
   - Install Argo CD via the pinned Helm chart (`argo-cd@9.1.3`) with base + environment values from GitOps.  
   - **Bootstrap uses public images** to avoid the ghcr-pull chicken-and-egg (see § 3.3.1 below).
   - Wait for `Deployment/argocd-repo-server`, `Deployment/argocd-application-controller`, and `Deployment/argocd-redis` to reach the desired state.
   - **Login flows:** ensure both password-based and OIDC-based access are validated after install (see login flow section).

### 3.3.1 Bootstrap public images (chicken-and-egg resolution)

ArgoCD's Redis uses the GHCR mirror (`ghcr.io/ameideio/mirror/redis`) which requires the `ghcr-pull` secret. However:

1. `ghcr-pull` is created by ExternalSecrets pulling from Azure Key Vault
2. ExternalSecrets operator is deployed BY ArgoCD
3. This creates a circular dependency during initial bootstrap

**Resolution:** Bootstrap installs ArgoCD with public images via environment-specific overrides:
- `sources/values/env/{env}/foundation/foundation-argocd.yaml` sets `global.imagePullSecrets: []` and uses `redis:7.2.5-alpine` from Docker Hub
- Once ArgoCD deploys ExternalSecrets → SecretStore → ExternalSecret, the `ghcr-pull` secret materializes
- ArgoCD self-manages and reconciles to the GHCR mirror images defined in `sources/values/common/argocd.yaml`

**Why not pre-deploy ExternalSecrets before ArgoCD?**
- Would require bootstrap to also deploy: ExternalSecrets CRDs, operator, SecretStore (with Workload Identity), ExternalSecret
- Adds significant bootstrap complexity for marginal benefit
- Public images work reliably and the transition to GHCR is automatic

**Azure Key Vault remains authoritative** for all third-party secrets (GHCR credentials, etc.). The public image workaround is bootstrap-only; steady-state uses GHCR mirror with proper auth.

3. **Seed repo + registry secrets**
   - **ghcr-pull is NOT seeded by bootstrap** - it comes from ExternalSecrets after ArgoCD deploys the operator.
   - AKS environments lean on Azure-managed identity wherever possible: node pools or workload identity service accounts get pull permissions on ACR.
   - Azure Key Vault is the authoritative source for GHCR credentials; Terraform seeds `.env` values to Key Vault during infrastructure provisioning.

4. **Apply GitOps definitions**  
   - Apply the single "root" Application checked into `gitops/ameide-gitops/environments/<env>/argocd/applications/ameide.yaml` that manages Argo’s own namespace, repo definitions, AppProjects, and the RollingSync ApplicationSet.  
   - AppSet filters are removed; the RollingSync ApplicationSet drives all components.

5. **Post-sync verification**  
   - Optionally poll `argocd app list` or `kubectl -n argocd get applicationset ameide-dev` to confirm RollingSync progress; `--wait-tier` remains available as a generic wait gate.  
   - Emit a concise human log and a structured JSON summary (opt-in via `--output json`) so pipelines can parse results.

Bootstrap commands remain re-entrant: every `kubectl` interaction uses `apply` semantics (or `kubectl create --dry-run=client -o yaml | kubectl apply -f -` for secrets), and the CLI surfaces clear exit codes (1 = kube API unreachable, 2 = Argo CD failed to become Ready, 3 = GitOps apply or wait timed out) so CI/CD can react deterministically.

### 3.4 Configuration artifacts

Maintain environment descriptors under `infra/environments/<name>/bootstrap.yaml` containing:

```yaml
# Current implementation: infra/environments/dev/bootstrap.yaml
env: dev
cluster:
  context: k3d-ameide
  type: k3d
gitops:
  root: gitops/ameide-gitops
  env: dev
  argoInstallManifest: infra/argocd/install/argo-cd-v3.2.1.yaml
  rootApplication: gitops/ameide-gitops/environments/dev/argocd/applications/ameide.yaml
bicepOutputsFile: artifacts/bicep-outputs/dev.json
secrets:
  repo:
    manifest: gitops/ameide-gitops/environments/dev/argocd/repos/ameide-gitops.yaml
    usernameEnv: ARGOCD_REPO_USERNAME
    tokenEnv: GITHUB_TOKEN
  dockerhub:
    strategy: env
    secretName: dockerhub-pull
    namespaces:
      - ameide
    registry: https://index.docker.io/v1/
    usernameEnv: DCKR_USERNAME
    tokenEnv: DCKR_PAT
  envFile: .env
```

The CLI can accept `--config` to load overrides per environment while still allowing flags (e.g., overriding the context). CI/CD exports the actual Bicep deployment outputs (via `az deployment ... --query properties.outputs -o json`) into `artifacts/bicep-outputs/<env>.json`, and bootstrap simply consumes that file without manual edits. This keeps Azure-specific values versioned next to their Bicep parameter files and documents how bootstrap v2 consumes them.

#### Target environment configs

| Environment | Config File | Status |
|-------------|-------------|--------|
| `dev` (k3d) | `infra/environments/dev/bootstrap.yaml` | ✅ Implemented |
| `staging` (AKS) | `infra/environments/staging/bootstrap.yaml` | ✅ Implemented (pending cluster migration) |
| `prod` (AKS) | `infra/environments/prod/bootstrap.yaml` | ⏳ Pending |
| `telepresence` | `infra/environments/dev/bootstrap-telepresence.yaml` | ✅ Implemented |

### 3.6 Argo CD login flows (dev/staging/prod)

- **Password-based (fallback / CI sanity):**  
  1. Ensure ExternalSecret `vault-secrets-argocd` is Healthy; verify keys `admin.password`, `admin.passwordMtime`, `server.secretkey`, `dex.oauth.clientSecret`.  
  2. With port-forward running (`0.0.0.0:8443 -> svc/argocd-server:443` in dev), run `argocd login localhost:8443 --username admin --password <decoded admin.password> --grpc-web`.  
  3. Expect Secure cookies (configs.cm.url is HTTPS and server.insecure=false) and RBAC from `argocd-rbac-cm`.
- **OIDC-based (primary):**  
  1. Keycloak realm import Healthy; `argocd` client exists with redirect URIs for dev/staging/prod and Dex client secret populated in the Argo secret.  
  2. Dex issuer points at the Keycloak realm HTTPS endpoint (no hostAlias/HTTP patches); TLS uses the cluster trust bundle. For local dev, add the Keycloak TLS CA to Argo/Dex via `configs.tls.cacert` or ensure the in-cluster service certificate chains to the cluster CA so Dex validates it.  
  3. With port-forward or ingress, use `argocd login localhost:8443 --sso --grpc-web` (or browser) and sign in via Keycloak. Group → role mapping: `/argocd-admin` -> admin, `/argocd-readonly` -> readonly.  
  4. Post-login validation: `argocd account get-user-info` shows claims; navigation works without local admin.
- **Failure signals:** OIDC failures often show Dex 401/400 (issuer/client mismatch) or missing secrets; password login fails if ExternalSecret isn’t ready or admin disabled in values.

### 3.5 Vendor-aligned alternatives

- **Argo CD Autopilot.** Provides a supported bootstrap + repo layout generator; rejected because we already have the RollingSync tier conventions and need to ingest Bicep outputs/Key Vault wiring that Autopilot does not model.
- **AKS GitOps (Flux/Argo) cluster extensions.** Azure can install/manage Argo as an AKS extension tied to Git repos. We may revisit this once the managed extension exposes the same customization surface, but today we need identical flows for AKS and k3d plus tighter control over RollingSync health gates.
- **Argo-manages-Argo pattern.** Adopted partially by having the root ApplicationSet own Argo’s namespace/config so the operator reconciles itself after bootstrap; this keeps us aligned with Argo’s declarative setup guidance.

### 3.7 Current implementation status (observed)

- **RollingSync layout:** Implementation now uses a single RollingSync ApplicationSet `ameide-dev` that walks every component (`gitops/ameide-gitops/environments/dev/argocd/apps/ameide.yaml`). The earlier four-tier AppSet model (foundation/data-services/platform/apps) and the `--appsets` filter are removed; `--wait-tier` remains as a generic wait gate.
- **Bicep outputs:** `--bicep-outputs` is parsed into an in-memory map but is not consumed for OIDC issuer, Key Vault, DNS, or secret sourcing. Consumers still rely on env/pinned values rather than Bicep outputs.
- **Argo self-management:** Bootstrap installs Argo via Helm, applies Argo projects/config/repo creds via `kubectl`, and applies the root `ameide` Application; Argo does not reconcile its own namespace/config beyond the RollingSync ApplicationSet.
- **Pinned install manifest:** Environment config exposes `gitops.argoInstallManifest` (currently `infra/argocd/install/argo-cd-v3.2.1.yaml`), and bootstrap uses the manifest when available.
- **Login/health validation:** The CLI prints the admin password when requested but does not validate OIDC/ExternalSecret health or Dex client wiring post-install.
- **Exit codes / summaries:** Exit codes are generic `1` on failure paths and the JSON summary omits applied AppSets because no names are recorded during the run.
- **GHCR seeding behavior:** GHCR logic has been removed; only Docker Hub env strategy remains.
- **Image availability:** Legacy k3d flow aligns on the k3d registry (`k3d-ameide.localhost:5001/ameide`) and uses `tag: dev` + `pullPolicy: IfNotPresent` in dev values. Current target-state image reference policy for GitOps-managed environments is tracked in `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.

### 3.8 Domain naming

Local development now uses the `local.ameide.io` domain so hostnames match the unified environment matrix (see [434-unified-environment-naming.md](434-unified-environment-naming.md#epic-env-1)). DNS helpers, cert-manager SANs, the Gateway listeners, and the Tilt-only Helm values were updated accordingly:
- Host dnsmasq scripts are no longer required; Telepresence routes the `dev.ameide.io`/`local.ameide.io` traffic directly to the AKS cluster.
- `gitops/ameide-gitops/sources/values/env/local/apps/platform/gateway.yaml` exposes the `https-local` listener plus `auth.local.ameide.io`.
- `gitops/ameide-gitops/sources/values/env/dev/platform/platform-cert-manager-config.yaml` includes the local wildcard SAN.
- `gitops/ameide-gitops/sources/values/env/dev/apps/apps-www-ameide-tilt.yaml` & `apps-www-ameide-platform-tilt.yaml` reference `platform.local.ameide.io`.
- `gitops/ameide-gitops/sources/values/_shared/platform/platform-keycloak-realm.yaml` whitelists the local callback URLs.

## 4. Implementation plan

### Phase 1: Local development (✅ Complete)
1. ✅ **Scaffold CLI + config loader** – `tools/bootstrap/bootstrap-v2.sh` with argument parsing, config file support, and shared logging utilities.
2. ✅ **Retire the v1 wrapper** – `.devcontainer/postCreate.sh` calls the CLI directly with baked-in defaults.
3. ✅ **Library modules** – Modular implementation in `tools/bootstrap/lib/*.sh` (see [429](429-devcontainer-bootstrap.md#4-library-modules)).

### Phase 2: Remote cluster support (⏳ In Progress)
4. ⏳ **Remote GitOps migration** – Migrate AKS clusters from legacy `ameide.git/infra/kubernetes/gitops/` to `ameide-gitops` repo (see [434](434-unified-environment-naming.md#migration-needed)).
5. ⏳ **Create staging/prod bootstrap configs** – Add `infra/environments/staging/bootstrap.yaml` and `infra/environments/prod/bootstrap.yaml`.
6. ⏳ **Wire Bicep outputs consumption** – Implement actual consumption of `--bicep-outputs` for OIDC issuer, Key Vault, DNS.
7. ⏳ **Add Azure runbook** – Document how to deploy Bicep (`infra/bicep/managed-application/README.md`), capture outputs, and call the new CLI with `az aks get-credentials`.

### Phase 3: Telepresence integration (⏳ Pending)
8. ✅ **Create telepresence bootstrap config** – `infra/environments/dev/bootstrap-telepresence.yaml` defines the remote context (`ameide-stg-aks`), marks `gitops.skip: true`, and enables auto-connect (see [434](434-unified-environment-naming.md#epic-env-4)).
9. ✅ **Wire online-telepresence mode** – `.devcontainer/postCreate.sh` now reads the telepresence config, persists mode via `~/.devcontainer-mode.env`, switches the kube context, sets `TILT_REMOTE=1`, and calls `tools/dev/telepresence.sh connect` when `autoConnect` is enabled (see [432](432-devcontainer-modes-offline-online.md#epic-dc-4)). The workflow is documented in `docs/dev-workflows/telepresence.md`.

### Phase 4: Domain and naming standardization (⏳ Pending)
10. ✅ **Domain rename** – `tilt.ameide.io` → `local.ameide.io` across DNS helpers, Gateway listeners, cert-manager values, and Tilt-only Helm values (see [434](434-unified-environment-naming.md#epic-env-1)).
11. ⏳ **Namespace conventions** – Implement per-environment namespaces for AKS (`ameide-dev`, `ameide-staging`, `ameide-prod`).

### Phase 5: CI/CD and validation
12. ⏳ **CI/CD validation** – Add a GitHub Actions workflow or reusable script that runs bootstrap v2 against k3d inside CI to catch regressions.
13. ⏳ **Progressive rollout** – Test against staging AKS cluster provisioned via Bicep, then production.

## 5. How to test in the devcontainer (happy path)

Prereqs: set GITHUB_TOKEN, GHCR_USERNAME, DCKR_USERNAME, DCKR_PAT in .env. Ensure Keycloak realm import and Vault secrets are available (argocd Dex client, realm client secrets).
Run bootstrap: tools/bootstrap/bootstrap-v2.sh --config infra/environments/dev/bootstrap.yaml --reset-k3d --show-admin-password --port-forward. Expect Helm install of Argo 9.1.3, repo creds applied, ApplicationSets seeded, port-forward on 8443.
Verify Argo deploys: kubectl -n argocd get deploy,statefulset shows all Ready; kubectl -n argocd get cm argocd-cm -o json | jq '.data.url' is https://localhost:8443.
Verify secrets/ExternalSecrets:
kubectl -n argocd get externalsecret vault-secrets-argocd -o json | jq '.status.conditions'
kubectl -n argocd get secret argocd-secret -o yaml | grep dex.oauth.clientSecret
kubectl -n ameide get externalsecret keycloak-realm-oidc-clients (for Keycloak clients).
Test password login: argocd login localhost:8443 --username admin --password <decoded admin.password> --grpc-web.
Test OIDC login: ensure Keycloak realm + argocd client exist and Keycloak TLS is trusted (either service cert from cluster CA or load CA via configs.tls.cacert), then argocd login localhost:8443 --sso --grpc-web and confirm argocd account get-user-info.
Check ApplicationSets: kubectl -n argocd get applicationset -l tier and argocd app list -l tier.

### Operational decisions
- **Structured logs:** CLI keeps human-readable output by default and adds `--output json` for CI/CD summaries.
- **RollingSync waits:** Optional `--wait-tier` and `--timeout` flags allow pipelines to block on specific tiers without forcing waits on every run.
- **Bicep outputs:** Pipelines export `properties.outputs` JSON artifacts and pass them to `--bicep-outputs`, removing manual copy/paste
