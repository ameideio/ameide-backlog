Here’s a concrete, vendor-aligned way to do this:

* **GitLab on Kubernetes via Helm or Operator**
* **Managed by Argo CD**
* **SSO with Keycloak via OIDC / OmniAuth**

I’ll start with the **Helm + Argo CD + Keycloak** path (most common), then show how the **Operator** fits in. I’ll explicitly point out which bits come straight from GitLab docs so you can cross-check.

---

## 1. What the vendors say (quick cross-check)

**GitLab Helm & Operator**

* GitLab recommends the **Helm chart** as the cloud-native way to install GitLab on Kubernetes. ([GitLab Docs][1])
* The **GitLab Helm repository** is `https://charts.gitlab.io`. ([fnde.gov.br][2])
* **GitLab Operator** is another install method; it just wraps the **same Helm chart** and exposes all chart values under `spec.chart.values` in the `GitLab` custom resource. ([GitLab Docs][3])

**OmniAuth + OpenID Connect**

* GitLab uses **OmniAuth** for SSO providers, including OpenID Connect (`openid_connect`). ([GitLab Docs][4])
* To configure OIDC you define a provider with: `name: "openid_connect"`, a `scope`, `response_type: "code"`, `issuer`, `discovery`, and `client_options.identifier/secret/redirect_uri`. ([GitLab Docs][5])

**Helm chart + OmniAuth**

* The Helm chart exposes Rails `appConfig` (including OmniAuth) via `global.appConfig` in `values.yaml`. ([GitLab Docs][6])
* OmniAuth configuration can be supplied via Kubernetes Secrets referenced from `global.appConfig.omniauth.providers[].secret` and `.key`. This pattern is used in several vendor-aligned examples (IBM, Big Bang), all pointing back to GitLab’s own OmniAuth and OIDC docs. ([IBM DevOps Solution Workbench][7])

So the design you want (Helm or Operator + Argo CD + Keycloak) is exactly how GitLab and those vendors expect you to do it.

---

## 2. Pre-requisites and naming assumptions

I’ll assume:

* Namespace: **`gitlab`**
* GitLab external URL: **`https://gitlab.example.com`**
* Keycloak URL: **`https://keycloak.example.com`**
* Keycloak realm: **`gitlab`**
* Keycloak client ID: **`gitlab`**
* GitOps repo: **`git@gitlab.example.com:infra/gitops.git`**
* Argo CD installed in `argocd` namespace (per Argo docs). ([Argo CD][8])

You can rename everything; just be consistent.

---

## 3. Configure Keycloak for GitLab OIDC

From GitLab’s OIDC + OmniAuth docs plus a Keycloak+GitLab reference example (BigBang), the important things for Keycloak are: ([GitLab Docs][5])

1. **Create / choose a realm** (e.g. `gitlab`).

2. **Create a client** (OIDC) named `gitlab`:

   * Client type: **OpenID Connect**
   * Access type: **confidential**
   * Standard flow: **ON**
   * Valid redirect URIs:
     `https://gitlab.example.com/users/auth/openid_connect/callback`
   * Web origins (or allowed CORS origins):
     `https://gitlab.example.com`
   * Base URL:
     `https://gitlab.example.com`

3. **Client scopes / mappers**
   Ensure GitLab gets **email** and a username-like claim (GitLab often uses `preferred_username` as the UID field in Keycloak examples): ([Big Bang Docs][9])

   * Map **email** → claim `email`
   * Map **username** → claim `preferred_username`
   * Optionally map **name** / **profile** claims.

4. Note:

   * `issuer` → `https://keycloak.example.com/realms/gitlab` (Keycloak >= 17 path style)
     or `https://keycloak.example.com/auth/realms/gitlab` (older Keycloak).
   * `client_id` → `gitlab`
   * `client_secret` → copy from the **Credentials** tab.

These values feed directly into the GitLab OmniAuth OIDC provider config.

---

## 4. Build the OmniAuth provider JSON for Keycloak

GitLab’s OIDC docs say the key options are: `name`, `scope`, `response_type`, `issuer`, `discovery`, `client_auth_method`, and `client_options` (identifier, secret, redirect_uri). ([GitLab Docs][5])

A **Keycloak-specific provider JSON** that follows GitLab’s documented shape might look like this (adjust URLs and IDs):

```jsonc
{
  "name": "openid_connect",
  "label": "Keycloak",
  "args": {
    "name": "openid_connect",
    "scope": ["openid", "profile", "email"],
    "response_type": "code",
    "issuer": "https://keycloak.example.com/realms/gitlab",
    "discovery": true,
    "client_auth_method": "query",
    "uid_field": "preferred_username",
    "client_options": {
      "identifier": "gitlab",
      "secret": "<KEYCLOAK_CLIENT_SECRET>",
      "redirect_uri": "https://gitlab.example.com/users/auth/openid_connect/callback"
    }
  }
}
```

This is structurally the same as the example in GitLab’s OIDC docs (names/values changed) and aligned with the working Keycloak JSON in the BigBang GitLab+Keycloak docs. ([GitLab Docs][5])

---

## 5. Put the provider JSON into a Kubernetes Secret (GitOps-friendly)

Vendor examples (IBM, BigBang) store the full OmniAuth provider JSON in a **Kubernetes Secret**, then reference it from `global.appConfig.omniauth.providers`. ([IBM DevOps Solution Workbench][7])

In your GitOps repo, create something like:

**`gitlab/base/secret-gitlab-oidc-provider.yaml`**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-oidc-provider
  namespace: gitlab
type: Opaque
stringData:
  gitlab-oidc.json: |
    {
      "name": "openid_connect",
      "label": "Keycloak",
      "args": {
        "name": "openid_connect",
        "scope": ["openid", "profile", "email"],
        "response_type": "code",
        "issuer": "https://keycloak.example.com/realms/gitlab",
        "discovery": true,
        "client_auth_method": "query",
        "uid_field": "preferred_username",
        "client_options": {
          "identifier": "gitlab",
          "secret": "<KEYCLOAK_CLIENT_SECRET>",
          "redirect_uri": "https://gitlab.example.com/users/auth/openid_connect/callback"
        }
      }
    }
```

> In production, encrypt this secret in Git (SOPS, SealedSecrets, External Secrets Operator, etc.). The pattern (secret with key containing JSON) is what GitLab-aligned docs use; only the crypto tool is your choice. ([Big Bang Docs][9])

---

## 6. Helm values for GitLab: `global.appConfig.omniauth` with Keycloak

GitLab’s Helm docs state that **`global.appConfig`** controls Rails settings, including OmniAuth, and that **OmniAuth providers can be wired from secrets under `global.appConfig.omniauth.providers`**. ([GitLab Docs][6])

Create a GitLab values file in your GitOps repo:

**`gitlab/base/gitlab-values.yaml`**

```yaml
global:
  hosts:
    domain: gitlab.example.com
    gitlab:
      name: gitlab.example.com
    https: true

  ingress:
    configureCertmanager: false
    tls:
      enabled: true
      secretName: gitlab-tls

  appConfig:
    omniauth:
      enabled: true
      # Only allow OIDC to auto-create users (optional; adjust for your policy)
      allowSingleSignOn:
        - openid_connect
      blockAutoCreatedUsers: false
      syncProfileAttributes:
        - email
      # Sync from which provider(s) – optional but common
      # syncProfileFromProvider:
      #   - openid_connect
      providers:
        - secret: gitlab-oidc-provider
          key: gitlab-oidc.json
```

This matches the structure used in vendor examples:

* `global.appConfig.omniauth.enabled`
* `allowSingleSignOn`, `blockAutoCreatedUsers`, `syncProfileAttributes`
* `providers: - secret: <name> key: <key-in-secret>` ([IBM DevOps Solution Workbench][7])

…and is fully compatible with the GitLab charts’ internal `_omniauth.tpl` template which renders `omniauth` from `global.appConfig.omniauth`. ([about.gitlab.com][10])

You’ll obviously also add Postgres/Redis/object-storage/etc. settings here as per the GitLab Helm deployment guide. ([GitLab Docs][11])

---

## 7. Argo CD Application for GitLab via Helm chart

Argo CD’s Helm docs show how to reference a chart from a Helm repo declaratively. ([Argo CD][12])

### 7.1 Choose a chart version

GitLab publishes a **version mapping table** GitLab-version ↔ Helm chart version; choose a supported combo instead of blindly “latest”. ([GitLab Docs][13])

Example: Chart `9.6.1` ↔ GitLab `18.6.1` (just an example row from the table).

### 7.2 Application manifest

In your GitOps repo, add:

**`argocd/applications/gitlab-helm.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitlab
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://charts.gitlab.io
    chart: gitlab
    targetRevision: 9.6.1          # pick a supported chart version from the mapping table
    helm:
      releaseName: gitlab
      values: |                    # inline helm values (can also use valuesObject)
        # You can also import this file via Application multiple-sources, but keeping
        # it inline here for clarity.
        global:
          hosts:
            domain: gitlab.example.com
            gitlab:
              name: gitlab.example.com
            https: true
          ingress:
            configureCertmanager: false
            tls:
              enabled: true
              secretName: gitlab-tls
          appConfig:
            omniauth:
              enabled: true
              allowSingleSignOn: ['openid_connect']
              blockAutoCreatedUsers: false
              syncProfileAttributes: ['email']
              providers:
                - secret: gitlab-oidc-provider
                  key: gitlab-oidc.json

  destination:
    server: https://kubernetes.default.svc
    namespace: gitlab

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Notes (confirmed from Argo CD Helm docs): ([Argo CD][12])

* `source.chart`, `source.repoURL`, `targetRevision` is the documented pattern for Helm repos.
* You can supply values either via `helm.values` (string), `helm.valuesObject`, or `helm.valueFiles`.
* If you prefer a **separate Git repo for values**, Argo CD ≥ 2.6 supports *multiple sources*, where one source is the Helm repo and another source is a Git repo with values files.

### 7.3 Make sure the OIDC Secret is included

Argo CD just applies manifests; the **Secret** we created needs to be in the same GitOps repo and synced as well. E.g. create another Application or a Kustomize overlay:

**Example: Kustomize for GitLab namespace**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: gitlab
resources:
  - secret-gitlab-oidc-provider.yaml
```

…and either:

* Point a second `Application` at this kustomization, or
* Use Argo CD **ApplicationSet/multiple sources** to compose the secret kustomization with the Helm chart.

As long as Argo CD applies *both* the secret and the Helm release, GitLab will see the provider JSON and enable Keycloak SSO.

---

## 8. Deploying GitLab via **GitLab Operator** in Argo CD (alternative)

Vendor docs make it clear the Operator’s `GitLab` CR just passes values to the same Helm chart under `spec.chart.values`. ([GitLab Docs][14])

### 8.1 Install the GitLab Operator (managed by Argo CD)

Follow GitLab’s operator install doc to get the YAML bundle (namespace `gitlab-system`, CRDs, deployment, RBAC). ([GitLab Docs][15])

Put those manifests in your GitOps repo (or point to a fork), then create an Argo CD Application, e.g.:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitlab-operator
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@gitlab.example.com:infra/gitlab-operator-manifests.git
    targetRevision: main
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: gitlab-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Argo CD now keeps the operator installed and up-to-date.

### 8.2 GitLab custom resource with OIDC + OmniAuth values

GitLab’s operator docs say: *“For more details on configuration options to use under `spec.chart.values`, see our GitLab Helm Chart documentation.”* ([GitLab Docs][14])

So we reuse the **same Helm values** as before, but nest them under `spec.chart.values`.

Create **`gitlab/base/gitlab-cr.yaml`**:

```yaml
apiVersion: apps.gitlab.com/v1beta1
kind: GitLab
metadata:
  name: gitlab
  namespace: gitlab-system
spec:
  chart:
    version: 9.6.1         # same chart version table as before
    values:
      global:
        hosts:
          domain: gitlab.example.com
          gitlab:
            name: gitlab.example.com
          https: true
        ingress:
          configureCertmanager: false
          tls:
            enabled: true
            secretName: gitlab-tls
        appConfig:
          omniauth:
            enabled: true
            allowSingleSignOn: ['openid_connect']
            blockAutoCreatedUsers: false
            syncProfileAttributes: ['email']
            providers:
              - secret: gitlab-oidc-provider
                key: gitlab-oidc.json
```

Then create an Argo CD Application that applies this CR (and the OIDC secret):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitlab-operator-instance
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@gitlab.example.com:infra/gitops.git
    targetRevision: main
    path: gitlab/base
  destination:
    server: https://kubernetes.default.svc
    namespace: gitlab-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The Operator watches `GitLab` resources in `gitlab-system`, reads `spec.chart.values` (including your OmniAuth config), and orchestrates the underlying Helm install/upgrade accordingly. ([GitLab Docs][14])

---

## 9. Upgrades / “latest version” with Argo CD

Vendor guidance you should follow:

* Use **supported chart ↔ GitLab combos** from the official mapping table. ([GitLab Docs][13])
* If you use the **Operator**, they currently require **minor-by-minor** upgrades (no big jumps). ([GitLab Docs][3])

With Argo CD that looks like:

1. Update **`targetRevision`** (Helm case) or `spec.chart.version` (Operator case) to the desired next chart version.
2. Commit and push to Git.
3. Let Argo CD sync; it will:

   * Re-template the Helm chart at the new version.
   * Apply changes.
   * Operator (if used) performs the recommended zero-downtime upgrade.

---

## 10. Summary: how this matches vendor docs

You asked to “verify vendor documentation,” so here’s a concise mapping:

* **Install method**:

  * GitLab Helm chart (Kubernetes) and GitLab Operator are both official, documented installation methods. ([GitLab Docs][1])

* **Helm repo / chart**:

  * `repoURL: https://charts.gitlab.io` and `chart: gitlab` matches GitLab Helm repo docs. ([fnde.gov.br][2])

* **Argo CD Application (Helm)**:

  * Using `source.chart`, `source.repoURL`, `targetRevision`, `source.helm.values` is exactly the pattern from Argo CD Helm documentation. ([Argo CD][12])

* **OmniAuth + OIDC provider structure**:

  * `name: "openid_connect"`, `scope`, `response_type`, `issuer`, `discovery`, and `client_options.identifier/secret/redirect_uri` follow GitLab’s “Use OpenID Connect as an authentication provider” doc. ([GitLab Docs][5])

* **Helm chart OmniAuth wiring**:

  * Using `global.appConfig.omniauth.enabled`, `allowSingleSignOn`, `blockAutoCreatedUsers`, and `providers: - secret/key` matches examples from GitLab-aligned docs (IBM, BigBang) which defer to GitLab’s OmniAuth + Helm chart documentation. ([GitLab Docs][6])

* **Operator CR**:

  * `apiVersion: apps.gitlab.com/v1beta1`, `kind: GitLab`, `spec.chart.version`, `spec.chart.values` is the format shown in GitLab Operator docs and examples, which explicitly state that `spec.chart.values` accepts standard Helm chart values. ([GitLab Docs][14])

If you paste in your **actual** Argo CD / values / CR YAML later, I can sanity-check it line-by-line against these docs for you.

[1]: https://docs.gitlab.com/charts/?utm_source=chatgpt.com "GitLab Helm chart"
[2]: https://www.fnde.gov.br/repositorio/help/install/kubernetes/index.md?utm_source=chatgpt.com "Index · Kubernetes · Install · Help · GitLab"
[3]: https://docs.gitlab.com/operator/?utm_source=chatgpt.com "GitLab Operator"
[4]: https://docs.gitlab.com/integration/omniauth/?utm_source=chatgpt.com "OmniAuth | GitLab Docs"
[5]: https://docs.gitlab.com/administration/auth/oidc/?utm_source=chatgpt.com "Use OpenID Connect as an authentication provider"
[6]: https://docs.gitlab.com/charts/charts/globals/?utm_source=chatgpt.com "Configure charts using globals"
[7]: https://fswb-documentation.knowis.net/1.0/Installation/content/post_install_configuration.html?utm_source=chatgpt.com "Post install configuration - IBM DevOps Solution Workbench"
[8]: https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/?utm_source=chatgpt.com "Installation - Argo CD - Declarative GitOps CD for Kubernetes"
[9]: https://docs-bigbang.dso.mil/latest/packages/gitlab/docs/keycloak/ "Keycloak - Big Bang Docs"
[10]: https://gitlab.com/gitlab-org/charts/gitlab/-/blob/master/charts/gitlab/templates/_omniauth.tpl?utm_source=chatgpt.com "charts/gitlab/templates/_omniauth.tpl · master"
[11]: https://docs.gitlab.com/charts/installation/?utm_source=chatgpt.com "Installing GitLab by using Helm"
[12]: https://argo-cd.readthedocs.io/en/stable/user-guide/helm/ "Helm - Argo CD - Declarative GitOps CD for Kubernetes"
[13]: https://docs.gitlab.com/charts/installation/version_mappings/?utm_source=chatgpt.com "GitLab Helm chart versions"
[14]: https://docs.gitlab.com/operator/developer/installation/?utm_source=chatgpt.com "Installation"
[15]: https://docs.gitlab.com/operator/installation/?utm_source=chatgpt.com "Installation"
