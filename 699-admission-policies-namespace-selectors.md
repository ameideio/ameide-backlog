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

## Implementation (current repo)

- Workspace dev routing policy uses VAP and namespace label selectors:
  - `workspace-innerloop-httproute-shape` (cluster-scoped)
- App HTTPRoute policy is deployed via ArgoCD and selects by label:
  - `disallow-app-httproute-on-http` with `namespaceSelector: gateway-access=allowed`
  - deployed as `cluster-disallow-app-httproute-on-http` (cluster-scoped ArgoCD Application)

## Follow-ups (recommended)

- Decide whether to introduce a policy engine (Kyverno) for mutate/generate use cases, or keep validation-only via VAP.
- Add a baseline “image policy” strategy (latest-tag ban, require digest) with per-environment enforcement.
