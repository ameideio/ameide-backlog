# 374 – ArgoCD ApplicationSet YAML Safety

## Context
During the devcontainer bootstrap, the `kubectl apply` step failed while loading ApplicationSets (`apps.yaml`, `data-services.yaml`, etc.). The manifests contained inline Go template expressions such as:

```yaml
valueFiles: {{- $valuesEnv := ... -}}{{ toJson (default $defaultValueFiles .chart.valueFiles) }}
```

`kubectl` must parse ApplicationSet CRs as plain YAML before ArgoCD ever sees them, so these inline expressions cause YAML parsing errors (`did not find expected node content`). The bootstrap stops at the first such error, leaving k3d and ArgoCD partially initialized.

## Decision
Follow Argo CD’s supported pattern for non-string templating: keep the base `.spec.template` pure YAML and move array/object templating into `templatePatch` with `goTemplate: true`.

- **`templatePatch`** now renders the entire `sources` list (including Helm value file logic) plus `syncPolicy`, so kubectl only sees literal YAML while the ApplicationSet controller performs the Go templating. This mirrors the vendor docs (“Templates → Template Patch”).
- **`template`** remains simple (string substitutions only) which keeps manifests readable and kubectl-friendly.
- **Safety net** – continue running `kubectl apply --dry-run=client -n argocd -f gitops/ameide-gitops/environments/dev/argocd/apps/*.yaml` locally/CI to catch syntax regressions.
- **Argo controller context quirks** – `templatePatch` executes outside the ApplicationSet object, so references to `$.metadata.*` (or other fields not passed through the generator) fail. Hard-code or pass the needed values via generator parameters, and use helpers such as `get .chart "valueFiles"` instead of `.chart.valueFiles` to avoid missing-key panics.

## Follow-up Tasks
- [ ] Add a lint/check target (Makefile, CI job, or git hook) that dry-runs all ApplicationSet manifests.
- [ ] Document the “kubectl-first, templating-second” rule in the contributor guide for GitOps assets.
- [ ] Audit other environments (stage, prod) for the same pattern and fix as needed.
