---
title: 640 – Camunda 8 (Self-Managed) GitOps Configuration Standard
status: draft
owners:
  - platform
created: 2026-01-12
updated: 2026-01-12
---

## Summary

Document the **standard way** to deploy and operate **Camunda 8 (Self-Managed)** in `ameide-gitops` with **no commercial production licenses**, aligned with existing repo patterns:

- Argo CD `ApplicationSet` components (`environments/_shared/components/**`)
- Vendored third-party charts (`sources/charts/third_party/**` via `charts.lock.yaml`)
- **Wrapper chart** as the policy translation layer (routing, secrets, enablement, image policy)
- Operator-first secrets: **Vault KV → External Secrets Operator → Kubernetes Secret**
- Gateway API (HTTPRoute/GRPCRoute) instead of Ingress
- CNPG-backed Postgres (shared cluster)
- Environment overlays (`local`, `dev`, `staging`, `production`) deploy the same logical Camunda stack.

This backlog is a **configuration and integration** guide (not an implementation PR by itself).

## What We Learn From Camunda’s Helm + Argo Guidance

This is what we carry forward from upstream docs and community examples:

1. **Pin by chart version, not “latest app”**
   - Camunda app versions and Helm chart versions are decoupled; upgrades are deliberate.
2. **Separate chart source from environment values**
   - Treat values as reviewable, environment-scoped config (and keep secrets out of git).
3. **Combined “web ingress” vs separate “Zeebe gRPC ingress”**
   - Web UIs and Zeebe gRPC have different protocols and exposure requirements; model them separately.
4. **Redirect URLs must match the public URLs**
   - Whatever public URL scheme we use, Identity/Console/Modeler/Optimize redirect URIs must be correct or login breaks.
5. **Vendor-supported infrastructure matters**
   - Camunda recommends vendor-supported infrastructure for production; in `ameide-gitops` we reuse existing cluster services (Keycloak, CNPG Postgres) and keep all other dependencies explicit and GitOps-owned.

## What Is *Not* Applicable In `ameide-gitops` (And Our Translation)

Many “standard AKS + Argo + Camunda” writeups assume different primitives than this repo.

### 1) “Install an Ingress controller and configure `global.ingress.*`”

Not applicable as written:

- This repo standardizes on **Gateway API** (Envoy Gateway) for north-south HTTP, not Kubernetes `Ingress`.
- Therefore, Camunda chart’s Ingress configuration (`global.ingress.*`, `identityKeycloak.ingress.*`, etc.) is **disabled**.

Translation:

- Create `HTTPRoute`s for web endpoints and a `GRPCRoute` for Zeebe gRPC in the wrapper chart, consistent with platform routing.
- TLS is handled via the repo’s cert-manager + Gateway API approach (not Ingress annotations).

### 2) “Use an Argo CD `Application` manifest per app”

Not applicable:

- This repo generates apps via **ApplicationSets** and `component.yaml` definitions, not one-off `Application` manifests under `apps/`.

Translation:

- Add a component under `environments/_shared/components/**` and values under the standard six-layer values model (base/cluster/env/node-profile/shared/env-specific).

### 3) “Use Argo multi-source to reference the Helm repo directly”

Not applicable by default:

- This repo prefers **vendoring** charts into `sources/charts/third_party/**` for reproducibility and auditability.

Translation:

- Pin the chart in `sources/charts/third_party/charts.lock.yaml` and run the vendoring scripts.
- Consume the vendored chart path from Argo CD.

### 4) “Combined Ingress host + paths is the only access pattern”

Upstream Camunda 8.8+ (“Orchestration Cluster”) serves **Operate and Tasklist from the same service** and derives their redirect URLs from a single root (`orchestration.security.authentication.oidc.redirectUrl`).

Translation (fixed):

- Use **single-host + path routing** for the Orchestration Cluster:
  - `https://camunda.{env}.ameide.io/operate`
  - `https://camunda.{env}.ameide.io/tasklist`
- Route Connectors on the same host under `/connectors`:
  - `https://camunda.{env}.ameide.io/connectors`

This keeps OIDC redirect handling correct and matches the upstream chart’s “combined web ingress” model (Ingress disabled; Gateway API provides the routes).

## Context (Repo)

- Repo conventions:
  - `sources/charts/README.md` (layout + vendoring workflow)
  - `backlog/519-gitops-fleet-policy-hardening.md` (wrappers for third-party; avoid patching vendored trees)
  - `backlog/465-applicationset-architecture.md` (how components are wired)
  - `backlog/464-chart-folder-alignment.md` (wrappers live in the correct domain folder)
- Existing wrapper examples:
  - `sources/charts/platform/coder` (wrapper + dependency)
  - `sources/charts/platform/gitlab` (wrapper + dependency)

## Goals

1. Provide a **single documented deployment shape** for Camunda 8 in this repo (chart selection, folders, component wiring).
2. Avoid drift-prone patterns:
   - no direct editing under `sources/charts/third_party/**` for Ameide-specific requirements
   - no stable secrets generated by Helm; use Vault/ESO
   - no Ingress; use Gateway API routes
3. Clearly define environment-specific choices:
   - dev (low-cost) vs production (durable / scalable / secure)
4. Provide an **upgrade playbook** (chart version bump + vendor + validation).

## Non-goals

- Deploying any commercial-only Camunda components or providing any commercial license material.
- Designing a full multi-tenant Camunda governance model (namespaces/projects/users).
- Replacing existing platform auth (Keycloak).
- Building custom Camunda images.

## Decisions (Fixed)

### 1) Chart source

- Use the official Helm chart: `camunda/camunda-platform` (vendored under `sources/charts/third_party`).

### 2) Wrapper chart (required)

Camunda 8 is deployed via a **wrapper chart** that depends on the vendored upstream chart. The wrapper owns:

- Gateway API exposure (HTTPRoutes/GRPCRoutes) consistent with Ameide routing
- ExternalSecrets / Vault wiring
- Standard enablement toggle (`enabled: true|false`) and policy translation
- Image policy enforcement (digest pinning/mirror preference)

### 3) Auth strategy

Camunda 8 uses the existing **Ameide Keycloak** (no bundled Keycloak):

- `identityKeycloak.enabled=false` (do not deploy Camunda’s Keycloak dependency)
- Configure OIDC across Camunda components (`global.security.authentication.method=oidc` plus component OIDC settings)
- Create OIDC clients and required roles/groups in the existing Keycloak realm via the repo’s Keycloak realm GitOps flow
- Seed dev/local with a deterministic set of users/groups/roles needed to log in and see the UIs (see “Dev/local seeding”)

### 4) Networking + DNS (fixed)

Camunda 8 web UIs are exposed on a **single host** via Gateway API path routing:

- `camunda.{env}.ameide.io` (HTTP)
  - `/operate`
  - `/tasklist`
  - `/connectors`
- `camunda-zeebe.{env}.ameide.io` (gRPC)

All routes are implemented using Gateway API resources (no Kubernetes Ingress).

### 5) Component set (fixed)

This GitOps standard deploys the following Camunda components:

- Enabled in all environments:
  - `orchestration.enabled=true` (Zeebe + Operate + Tasklist)
  - `connectors.enabled=true`
- Disabled in all environments:
  - `console.enabled=false`
  - `optimize.enabled=false`
  - `identity.enabled=false`
  - `webModeler.enabled=false`
- Future work:
  - Web Modeler + Management Identity enablement is tracked in `backlog/642-camunda-web-modeler-identity-gitops.md`.
- License:
  - No `global.license.*` secret is configured and no license material is shipped via GitOps.

### 6) Databases (fixed)

- No external Postgres is required for the enabled component set (Orchestration Cluster + Connectors).
- Any bundled Postgres dependency charts remain disabled (`identityPostgresql.enabled=false`, `webModelerPostgresql.enabled=false`).

### 7) Search backend (fixed)

- Use the chart’s bundled Elasticsearch (`global.elasticsearch.enabled=true`).
- Disable OpenSearch (`global.opensearch.enabled=false`).

## Vendor Hard Edges (Non-Negotiable)

These are the constraints that *must* be treated as “install-time” decisions in GitOps. If we violate them, Argo will not converge by retrying; the only correct path is a deliberate replace/migration.

### 1) Zeebe partitioning is not dynamically rescalable

The upstream documentation states that the Orchestration cluster does not support dynamic scaling for at least some topology parameters (notably `partitionCount`). Treat the Orchestration cluster topology as immutable for an installed environment:

- `orchestration.partitionCount`: **install-time only**
- `orchestration.replicationFactor`: treat as install-time in practice
- `orchestration.clusterSize`: can be reduced/increased, but only within the constraints of the installed topology and available capacity

**Rule:** if an environment is installed with the wrong partitioning, do not “try to fix it via Helm upgrades”. Plan an environment replace/migration.

### 2) StatefulSet storage shape/persistence changes are not upgrade-safe

Kubernetes forbids updates to many StatefulSet spec fields (including volume claim templates). In practice:

- Do **not** attempt to “flip persistence mode / storage shape” for Zeebe or Elasticsearch via in-place upgrades.
- If persistence must change, treat it as a **replace/migration** task (new StatefulSet identity + controlled cutover).

### 3) Values schema types matter (strings vs numbers)

The upstream chart schema defines several topology values as **strings** (`"1"`, `"3"`, …). If env overlays provide numbers, Helm schema validation rejects the values and Argo cannot apply the intended configuration.

**Rule:** keep `clusterSize`, `partitionCount`, and `replicationFactor` as **quoted strings** in values.

## Guardrails (Prevent Recurrence)

### A) Render-time URL sanity check (CI)

Add a CI gate that fails if rendered manifests contain “empty-host” OIDC URLs like:

- `https://auth./realms/ameide`
- `https://camunda./...`

This catches `tpl` scoping regressions early (before they become runtime `500`s / crashloops).

### B) Secret authority / placeholder prevention

Camunda requires valid OIDC client secrets. In this repo, the contract is:

1. Keycloak generates client secrets
2. `platform-keycloak-realm-client-patcher` extracts them and writes to Vault
3. ExternalSecrets sync them into Kubernetes Secrets (e.g. `camunda-oidc-client-secrets`)

If `camunda-oidc-client-secrets` contains placeholder values, treat it as a **platform secret pipeline failure**, not a Camunda app bug.

### C) Service-to-service tokens must include the Zeebe audience

Connectors (and any other non-browser clients) authenticate to Zeebe Gateway using a **service account token** from Keycloak.
The Camunda chart defaults expect the token to include the audience `orchestration-api`.

**If the token does not include `aud=orchestration-api`, Zeebe rejects it** (`UNAUTHENTICATED`, HTTP 401), and Connectors never becomes Ready.

Repo standard:

- Add a Keycloak client scope that injects the audience via `oidc-audience-mapper` (e.g. `orchestration-api-audience`).
- Attach that scope to the relevant clients (`orchestration`, `connectors`) via the `platform-keycloak-realm` client reconciliation values.

## Target GitOps Shape (Files To Add)

### A) Vendoring entry (upstream chart pin)

- Add to: `sources/charts/third_party/charts.lock.yaml`
  - `alias: camunda`
  - `name: camunda-platform`
  - `version: <pinned>`
  - `repo: https://helm.camunda.io`
- Run:
  - `./scripts/vendor-charts.sh`
  - `./scripts/check-vendored-charts.sh`

### B) Component definition (ArgoCD ApplicationSet)

Create:

- `environments/_shared/components/platform/control-plane/camunda8/component.yaml`

Guidelines:
- Choose a rollout phase that respects dependencies:
  - DB + secrets must exist before Camunda workloads start.
  - Gateway must exist before routes are useful (but is not strictly required for pods to run).

### C) Wrapper chart (required)

Create:

- `sources/charts/platform/camunda8/Chart.yaml` (dependency on vendored camunda-platform)
- `sources/charts/platform/camunda8/values.yaml` (disabled by default)
- `sources/charts/platform/camunda8/templates/httproute-*.yaml` for UIs (single host `camunda.{env}.ameide.io` with path routing)
- `sources/charts/platform/camunda8/templates/externalsecret-*.yaml` for secrets sourced from Vault

### D) Values layering

Create:

- `sources/values/_shared/platform/platform-camunda8.yaml` (no env-specific hosts)
- `sources/values/env/dev/platform/platform-camunda8.yaml` (enable + hostnames + sizing)
- `sources/values/env/local/platform/platform-camunda8.yaml` (enable + hostnames + sizing for local overlay)
- `sources/values/env/staging/platform/platform-camunda8.yaml`
- `sources/values/env/production/platform/platform-camunda8.yaml`

## Environment overlays requirement (fixed)

All environment overlays are first-class and deploy the same logical Camunda stack:

- `local` deploys the same stack into the local cluster.
- `dev`, `staging`, and `production` deploy the same stack into their respective namespaces.
- Per-environment overlays only change:
  - hostnames (via `{env}` in `{{ .Values.domain }}`)
  - sizing and scheduling (resources, replicas, node selectors/tolerations)
  - environment-specific infra wiring where required by the platform (e.g., storageClass, DNS zone inputs)

## Done definition (deployment + UX)

“Done” means Camunda 8 is fully deployed and usable in **local**, **dev**, **staging**, and **production**:

- All enabled services are deployed, Healthy, and reachable via:
  - `camunda.{env}.ameide.io/operate` (HTTP)
  - `camunda.{env}.ameide.io/tasklist` (HTTP)
  - `camunda.{env}.ameide.io/connectors` (HTTP)
  - `camunda-zeebe.{env}.ameide.io` (gRPC)
- OIDC is integrated with the existing Ameide Keycloak and **seeded** in dev/local with the required realm config (clients/redirect URIs/roles/users).
- Smoke tests exist and pass for every enabled service (all envs):
  - Route reachability (HTTP 200/302) per hostname
  - OIDC correctness (issuer discovery works; redirects match configured hostnames; token-based API call succeeds)
  - Zeebe gRPC reachability (in-cluster smoke can obtain a token and connect)
  - Basic “app is functioning” checks for Operate/Tasklist/Connectors (authenticated endpoint responds)

Note: the Connectors bundle does not provide a “web UI” at `/connectors/` and can return a 500 at that path root; use
`/connectors/actuator/health/readiness` (200) for reachability and readiness smoke tests.

## Dev/local seeding (fixed)

Non-production environments (`local`, `dev`, `staging`) are seeded so a human can log in immediately and see all configured UIs:

- OIDC clients and redirect URIs are reconciled via the existing Keycloak realm GitOps flow (`platform-keycloak-realm` client reconciliation).
- Users/groups/role assignments are seeded via the existing platform seeding jobs (`platform-dev-data`) so `local`/`dev`/`staging` converge without manual UI steps.

## Upgrade procedure (standard)

1. Bump pinned version in `sources/charts/third_party/charts.lock.yaml`.
2. Run `./scripts/vendor-charts.sh` and commit updated vendored directories.
3. Update the wrapper chart dependency to the new vendored version.
4. Validate in dev (Argo sync, login, UI reachability, Zeebe health).
5. Promote via GitOps environment promotion flow.

## Acceptance criteria

- A new engineer can deploy Camunda 8 in `dev` by following this doc and creating only:
  - non-secret values files, and
  - secrets in the approved secrets source (AKV/Vault) with documented keys.
- The doc clearly differentiates:
  - what goes in `_shared` vs `env/*`
  - what is vendored vs wrapped
  - which upstream “Ingress” guidance is not applicable because we use Gateway API
- Dev/local “done definition” above is met (fully deployed, OIDC integrated + seeded, smoke tests passing, user can access and see all configured UIs).
