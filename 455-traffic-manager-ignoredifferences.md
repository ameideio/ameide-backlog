# 455: Traffic-Manager ignoreDifferences Namespace Templating

**Status**: Implemented
**Related**: [446-namespace-isolation.md](446-namespace-isolation.md), [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md)

## Problem Statement

Traffic-manager (Telepresence) creates MutatingWebhookConfigurations with names that include the namespace:
- `agent-injector-webhook-ameide-dev`
- `agent-injector-webhook-ameide-staging`
- `agent-injector-webhook-ameide-prod`

These webhooks are modified at runtime by:
1. **Traffic-manager itself** - generates `caBundle` dynamically at startup
2. **AKS Admission Enforcer** - injects `namespaceSelector` and `objectSelector`

The original `ignoreDifferences` configuration used a static name (`agent-injector-webhook-ameide`) which didn't match any actual resource, causing perpetual OutOfSync status.

## Solution

### 1. Namespace Templating in ignoreDifferences

Component files can now use `{{ .namespace }}` in ignoreDifferences name fields:

```yaml
# environments/_shared/components/platform/control-plane/telepresence/component.yaml
ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: MutatingWebhookConfiguration
    name: agent-injector-webhook-{{ .namespace }}
    jsonPointers:
      - /webhooks/0/clientConfig/caBundle
      - /webhooks/0/namespaceSelector
      - /webhooks/0/objectSelector
```

### 2. ApplicationSet templatePatch Enhancement

The `ameide` ApplicationSet templatePatch now processes ignoreDifferences to resolve namespace placeholders:

```yaml
# argocd/applicationsets/ameide.yaml (templatePatch section)
{{- if hasKey . "ignoreDifferences" }}
ignoreDifferences:
  {{- range .ignoreDifferences }}
  - {{- if hasKey . "group" }}
    group: {{ .group }}
    {{- end }}
    kind: {{ .kind }}
    {{- if hasKey . "name" }}
    {{- $resolvedName := replace "{{ .namespace }}" $ns .name }}
    name: {{ $resolvedName }}
    {{- end }}
    {{- if hasKey . "jsonPointers" }}
    jsonPointers:
      {{- toYaml .jsonPointers | nindent 12 }}
    {{- end }}
    {{- if hasKey . "jqPathExpressions" }}
    jqPathExpressions:
      {{- toYaml .jqPathExpressions | nindent 12 }}
    {{- end }}
    {{- if hasKey . "managedFieldsManagers" }}
    managedFieldsManagers:
      {{- toYaml .managedFieldsManagers | nindent 12 }}
    {{- end }}
  {{- end }}
{{- end }}
```

Key implementation details:
- Uses `replace "{{ .namespace }}" $ns .name` to substitute the placeholder
- Uses `hasKey` checks for all optional fields to avoid template errors
- Supports all ignoreDifferences fields: `group`, `kind`, `name`, `jsonPointers`, `jqPathExpressions`, `managedFieldsManagers`

## Architecture Alignment

This implementation follows the dual ApplicationSet model from [446-namespace-isolation.md](446-namespace-isolation.md):

- **Traffic-manager is environment-scoped**: Deployed per-environment via the `ameide` ApplicationSet
- **Webhook names are namespace-specific**: Each environment gets its own webhook
- **Template resolution happens in ApplicationSet**: The `$ns` variable comes from the matrix generator

## Ignored Fields

The following fields are ignored for traffic-manager MutatingWebhookConfigurations:

| Field | Reason |
|-------|--------|
| `/webhooks/0/clientConfig/caBundle` | Generated dynamically by traffic-manager at startup |
| `/webhooks/0/namespaceSelector` | Injected by AKS Admission Enforcer |
| `/webhooks/0/objectSelector` | Injected by AKS Admission Enforcer |

Additionally, the `mutator-webhook-tls` Secret has TLS certificate fields ignored:
- `/data/ca.crt`
- `/data/tls.crt`
- `/data/tls.key`

## Operational Notes

### Applying Changes to ApplicationSet

The ApplicationSet templatePatch is embedded in the ApplicationSet resource itself. Changes to `argocd/applicationsets/ameide.yaml` require:

```bash
# Force replace to update templatePatch
kubectl replace -f argocd/applicationsets/ameide.yaml --force
```

A simple `kubectl apply` may not update the templatePatch due to strategic merge behavior.

### Verification

After changes, verify the ignoreDifferences resolved correctly:

```bash
# Check resolved name (should show actual namespace, not template)
kubectl get app dev-traffic-manager -n argocd -o jsonpath='{.spec.ignoreDifferences[0].name}'
# Expected: agent-injector-webhook-ameide-dev

# Check sync status
kubectl get app -n argocd | grep traffic
# Expected: Synced / Healthy for all environments
```

## Other Components Using This Pattern

Components that may benefit from namespace templating in ignoreDifferences:
- Any component creating cluster-scoped resources with namespace-specific names
- Webhook configurations that include namespace in their name
- Resources modified by namespace-specific controllers

## Implementation Commits

- `fix(traffic-manager): support namespace-templating in ignoreDifferences`
