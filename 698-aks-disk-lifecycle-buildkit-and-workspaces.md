---
title: "698 – AKS disk lifecycle: buildkit + workspaces"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-19
suite: "aks"
source_of_truth: false
---

# 698 – AKS disk lifecycle: buildkit + workspaces

## Evidence

- Snapshot: `backlog/695-aks-disk-usage-snapshot.md`
  - Unattached managed disks can remain billed even when the corresponding pods are scaled to 0.
  - BuildKit cache PVCs are a common source when `StatefulSet` replicas are scaled down.

## Contract (target state)

- “Scale-to-zero” components must not leave long-lived billed disks by default.
- Any deliberate long-lived caches must be explicitly documented as “persistent cost”.

## Implementation (current repo)

### BuildKit

- StatefulSet sets `persistentVolumeClaimRetentionPolicy`:
  - `whenScaled: Delete`
  - `whenDeleted: Delete`
- BuildKit cache PVC size is configurable:
  - `buildkitd.stateVolumeSize` in `sources/values/_shared/cluster/buildkit.yaml`

Operational note:
- If BuildKit is already at `replicas: 0` with existing PVCs, scaling up then back down will trigger PVC deletion via the retention policy.

## Follow-ups (recommended)

- Add a PVC lifecycle strategy for Che/Coder workspace volumes (TTL / janitor / retention policy where applicable).
- Define a standard retention window for “workspace-like” PVCs to keep unattached disk costs bounded.

