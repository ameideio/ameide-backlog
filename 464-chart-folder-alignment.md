# 464 – Chart Folder Alignment

**Created**: 2025-12-07

> **Cross-References (Deployment Architecture Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | **READ THIS FIRST**: How the two ApplicationSets work |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Rollout phases & sub-phase patterns (`*10`, `*20`, `*50`, etc.) |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | Secrets handling, OIDC client extraction patterns |
> | [473-ameide-technology.md](473-ameide-technology.md) | Technology architecture (how Backstage, Temporal fit in) |
> | [490-vendor-charts-alignment.md](490-vendor-charts-alignment.md) | Vendoring script/CI/documentation alignment for third-party charts |
>
> **Related (Implementation Details)**:
> - [459-httproute-ownership.md](459-httproute-ownership.md) – HTTPRoute ownership patterns

## Overview

Align chart folder structure with rollout sequence stages and component domains.

## Principles

1. **Third-party charts should always be vendored** in `sources/charts/third_party/`
2. **Filesystem folders should mimic rollout sequence stages** (matching component domains)
3. **All configurations should follow appropriate folders** (charts, values, components aligned)
4. **ApplicationSet generator paths must be updated** when adding new component domains

## ApplicationSet Generator Paths

The `argocd/applicationsets/ameide.yaml` file contains git file generators that scan for component.yaml files. When adding a new domain folder under `environments/_shared/components/`, the generator must be updated.

**Current paths** (in `spec.generators[].matrix.generators[].git.files`):
```yaml
- path: environments/_shared/components/apps/**/component.yaml
- path: environments/_shared/components/data/**/component.yaml
- path: environments/_shared/components/foundation/**/component.yaml
- path: environments/_shared/components/observability/**/component.yaml
- path: environments/_shared/components/platform/**/component.yaml
```

**Note**: The `cluster/` domain is handled by a separate ApplicationSet (`argocd/applicationsets/cluster.yaml`).

## Current State

| Layer | Charts Dir | Values Dir | Component Domain | Rollout Phase |
|-------|-----------|------------|------------------|---------------|
| crds | - | _shared/cluster | cluster | 010 |
| operators | - | _shared/cluster | cluster | 020 |
| configs | cluster | _shared/cluster ✅ | cluster | 030 |
| foundation | foundation | foundation | foundation | 115-199 |
| data | data | data | data | 230-299, 450-499 |
| platform | platform ✅ | platform | platform | 310-399 |
| observability | observability ✅ | observability | observability | 530-599 |
| apps | apps | apps | apps | 650-699 |

## Gaps

### GAP-1: `platform-layers/` should be `platform/`

**Current**: `sources/charts/platform-layers/`
**Expected**: `sources/charts/platform/` (matches values dir and component domain)

**Charts affected**:
- common
- coredns-config
- db-migrations
- gitlab
- langfuse-bootstrap
- namespace
- pgadmin
- registry-alias
- registry-mirror
- temporal-cluster

### GAP-2: Plausible in wrong location

**Current**: `sources/charts/apps/plausible/`
**Problem**: Plausible is a third-party analytics tool, not a first-party app

**Recommendation**: Move to `observability/` since:
- Plausible is analytics/observability tooling
- Component rolloutPhase 650 is apps layer, but domain should be `observability`
- Aligns with other observability tools (Grafana, Prometheus, Langfuse)
- Values should be in `sources/values/_shared/observability/plausible.yaml`

### GAP-3: Observability missing charts folder

**Current**: `sources/values/_shared/observability/` exists
**Missing**: `sources/charts/observability/` does not exist

Observability should follow the same pattern as other layers:
- Create `sources/charts/observability/` for wrapper charts
- Move observability-related custom charts there (if any)
- Third-party charts remain in `third_party/` but wrappers/configs go in `observability/`

### GAP-4: Orphaned/unused charts (RESOLVED)

**6 charts were truly orphaned and removed:**

| Chart | Location | Notes |
|-------|----------|-------|
| common | `sources/charts/platform-layers/common/` | Superseded by `foundation/common/raw-manifests` |
| coredns-config | `sources/charts/platform-layers/coredns-config/` | Unused |
| gitlab | `sources/charts/platform-layers/gitlab/` | Unused |
| namespace | `sources/charts/platform-layers/namespace/` | Unused |
| registry-alias | `sources/charts/platform-layers/registry-alias/` | Unused |
| registry-mirror | `sources/charts/platform-layers/registry-mirror/` | Unused |

**Note**: `sources/charts/cluster/gateway/` was NOT orphaned - it was missing a component.yaml. Now properly managed by ApplicationSet (see GAP-4b).

### GAP-4b: Cluster config values inconsistency (RESOLVED)

**Problem**: Cluster-scoped config values were in `sources/values/cluster/` instead of `_shared/cluster/`:
- `coredns-config.yaml` used explicit `valueFiles` workaround
- `cluster-gateway.yaml` had redundant "cluster-" prefix
- Components declared `domain: configs` instead of `domain: cluster`

**Solution**: Aligned to convention:
- Moved values to `sources/values/_shared/cluster/{name}.yaml`
- Created `gateway` component.yaml for ApplicationSet management
- Changed component domains from `configs` to `cluster`
- Removed explicit `valueFiles` overrides (rely on convention)

**Result**: All cluster components now follow same pattern as operators.

### GAP-5: Observability components in wrong folder

**10 observability components live under `platform/observability/` but declare `domain: observability`:**

| Component | Current Path | Declared Domain |
|-----------|--------------|-----------------|
| platform-langfuse | `platform/observability/langfuse/` | observability |
| platform-loki | `platform/observability/loki/` | observability |
| platform-grafana | `platform/observability/grafana/` | observability |
| platform-grafana-datasources | `platform/observability/grafana-datasources/` | observability |
| platform-alloy-logs | `platform/observability/alloy-logs/` | observability |
| platform-tempo | `platform/observability/tempo/` | observability |
| platform-prometheus | `platform/observability/prometheus/` | observability |
| platform-otel-collector | `platform/observability/otel-collector/` | observability |
| platform-langfuse-bootstrap | `platform/observability/langfuse-bootstrap/` | observability |
| platform-observability-smoke | `platform/observability/observability-smoke/` | observability |

**Recommendation**: Move to `environments/_shared/components/observability/` to match domain.

### GAP-6: Cross-domain component placement

**3 components declare domains that don't match their folder:**

| Component | Current Path | Declared Domain | Expected |
|-----------|--------------|-----------------|----------|
| foundation-cnpg-monitoring | `foundation/operators/cnpg-monitoring/` | data | foundation |
| foundation-vault-secrets-observability | `foundation/secrets/vault-secrets-observability/` | observability | foundation |
| platform-postgres-clusters | `platform/auth/postgres-clusters/` | data | platform |

**Recommendation**: Either move components to match domain or update domain to match folder.

### GAP-7: Apps components missing valueFiles

**16 app components have no `valueFiles` declarations, but values files exist:**

Affected components in `environments/_shared/components/apps/`:
- agents, agents-runtime, gateway, graph, inference, inference-gateway
- keycloak, langfuse-route, platform, test-job-runner, threads
- transformation, workflows, workflows-runtime, www-ameide, www-ameide-platform

Corresponding values exist in `sources/values/_shared/apps/*.yaml` but are not referenced.

**Recommendation**: Add valueFiles declarations to component.yaml files or remove orphaned values.

## Backlog

### Phase 1: Immediate Fix (done in 463)
- [x] Fix helm-test.yaml kubeconform flag
- [x] Fix plausible component.yaml chart path reference
- [x] Consolidate duplicate plausible values files

### Phase 2: Rename platform-layers → platform (DONE)
- [x] **464-1**: Rename `sources/charts/platform-layers/` → `sources/charts/platform/`
- [x] **464-2**: Update all component.yaml files referencing platform-layers charts
- [x] **464-3**: Update helm-test.yaml workflow to use new path
- [x] **464-4**: Update any documentation references

### Phase 3: Create observability charts folder (DONE)
- [x] **464-5**: Create `sources/charts/observability/`
- [x] **464-6**: Move plausible to `sources/charts/observability/plausible/` (analytics = observability)
- [ ] **464-7**: Create wrapper charts for observability stack if needed (grafana, prometheus, etc.)
- [x] **464-8**: Update helm-test.yaml to include observability folder

### Phase 4: Relocate plausible (DONE)
- [x] **464-9**: Update plausible component.yaml chart path to observability
- [x] **464-10**: Move values from `apps/` to `observability/` scope
- [ ] **464-11**: Update 459 backlog RT-9 entry
- [x] **464-12**: Update component domain from `apps` to `observability`

### Phase 5: Clean up orphaned charts (GAP-4) (DONE)
- [x] **464-13**: ~~Remove `sources/charts/cluster/gateway/`~~ NOT orphaned - restored and component created
- [x] **464-14**: Remove `sources/charts/platform-layers/common/` (superseded by raw-manifests)
- [x] **464-15**: Remove `sources/charts/platform-layers/coredns-config/` (unused)
- [x] **464-16**: Remove `sources/charts/platform-layers/gitlab/` (unused)
- [x] **464-17**: Remove `sources/charts/platform-layers/namespace/` (unused)
- [x] **464-18**: Remove `sources/charts/platform-layers/registry-alias/` (unused)
- [x] **464-19**: Remove `sources/charts/platform-layers/registry-mirror/` (unused)

### Phase 5b: Clean cluster config structure (GAP-4b) (DONE)
- [x] **464-29**: Move `sources/values/cluster/coredns-config.yaml` → `_shared/cluster/`
- [x] **464-30**: Move `sources/values/cluster/cluster-gateway.yaml` → `_shared/cluster/gateway.yaml`
- [x] **464-31**: Update coredns-config component (domain: cluster, remove valueFiles)
- [x] **464-32**: Create `environments/_shared/components/cluster/configs/gateway/component.yaml`
- [x] **464-33**: Delete manually-created cluster-gateway Application

### Phase 6: Relocate observability components (GAP-5) (DONE)
- [x] **464-20**: Create `environments/_shared/components/observability/` folder structure
- [x] **464-21**: Move 12 observability components from `platform/observability/` to `observability/`
- [x] **464-22**: Update ApplicationSet to include observability path

### Phase 7: Fix cross-domain placements (GAP-6)
- [ ] **464-23**: Move `cnpg-monitoring` to `data/` or change domain to `foundation`
- [ ] **464-24**: Move `vault-secrets-observability` to `observability/` or change domain to `foundation`
- [ ] **464-25**: Move `postgres-clusters` to `data/` or change domain to `platform`

### Phase 8: Apps valueFiles alignment (GAP-7)
- [ ] **464-26**: Audit which apps values files are actually needed
- [ ] **464-27**: Add valueFiles declarations to app components that need them
- [ ] **464-28**: Remove orphaned values files that aren't referenced

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-12-07 | Created backlog | Fit-gap analysis during helm-test fix revealed structural misalignment |
| 2025-12-07 | Added GAP-4 through GAP-7 | Comprehensive codebase audit revealed additional misalignments beyond charts |
| 2025-12-07 | Added Phases 5-8 | Activities to address orphaned charts, component folder alignment, and valueFiles |
| 2025-12-07 | Completed Phases 3, 4, 6 | Observability refactoring - charts folder, plausible relocation, component moves |
| 2025-12-07 | Completed Phase 5 | Removed 6 orphaned charts (platform-layers/{common,coredns-config,gitlab,namespace,registry-alias,registry-mirror}) |
| 2025-12-07 | Completed Phase 2 | Renamed platform-layers → platform, updated 5 components, helm-test.yaml, and docs |
| 2025-12-07 | Corrected 464-13 | cluster/gateway was NOT orphaned - was missing component.yaml. Restored chart. |
| 2025-12-07 | Completed Phase 5b | Clean cluster config structure - moved values to _shared/cluster/, created gateway component, removed workarounds |
