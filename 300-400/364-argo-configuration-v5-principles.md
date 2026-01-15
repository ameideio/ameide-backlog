Below is a **vendor‑backed audit checklist** you can run against your repo(s) and clusters to ensure the implementation is 100% aligned with Argo CD / ApplicationSet guidance. Each principle includes a **source link to official docs** so you (or auditors) can verify behavior and wording directly.

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

> **Repo structure refresher:** Helm charts & values now live inside the GitOps repo (`sources/charts/**`, `sources/values/**`), and component folders (`gitops/ameide-gitops/environments/<env>/components/<tier>`) hold metadata only. ApplicationSets pull everything from this repo via the Git generator and render Helm directly.

---

## A. ApplicationSet orchestration (progressive, health‑gated waves)

1. **Enable Progressive Syncs explicitly**
   Ensure the feature is turned on via **one** of the documented methods (`--enable-progressive-syncs` flag, env var, or `argocd-cmd-params-cm`). ([Argo CD][1])

2. **Use RollingSync for wave ordering**

* Steps defined with **label selectors** (`matchExpressions`) and optional **`maxUpdate`** to throttle.
* **Next step waits** until **all** Applications in the current step are **Healthy**. ([Argo CD][1])

3. **Understand how RollingSync drives syncs**

* **Autosync is forced OFF** on generated Applications (warnings are logged if `automated:` is set).
* RollingSync triggers syncs as if the UI/CLI Sync button was pressed—**respects Sync Windows** and preserves per‑Application **retry**/sync settings. ([Argo CD][1])

4. **Set deletion order for safe tear‑downs**
   Use `strategy.deletionOrder: Reverse` so deletion runs **frontends → backends → data → infra** (reverse of steps). ([Argo CD][1])

5. **Multi‑cluster fan‑out is first‑class**
   When needed, use **Matrix** (`clusters` × `git`) to stamp per‑cluster Applications while keeping the same RollingSync steps. ([Argo CD][2])

---

## B. Generators & templates (how you stamp Applications)

6. **Choose the correct Git generator subtype**

* Use **Git *Files*** when your per‑component metadata is in JSON/YAML files (values become template params).
* Use **Git *Directories*** when directory names are your parameters. ([Argo CD][3])

7. **ApplicationSet is the place that “makes Applications”**
   The controller **generates** `Application` objects from the `ApplicationSet` spec and keeps them up‑to‑date. ([Argo CD][4])

---

## C. AppProjects (guardrails, tenancy, governance)

8. **Constrain sources, destinations, and kinds**
   `AppProject` must restrict **`sourceRepos`**, **`destinations`**, and **(namespace/cluster) resource allow/deny lists**; use **Sync Windows** if needed. ([Argo CD][5])

9. **Use Sync Windows for change control**
   Define allow/deny windows in projects; RollingSync obeys them because it triggers standard syncs. ([Argo CD][6])

---

## D. Health‑gating & hooks (what “Healthy” means)

10. **Custom health checks for CRDs**
    If a resource (e.g., DB operator/instances, ExternalSecrets) lacks a built‑in health check, add **Lua health** so “Healthy” truly means ready—RollingSync depends on this. ([Argo CD][7])

11. **Resource hooks for migrations/tests**
    Use `PreSync`/`PostSync` hooks for migrations and smoke tests that must pass before the next wave. ([Argo CD][8])

12. **Resource‑level sync waves remain valid**
    Inside a single `Application`, use **sync waves** to order CRDs → operators → CRs. (This complements RollingSync, it doesn’t replace it.) ([Argo CD][9])

---

## E. Diff & drift controls (OutOfSync hygiene)

13. **Prefer surgical Diff Customization over blanket ignores**
    Use **`ignoreDifferences`** (JSON Pointers/JQ) and/or **`managedFieldsManagers`** to ignore controller‑mutated fields; this is the standard, vendor‑documented approach. ([Argo CD][10])

14. **Honor ignore rules during *sync***
    Enable **`RespectIgnoreDifferences=true`** in `spec.syncPolicy.syncOptions` so ignores apply during **diff and sync**. ([Argo CD][11])

15. **Use Compare Options only when you truly want to skip a resource in sync status**
    `argocd.argoproj.io/compare-options: IgnoreExtraneous` **removes a resource from sync status** (health can still degrade). Use it for generated artifacts (e.g., Kustomize generators), not for general controller noise. ([Argo CD][12])

---

## F. Sync Options (only what you need, documented behavior)

16. **CreateNamespace** when needed
    Let Argo CD create the target namespace and manage its metadata. ([Argo CD][11])

17. **SkipDryRunOnMissingResource** for CRDs+CRs in one graph
    Avoid dry‑run failures when CRDs and their CRs land in the same sync. ([Argo CD][11])

18. **PruneLast** for safe pruning after healthy rollout
    Prune as a final implicit wave once all updates are Healthy. ([Argo CD][11])

19. **ApplyOutOfSyncOnly** for scale
    For very large apps, sync only out‑of‑sync resources to reduce API pressure. ([Argo CD][11])

20. **ServerSideApply / Replace / Force** (use sparingly, with intent)
    Each has documented behaviors and risks; only enable when you actually need that mode. ([Argo CD][11])

---

## G. Secrets (how Argo CD says to do it)

21. **Populate secrets on the destination cluster**
    Vendor guidance prefers populating Secrets **in‑cluster** (e.g., External Secrets Operator with Vault) rather than injecting secrets during manifest generation. ([Argo CD][13])

---

## H. Declarative install & lifecycle hardening

22. **Declarative setup for Apps/Projects/Settings**
    Manage everything (Applications, AppProjects, config) as manifests in the `argocd` namespace. ([Argo CD][14])

23. **Use the deletion finalizer**
    Add `resources-finalizer.argocd.argoproj.io` to Applications to ensure **cascading deletes** behave as expected. Background mode is also supported. ([Argo CD][15])

24. **Orphaned Resources monitoring**
    Enable at the Project level to flag resources in target namespaces that don’t belong to any Application. ([Argo CD][16])

---

## I. Resource tracking (who owns what)

25. **Choose and document your tracking method**
    Understand default **label‑based** tracking (`app.kubernetes.io/instance`) and its limits; consider **annotation‑based** tracking (or combo) if you run multiple Argo CD instances or hit label length/conflicts. ([Argo CD][17])

26. **Use documented annotations/labels, don’t invent your own**
    For compare options, sync options, waves, tracking id, etc., use the keys from the official list. ([Argo CD][18])

---

## J. What to **avoid** (per docs behavior)

27. **Don’t rely on autosync inside RollingSync**
    RollingSync **disables autosync** on generated Applications—set your intent in the **ApplicationSet strategy**, not per‑app `automated:`. ([Argo CD][1])

28. **Don’t use Compare‑Options to hide real drift**
    `IgnoreExtraneous` is coarse. Prefer **`ignoreDifferences`** for specific fields so Argo CD still tracks everything else. ([Argo CD][12])

---

## K. Minimal “golden” snippets (doc‑aligned)

**1) Enable Progressive Syncs (any one method)**

```yaml
# argocd-cmd-params-cm
apiVersion: v1
kind: ConfigMap
metadata: { name: argocd-cmd-params-cm, namespace: argocd }
data:
  applicationsetcontroller.enable.progressive.syncs: "true"
```

([Argo CD][1])

**2) RollingSync with Reverse deletion**

```yaml
spec:
  strategy:
    type: RollingSync
    deletionOrder: Reverse
    rollingSync:
      steps:
        - matchExpressions: [{ key: wave, operator: In, values: ["-10"] }]
        - matchExpressions: [{ key: wave, operator: In, values: ["-5"] }]
        - matchExpressions: [{ key: wave, operator: In, values: ["0"] }]
        - matchExpressions: [{ key: wave, operator: In, values: ["1"] }]
```

(Autosync will be disabled on generated Applications.) ([Argo CD][1])

**3) ExternalSecrets (example) — diff customization + respect during sync**
*App‑level (narrow scope):*

```yaml
spec:
  ignoreDifferences:
    - group: external-secrets.io
      kind: ExternalSecret
      jsonPointers:
        - /status
  syncPolicy:
    syncOptions:
      - RespectIgnoreDifferences=true
```

([Argo CD][10])

**4) Sync options you will actually use**

```yaml
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - SkipDryRunOnMissingResource=true
      - PruneLast=true
      # - ApplyOutOfSyncOnly=true   # (very large apps)
```

([Argo CD][11])

**5) Git generator (Files) when reading component YAML**

---

## L. Operational snapshot (2025‑11‑13)

* **Charts & values are fully local.** All third-party Helm packages are committed under `sources/charts/third_party/**`, and ApplicationSets use a secondary `$values` source so Helm always includes `_shared/<tier>/<name>.yaml` plus the environment overlay.
* **Redis operator currently missing.** Running `kubectl -n redis-operator get pods` returns “No resources found”, so the redis operator + failover Applications remain `OutOfSync`. Fix the image reference (`quay.io/spotahome/redis-operator:v1.2.4` or a mirrored tag) before progressing the foundation/data waves.
* **What still blocks “all green”:**
  1. Publish or mirror every `docker.io/ameide/*:dev` image (agents, platform, workflows runtime, namespace guard) or add `imagePullSecrets` so pods stop hitting `ImagePullBackOff`.
  2. Recreate pgAdmin bootstrap/OAuth secrets (or keep OAuth disabled) so the Deployment leaves `CreateContainerConfigError`.
  3. Delete the failed `master-realm-*` jobs and re-sync `platform-keycloak-realm` to validate the updated realm export.
  4. Prune Helmfile-era resources once each namespace is owned by Argo; only then will `OrphanedResourceWarning` highlight real drift.

```yaml
spec:
  generators:
    - git:
        repoURL: https://github.com/your-org/gitops.git
        revision: HEAD
        files:
          - path: environments/dev/components/**/component.yaml
```

([Argo CD][3])

---

## L. Quick audit worksheet (what to check, fast)

* **Feature flag:** Progressive Syncs enabled? (ops config) ([Argo CD][1])
* **Strategy:** `type: RollingSync` with labeled **steps** and (optional) `maxUpdate`? **`deletionOrder: Reverse`** present? ([Argo CD][1])
* **Autosync:** Not set on generated Applications (or at least understood it’s disabled)? ([Argo CD][1])
* **Generators:** Using **`files:`** when consuming per‑component config? ([Argo CD][3])
* **Projects:** `sourceRepos`/`destinations`/resource allowlists set + **Sync Windows** (if needed)? ([Argo CD][5])
* **Health:** Lua checks in `argocd-cm` for operator CRDs that gate waves? ([Argo CD][7])
* **Hooks:** Migrations/smoke as `PreSync`/`PostSync`? ([Argo CD][8])
* **Diff:** `ignoreDifferences` for mutated fields (e.g., ExternalSecrets), plus **`RespectIgnoreDifferences=true`**? ([Argo CD][10])
* **Sync options:** Only those needed (`CreateNamespace`, `SkipDryRunOnMissingResource`, `PruneLast`, optional `ApplyOutOfSyncOnly`)? ([Argo CD][11])
* **Secrets:** In‑cluster population approach documented (ESO/Vault)? ([Argo CD][13])
* **Declarative:** Apps/Projects/Settings applied as YAML in `argocd` ns? ([Argo CD][14])
* **Deletion:** `resources-finalizer.argocd.argoproj.io` on Applications? ([Argo CD][15])
* **Orphans:** Project-level orphaned resources monitoring enabled? ([Argo CD][16])
* **Tracking:** Resource tracking method chosen & documented (label vs annotation)? ([Argo CD][17])

---

## M. Helm-sourced Applications (pivot checklist)

1. **Use native goTemplate** (`spec.goTemplate: true`) when you need conditionals; ApplicationSet already understands Go templates, so no external renderer is required. ([Argo CD][5])
2. **Helm source block** must set `repoURL`, `chart`/`path`, and `targetRevision`. Add `helm.valueFiles`, `skipCrds`, `ignoreMissingValueFiles`, and `passCredentials` as needed. ([Argo CD][6])
3. **CRD-heavy charts:** prefer splitting CRDs into an earlier wave and setting `helm.skipCrds=true`, or keep CRDs with the chart but add sync options like `SkipDryRunOnMissingResource=true` / `PruneLast=true`. ([Argo CD][6])
4. **Git-hosted charts:** vendor dependencies or publish to a Helm/OCI repo—Argo will not run `helm dependency update` for you. ([Stack Overflow][8])
5. **Private repos/OCI:** register secrets in Argo and set `helm.passCredentials=true` when the chart and repo differ. ([Argo CD][6])
6. **Hooks:** rely on Helm hook annotations when they match your lifecycle; otherwise translate the job into an Argo resource hook (`argocd.argoproj.io/hook`). ([Argo CD][2])
7. **Multiple sources:** when values live outside the chart repo, use the documented `sources` + `$values` capability instead of pre-rendering manifests. ([Argo CD][3])
8. **Template best practice:** only template string fields. Lists/objects (e.g., `helm.valueFiles`, `syncOptions`) must stay static unless they’re injected via `templatePatch`. Use `ignoreMissingValueFiles: true` to list shared + env files without per-component arrays. ([Argo CD][5])

---

If you want, I can run through your current `ApplicationSet`/`AppProject`/`argocd-cm` manifests and produce an **annotated diff** that flags any gaps against the checklist above.

[1]: https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Progressive-Syncs/ "Progressive Syncs - Argo CD - Declarative GitOps CD for Kubernetes"
[2]: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Matrix/?utm_source=chatgpt.com "Matrix Generator - Declarative GitOps CD for Kubernetes"
[3]: https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-Git/ "Git Generator - Argo CD - Declarative GitOps CD for Kubernetes"
[4]: https://argo-cd.readthedocs.io/en/latest/user-guide/application-set/?utm_source=chatgpt.com "Generating Applications with ApplicationSet - Argo CD"
[5]: https://argo-cd.readthedocs.io/en/stable/user-guide/projects/?utm_source=chatgpt.com "Projects - Argo CD - Declarative GitOps CD for Kubernetes"
[6]: https://argo-cd.readthedocs.io/en/release-2.9/user-guide/sync_windows/?utm_source=chatgpt.com "Sync Windows - Declarative GitOps CD for Kubernetes"
[7]: https://argo-cd.readthedocs.io/en/latest/operator-manual/health/?utm_source=chatgpt.com "Resource Health - Declarative GitOps CD for Kubernetes"
[8]: https://argo-cd.readthedocs.io/en/release-2.14/user-guide/resource_hooks/?utm_source=chatgpt.com "Resource Hooks - Declarative GitOps CD for Kubernetes"
[9]: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/?utm_source=chatgpt.com "Sync Phases and Waves - Argo CD - Read the Docs"
[10]: https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/?utm_source=chatgpt.com "Diff Customization - Declarative GitOps CD for Kubernetes"
[11]: https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/ "Sync Options - Argo CD - Declarative GitOps CD for Kubernetes"
[12]: https://argo-cd.readthedocs.io/en/stable/user-guide/compare-options/?utm_source=chatgpt.com "Compare Options - Declarative GitOps CD for Kubernetes"
[13]: https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/?utm_source=chatgpt.com "Secret Management - Declarative GitOps CD for Kubernetes"
[14]: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/?utm_source=chatgpt.com "Declarative Setup - Argo CD - Read the Docs"
[15]: https://argo-cd.readthedocs.io/en/latest/user-guide/app_deletion/?utm_source=chatgpt.com "App Deletion - Argo CD - Declarative GitOps CD for Kubernetes"
[16]: https://argo-cd.readthedocs.io/en/latest/user-guide/orphaned-resources/?utm_source=chatgpt.com "Orphaned Resources Monitoring - Argo CD - Read the Docs"
[17]: https://argo-cd.readthedocs.io/en/latest/user-guide/resource_tracking/?utm_source=chatgpt.com "Resource Tracking - Declarative GitOps CD for Kubernetes"
[18]: https://argo-cd.readthedocs.io/en/latest/user-guide/annotations-and-labels/?utm_source=chatgpt.com "Annotations and Labels used by Argo CD - Read the Docs"
