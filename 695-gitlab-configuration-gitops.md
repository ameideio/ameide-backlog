---
title: "695 — GitLab (CE) GitOps Configuration Tracking"
status: in-progress
owners:
  - platform
created: 2026-01-19
related:
  - 694-elements-gitlab-v6.md
  - 496-eda-principles-v6.md
---

# 695 — GitLab (CE) GitOps Configuration Tracking

Track the GitOps-managed **GitLab Community Edition** deployment in this repo: versioning, ArgoCD wiring, routing/DNS/TLS, secrets, persistence, and production-readiness constraints.

This backlog item is a living “single place to look” so GitLab changes remain intentional, reviewable, and aligned with upstream guidance.

## Access model (aligns with 694)

GitLab is treated as a **platform-owned subsystem**:

- Primary users are **platform operators** and **platform services/bots** (not end-tenant users).
- Tenant governance is enforced by the platform; tenants do not get arbitrary GitLab accounts by default.
- Hostnames may look “public” for consistency (`gitlab.<env>.ameide.io`), but access should be restricted (private DNS/internal Gateway and/or network allowlists + SSO).

## Scope

- GitLab **workload deployment configuration** (ArgoCD + Helm values), not GitLab project/group governance.
- Environment overlays: `dev`, `staging`, `production`, `local`.
- Runtime dependencies that affect correctness (DNS/TLS, storage classes, external secrets, identity/OIDC).

## Non-goals

- No manual mutation of cloud cluster resources (GitOps-only path per `AGENTS.md`).
- No “make it pass” changes that weaken safety/validation contracts.

## Current state (repo truth)

- ArgoCD components:
  - Workload: `environments/_shared/components/platform/developer/gitlab/component.yaml`
  - Smokes: `environments/_shared/components/platform/developer/gitlab-smoke/component.yaml`
  - Local overlay components:
    - Workload: `environments/local/components/platform/developer/gitlab/component.yaml`
    - Smokes: `environments/local/components/platform/developer/gitlab-smoke/component.yaml`
- Wrapper chart (routes + secrets wiring): `sources/charts/platform/gitlab`
- Vendored upstream chart: `sources/charts/third_party/gitlab/gitlab/9.6.1` (appVersion `18.6.1`)
- CE/OSS enforcement: set `gitlab.global.edition=ce` (GitLab chart defaults to EE unless explicitly set to CE).
- Ingress disabled; exposure via Gateway API `HTTPRoute`: `sources/charts/platform/gitlab/templates/httproute.yaml`
- OIDC provider is sourced from Vault via ExternalSecrets: `sources/charts/platform/gitlab/templates/externalsecret.yaml`
- `gitlab-runner` is disabled in values (no in-cluster runner install by default).

## Progress (2026-01-19)

- GitLab is now part of the standard GitOps component set (no longer `_optional`).
- Added baseline PostSync smoke job checks for GitLab (`platform-gitlab-smoke`) to keep rollout evidence consistent with other platform smokes.
- Fixed hostname assembly footgun by removing the `global.hosts.gitlab.name: gitlab` override so the chart assembles `gitlab.<domain>`.
- Disabled GitLab chart `upgradeCheck` under ArgoCD to avoid Helm hook failures blocking first installs (GitOps rollouts).
- Switched GitLab object storage to the shared in-namespace MinIO (`data-minio`) and disabled GitLab’s bundled MinIO chart (temporary posture; see “Object storage”).
- Updated 694 ↔ 695 alignment notes and cross-references.

## Next (tracked work)

- Keep GitLab Container Registry disabled until object storage credentials are least-privilege and routing/TLS are explicitly defined.
- Rehearse and validate SSH exposure via Gateway TCPRoute in `dev`/`staging` (port `22`, hostname `gitlab.<env>.ameide.io`) and ensure network policies allow the listener end-to-end.
- Document the admin bootstrap runbook execution posture (where the break-glass token lives in Vault, rotation expectations, and audit evidence).
- Promote the shared-secrets vs Vault matrix from “decision” to “explicit secret names + owners” and add drift checks.
- Replace shared MinIO root credentials with a dedicated GitLab object-store user + scoped policies (and/or move to external object storage per the hybrid posture).

## Version posture

- Current pin: GitLab chart `9.6.1` (GitLab appVersion `18.6.1`), as recorded in `sources/charts/third_party/charts.lock.yaml`.
- Rationale: vendored charts are the determinism/default posture; upgrades should be explicit, rehearsed, and tracked (patch → minor) rather than “float to latest”.
- Next steps: track “why not latest patch/minor” and a planned upgrade path (e.g., `9.6.x` patch bump first, then evaluate `9.7/9.8`).

## Hostname convention

Standardize GitLab external hostnames as:

- `dev`: `gitlab.dev.ameide.io`
- `staging`: `gitlab.staging.ameide.io`
- `production`: `gitlab.ameide.io` (current) or `gitlab.prod.ameide.io` (strict `gitlab.<env>.ameide.io` pattern)
- `local`: `gitlab.local.ameide.io`

Notes:

- Prefer `gitlab.global.hosts.domain=<env>.ameide.io` and let the chart assemble `gitlab.<domain>`.
- If we adopt `gitlab.prod.ameide.io`, also update OIDC redirect URIs, clone URLs (`global.hosts.ssh`), DNS records, and any runtime-facts wiring that assumes `gitlab.ameide.io`.

## Endpoints matrix (per environment)

Minimum endpoints to keep consistent (DNS/TLS, clone URLs, and OIDC redirect URIs):

- Web: `https://gitlab.<env>.ameide.io/` (production currently `https://gitlab.ameide.io/`)
- SSH: `ssh://git@gitlab.<env>.ameide.io:<shell-port>/...` (requires L4 exposure; `global.hosts.ssh` + `global.shell.port`)
- Registry (if/when enabled): `https://registry.<env>.ameide.io/` (or another explicit hostname; must not be blocked by values collisions)

## SSH exposure (decision: Gateway TCPRoute on :22)

**Decision:** expose GitLab SSH via Gateway API `TCPRoute` on port `22`, using the same hostname as web (`gitlab.<env>.ameide.io`).

Contract:

- DNS: `gitlab.<env>.ameide.io` → the environment Gateway VIP
- Gateway: listener `ssh` on port `22` (protocol `TCP`)
- Route: `TCPRoute` attaches to the `ssh` listener and forwards to the GitLab Shell Service (`platform-gitlab-gitlab-shell:22`)
- GitLab values:
  - `global.hosts.ssh: gitlab.<env>.ameide.io`
  - `global.shell.port: 22`

Implementation note: requires the experimental Gateway API `TCPRoute` CRD (`tcproutes.gateway.networking.k8s.io`) installed cluster-wide.

## Bootstrap alignment (seeded `admin@ameide.io`)

The platform already has a deterministic “seeded persona” contract (Keycloak + platform DB) centered on `admin@ameide.io` (see `backlog/582-local-dev-seeding.md`). GitLab bootstrap should align with that same persona so operators don’t need a separate, manual “first user” path.

**Bootstrap contract**

- `admin@ameide.io` can authenticate to GitLab via Keycloak OIDC (no local-only GitLab user as the primary path).
- `admin@ameide.io` is an administrator in GitLab (or equivalent “platform admin” posture) without manual UI steps.
- Any break-glass local credentials (e.g., initial root password secret) are treated as emergency-only, not the default operational path.

**Inputs we already have**

- Seeded user `admin@ameide.io` exists in Keycloak and is used by SSO verifiers (password sourced from `Secret/playwright-int-tests-secrets` key `E2E_SSO_PASSWORD`; see `infra/scripts/verify-argocd-sso.sh`).
- Keycloak tokens include a `groups` claim (full path), so group-based mapping is available when needed.

**Decision (OSS/CE-safe, vendor-aligned)**

GitLab “SSO” is OmniAuth. OpenID Connect sign-in works on CE, but group-based access/admin mapping is not a safe contract to rely on for CE. Therefore:

1. **IdP enforcement (Keycloak):** restrict who can authenticate to the `gitlab` client (operators/services only).
2. **GitLab safety gate:** keep `blockAutoCreatedUsers=true` so JIT-created users are blocked (pending approval) until explicitly approved.
3. **Admin bootstrap:** use a **runbook** (API-driven) to approve + optionally promote `admin@ameide.io` after first OIDC login; do not rely on IdP group→admin mapping.
4. **Verification:** keep ArgoCD PostSync smokes for readiness + OIDC wiring; keep “is admin” as an operator-controlled evidence step (audit + API), not an automated always-on in-cluster Job.

**Runbook helpers (repo scripts)**

- `scripts/gitlab-approve-user.sh` (approve a pending OmniAuth user)
- `scripts/gitlab-promote-user-admin.sh` (promote a user to instance admin)

## ArgoCD smokes alignment

This repo’s standard verification mechanism is **ArgoCD PostSync smoke jobs**, implemented via:

- `sources/charts/foundation/helm-test-jobs` (generic hook Jobs; used by many `*-smoke` components)
- `sources/charts/foundation/platform-smoke` (platform-wide checks)
- `argocd/applicationsets/ameide.yaml` (rolling sync phases; smokes run as part of the normal convergence loop)

GitLab should follow the same pattern so we don’t rely on ad-hoc/manual validation.

**Required GitLab smoke coverage (when GitLab is enabled in an environment)**

1. **ExternalSecrets ready:** `ExternalSecret/gitlab-oidc-provider` is Ready and target Secret exists.
2. **Workload ready:** core GitLab webservice is rolled out (and responding on the in-cluster service endpoint).
3. **OIDC wiring sanity:** GitLab responds with an OIDC redirect (or presents the OIDC provider on the sign-in page) consistent with the Keycloak issuer and the environment hostnames.
4. **No-placeholder secrets:** prevent “placeholder_*” client secrets/redirect URIs from surviving in managed environments (same drift guard philosophy used elsewhere).

**Policy**

- If GitLab remains an optional workload, keep its smoke component optional as well (enabled/disabled together), to avoid failing the global smoke phases in environments where GitLab is not installed.
- Prefer “cheap” checks (HTTP + secret readiness) as PostSync hooks; keep full browser login assertions in separate CI verifiers if needed.

## Secrets posture (shared-secrets vs Vault)

Upstream GitLab uses a `shared-secrets` job to generate many “default” secrets unless you supply them. This is a primary drift lever and should be made explicit.

Decide and document (matrix) which secrets are:

- **Vault/ExternalSecrets supplied** (authoritative, stable per environment)
  - GitLab OIDC client secret (Keycloak-generated → client-patcher → Vault): `gitlab-oidc-client-secret`
  - GitLab OmniAuth provider Secret rendered from Vault: `Secret/gitlab-oidc-provider` (key `provider`)
- **Chart-generated** (acceptable to generate, but must be understood and monitored)
  - Initial root password secret (break-glass only)
  - Internal TLS, SSH host keys, and other shared secrets (if we keep ingress disabled, TLS is still relevant for internal components)

**Concrete names/patterns to track**

- Release name (Helm): `platform-gitlab`
- ExternalSecret (ours): `ExternalSecret/gitlab-oidc-provider` → `Secret/gitlab-oidc-provider`
- Shared-secrets job (upstream): `Job/platform-gitlab-shared-secrets` (pre-install/pre-upgrade)
- Initial root password (upstream, break-glass): `Secret/platform-gitlab-gitlab-initial-root-password` (key `password`)

## Known constraints / footguns

- **Tools/version prerequisites (upstream):** keep `kubectl` within one minor of the cluster; use Helm v3.17.3+ and avoid Helm v3.18.0 per GitLab’s prerequisites docs.
- **Hostname assembly:** prefer `gitlab.global.hosts.domain` per environment; only set `gitlab.global.hosts.gitlab.name` when providing a full hostname override (FQDN).
- **Gateway API routing:** GitLab web UI is routed via `HTTPRoute`; Git SSH requires L4 (`TCPRoute` or equivalent) and a Gateway implementation that supports it (tie back to `global.hosts.ssh` and `global.shell.port`).
- **Gateway API maturity:** treat Gateway routing changes as potentially breaking; rehearse in `dev`/`staging` before production.
- **Ingress-nginx retirement:** do not build a future plan that depends on ingress-nginx after March 2026; if ingress is used, document the supported controller posture explicitly.
- **Globals collision (`registry`):** the repo-wide image registry key is `imageRegistry` (not `registry`) to avoid collisions with vendor charts that define a top-level `registry:` object (e.g., GitLab Container Registry settings).
- **Production warning (upstream):** default “all-in-cluster” installs are PoC; production requires a cloud-native hybrid posture (external PostgreSQL/Redis/object storage/Gitaly) and careful sizing.

## Contract checkpoints (what “good” looks like)

1. **Version mapping is explicit:** GitLab Helm chart version selection is intentional and recorded here (chart version != GitLab version).
2. **Routing is consistent:** GitLab external URL, OIDC redirect URIs, and UI clone URLs match the active DNS/TLS posture.
3. **Secrets are GitOps-safe:** no embedded secrets in values; secrets come from Vault/ExternalSecrets with clear key conventions per env.
4. **Upgrade posture is safe:** upgrades are rehearsed in `dev`/`staging` with rollback notes (data migration awareness).
5. **Prod posture is explicit:** either “PoC only” is accepted and documented, or hybrid dependencies are planned and tracked.

## Object storage (temporary: shared in-cluster MinIO)

GitLab requires S3-compatible object storage for core features (artifacts, LFS, uploads, packages). Upstream’s production guidance is to use external object storage; in the current Ameide posture we use the **shared in-namespace MinIO** (`data-minio`) as a **temporary** stop-gap.

- **MinIO deployment:** Bitnami MinIO `data-minio` (per-environment namespace), configured via `sources/values/_shared/data/data-minio.yaml` and env overrides.
- **GitLab bundled MinIO:** disabled via `gitlab.global.minio.enabled=false` to avoid arm64 image issues and bucket-job drift.
- **Connection Secret:** `Secret/gitlab-object-storage` is materialized in the GitLab namespace via `ExternalSecret` (wrapper chart template `sources/charts/platform/gitlab/templates/externalsecret-object-storage.yaml`).
  - **Temporary credential source:** Vault keys `minio-root-user` + `minio-root-password` (root credentials; to be replaced).
- **Buckets:** created by MinIO default bucket bootstrap (`defaultBuckets`) and include:
  - `git-lfs`, `gitlab-artifacts`, `gitlab-uploads`, `gitlab-packages`
  - `gitlab-mr-diffs`, `gitlab-terraform-state`, `gitlab-ci-secure-files`, `gitlab-dependency-proxy`
  - `gitlab-backups`, `tmp`

Exit criteria for removing this “temporary” posture:

- Create a dedicated MinIO user for GitLab with scoped bucket policies (no root credentials in app namespaces).
- Decide whether the long-term target is external object storage (preferred for production) and track the migration plan (buckets, creds, backups).

## Decision points to resolve

- **Chart sourcing pattern**
  - Keep vendored upstream chart (`sources/charts/third_party/...`) for determinism; or
  - Switch to ArgoCD multi-source pulling from `https://charts.gitlab.io/` and referencing values from this repo (note: ArgoCD multi-source has sharp edges and has been documented as beta in some ArgoCD releases; if adopted, record the ArgoCD version(s) we validate against).
- **Exposure model**
  - Keep Gateway API (`HTTPRoute`/`TCPRoute`) as the standard; or
  - Enable upstream ingress (`global.ingress.*`) and manage cert-manager integration via the chart.
- **Gateway resources**
  - Keep the wrapper chart’s custom `HTTPRoute`/`ExternalSecret` resources; or
  - Adopt upstream chart Gateway API resources (where applicable) and document what we must replicate (TLS termination, TCPRoute for SSH, etc.).
- **SSH exposure**
  - **Decision:** Gateway API `TCPRoute` on `:22` using `gitlab.<env>.ameide.io` (see “SSH exposure”).
- **Runner strategy**
  - Keep `gitlab-runner.install=false` and run workloads elsewhere; or
  - Enable runner with explicit isolation, node pools, and secrets.
- **Resolve values collision (`registry`)**
  - **Decision:** repo-wide image registry key is `imageRegistry` (not `registry`); keep GitLab Container Registry disabled until object storage credentials are least-privilege and routing/TLS are explicitly defined.

## Upgrade / rollback posture

Upstream upgrade guidance is strict; document and follow:

- Backup before upgrades; restore typically requires the same version.
- Prefer vendor-supported backup/restore mechanisms (Toolbox `backup-utility`) and explicitly track which Kubernetes Secrets must be backed up alongside data.
  - Restore requires the same GitLab version (and same CE/EE type).
  - Back up/restore secrets alongside data (restores can break subtly otherwise).

## Work items

1. Record the chosen **chart version mapping** and upgrade cadence (include a “why this version” note).
2. Document the **routing/TLS** posture per environment (Gateway vs ingress, cert source, DNS records ownership).
3. Add an explicit section for **SSH exposure** and a tracked implementation plan (or a clear “not supported” stance).
4. Document **root/admin bootstrap** posture (where the initial credentials live, rotation expectations, and Vault integration).
5. Decide the **shared-secrets vs Vault** matrix (what is generated vs supplied) and add drift checks accordingly.
6. Resolve the **top-level `registry` collision** so GitLab Container Registry can be enabled/configured intentionally.
7. Define “production-ready” posture for Ameide’s use (PoC acceptance vs hybrid dependency plan) and link to the corresponding infra backlogs.

## Upstream references (docs)

- GitLab chart version mappings: https://docs.gitlab.com/charts/installation/version_mappings/
- GitLab chart tools/prerequisites: https://docs.gitlab.com/charts/installation/tools/
- GitLab globals (hosts/ingress/tls): https://docs.gitlab.com/charts/charts/globals/
- GitLab Helm install guidance + production notes: https://docs.gitlab.com/charts/installation/
- GitLab design decisions (incl. Gateway API direction): https://docs.gitlab.com/charts/architecture/decisions/
- GitLab shared-secrets job: https://docs.gitlab.com/charts/charts/shared-secrets/
- GitLab upgrade guidance: https://docs.gitlab.com/charts/installation/upgrade/
- GitLab backup/restore: https://docs.gitlab.com/charts/backup-restore/backup/
- GitLab external ingress notes (incl. ingress-nginx timelines): https://docs.gitlab.com/charts/advanced/external-ingress/
- ArgoCD multi-source values (`$values` + `ref`): https://argo-cd.readthedocs.io/en/latest/user-guide/multiple_sources/
