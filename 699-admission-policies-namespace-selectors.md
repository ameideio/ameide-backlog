---
title: "699 – Admission policies: namespace selectors + rollout"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-19
suite: "policy"
source_of_truth: false
---

# 699 – Admission policies: namespace selectors + rollout

## Problem

Admission policies are only effective if they are:
- deployed by GitOps, and
- scoped using selectors that match real namespaces (labels), not hard-coded names that can drift.

## Contract (target state)

- Policies select namespaces by stable labels:
  - `ameide.io/environment`
  - `ameide.io/managed-by=gitops`
  - and/or `gateway-access=allowed` where appropriate.
- Enforcement is environment-aware (recommended):
  - prod: `Deny`
  - dev: `Warn`/`Audit`

## Namespace label contract (normative)

Namespaces managed by GitOps MUST carry these labels (rendered by `foundation-namespaces`):

```yaml
metadata:
  labels:
    ameide.io/managed-by: gitops
    ameide.io/environment: dev|staging|production|local
    gateway-access: allowed   # only where HTTPRoutes are expected
```

Policies MUST select by these labels (or `kubernetes.io/metadata.name` only for truly singleton namespaces such as `che-devcluster`).

## Rollout sequence (recommended)

1. Deploy bindings in `Warn`+`Audit` everywhere.
2. Fix violations (or explicitly exclude known safe exceptions).
3. Flip production bindings to `Deny`.
4. Keep dev on `Warn`/`Audit` to reduce developer friction.

## VAP vs Kyverno (decision rule)

- Use **ValidatingAdmissionPolicy (VAP)** for validation-only guardrails.
- Use **Kyverno** only when mutation/generation is required (and do not duplicate the same control in both systems).

## Implementation (current repo)

- Workspace dev routing policy uses VAP and namespace label selectors:
  - `workspace-innerloop-httproute-shape` (cluster-scoped)
- App HTTPRoute policy is deployed via ArgoCD and selects by label:
  - `disallow-app-httproute-on-http` with `namespaceSelector: gateway-access=allowed`
  - deployed as `cluster-disallow-app-httproute-on-http` (cluster-scoped ArgoCD Application)

## Verification (practical)

- Confirm selector matches real namespaces:
  - `kubectl get ns -l gateway-access=allowed`
- Confirm bindings exist:
  - `kubectl get validatingadmissionpolicybindings.admissionregistration.k8s.io`
- Negative test (should be denied): apply an `HTTPRoute` that attaches to `sectionName: http` in a `gateway-access=allowed` namespace.

## Follow-ups (recommended)

- Decide whether to introduce a policy engine (Kyverno) for mutate/generate use cases, or keep validation-only via VAP.
- Add a baseline “image policy” strategy (latest-tag ban, require digest) with per-environment enforcement.
