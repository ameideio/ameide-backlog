# Bazel + Skaffold Architecture Decision

## Summary

Adopt Skaffold as the orchestration tool while keeping Bazel strictly for builds. This creates a clean separation: **"Bazel builds immutable artifacts; Skaffold delivers them to Kubernetes."**

## Current State

The graph has deep Bazel integration across all layers:
- 30+ orchestration targets in root BUILD.bazel (deploy, logs, shell, etc.)
- Custom shell script library wrapped in Bazel targets
- Everything flows through `bazel run` commands
- Initial Skaffold integration exists but minimal (just //:dev target)

## Problem Statement

1. **Mixed Concerns**: BUILD files contain both build rules and orchestration logic
2. **Non-Standard Workflow**: `bazel run //:logs` instead of standard `kubectl logs`
3. **Maintenance Burden**: 31 custom shell scripts wrapped in Bazel
4. **Limited Features**: Missing modern dev features like hot reload, debugging
5. **Learning Curve**: New developers must learn custom Bazel orchestration
6. **Ownership Confusion**: Unclear boundaries between build and deploy responsibilities

## Proposed Solution

### Rule of Three: Clear Ownership Boundaries

| Concern | Owner | Litmus Test |
|---------|-------|-------------|
| **Build + cache** | Bazel only | No shell script under `skaffold/` executes a compiler or unit test |
| **Tag, push, deploy, watch, debug** | Skaffold only | No `*_push` or `*_deploy` targets remain in any `BUILD.bazel` |
| **Cluster plumbing** | One shell wrapper outside both | Contributors can recreate dev cluster with one command that mentions neither Bazel nor Skaffold |

Each grey zone doubles the mental surface area - enforce clean boundaries.

## Migration Strategy

### Big Bang Migration - No Backward Compatibility

Complete cutover to Skaffold with no transition period. All orchestration moves to Skaffold immediately:

**Target State Only:**
```bash
# Developer workflows
skaffold dev                    # Development with watch
skaffold run                    # One-time deployment
skaffold delete                 # Teardown
kubectl logs/exec/port-forward  # Direct kubectl for debugging
```

No parallel systems, no gradual adoption. Clean break from Bazel orchestration.

### Preserve Hermeticity Where Valuable

Integration tests can still run hermetically under Bazel **inside** a Skaffold-managed cluster:

```python
sh_test(
    name = "e2e_payment",
    size = "large",
    srcs = ["tests/e2e_payment.sh"],
    data = ["//deployment:kubeconfig"],  # written by pre-test hook
)
```

Key: treat "cluster up" as test fixture, not part of Bazel graph.

## Critical Decisions Before Migration

### 1. Version Locking
- `bazelisk --bazel_version=8.x.x` (or `.bazelversion`)
- `rules_oci = "2.2.7"`, `rules_python = "1.9.0"` 
- `skaffold: v2.10.x` in `skaffold.yaml`
- Pin Helm, k3d/kind, kubectl versions

### 2. Tag vs Digest Strategy
- **Tags only (latest, git-SHA)**: Simple but race conditions in multi-branch CI
- **Digest only**: Audit-friendly but requires Helm chart changes
- **Tag + digest** (via `remote_tags=["latest"]`): Best of both with GC policy

Document choice in `CONTRIBUTING.md` to prevent re-litigation.

### 3. Automation Up Front

| Pain Point | Solution |
|------------|----------|
| Divergent env vars | Shared `.env` file that both tools honor |
| Script sprawl | Standardize on `./scripts/xxx.sh` executed by both |
| CI confusion | Golden commands: `ci/local.sh` and `ci/pipeline.sh` |

## Implementation Plan

### Week 1: Preparation
- [ ] Create `.bazelversion` and lock all rule versions
- [ ] Document tag/digest decision in CONTRIBUTING.md
- [ ] Create comprehensive skaffold.yaml with all services
- [ ] Port critical scripts (preflight, migrations) to skaffold hooks
- [ ] Update CI/CD pipeline configuration

### Week 2: Cutover
- [ ] Remove ALL orchestration targets from BUILD.bazel files
- [ ] Delete/archive all Bazel wrapper scripts
- [ ] Update all documentation to Skaffold commands
- [ ] Team training on new workflows
- [ ] Fix any issues discovered during cutover

### Target End State

```yaml
# BUILD.bazel files contain ONLY:
py_binary(...)      # Build artifacts
py_test(...)        # Unit tests  
oci_image(...)      # Container images

# skaffold.yaml contains ALL:
build:              # How to build images
deploy:             # How to deploy to k8s
portForward:        # Dev environment setup
profiles:           # dev/staging/prod
```

## Success Metrics

1. **Developer Velocity**: Time from code change to running in cluster < 30s
2. **Onboarding Time**: New developer to first deployment < 1 hour
3. **Build Cache Hit Rate**: >90% for unchanged code
4. **Lines of Code**: 50% reduction in orchestration scripts

## What We Gain

1. **Industry Standard Tooling**: Every k8s developer knows Skaffold
2. **Better Developer Experience**: Hot reload, file sync, debugging
3. **Cleaner Architecture**: No more mixed concerns in BUILD files
4. **Reduced Maintenance**: No custom orchestration to maintain

## What We Lose

1. **Unified Interface**: No more "bazel run" for everything
2. **Custom Features**: Preflight checks need reimplementation
3. **Hermetic Deploy**: Skaffold relies on ambient environment
4. **Sunk Cost**: 30+ Bazel targets thrown away

## Recommendation

**Proceed with big bang migration**. The benefits of clean separation and industry-standard tooling outweigh the migration cost. A clean break prevents the complexity of maintaining two systems.

Schedule the cutover during a low-activity period (e.g., start of sprint) to minimize disruption.

## References

- [Skaffold Documentation](https://skaffold.dev/docs/)
- [Current Bazel orchestration](../BUILD.bazel)
- [OCI Image Building Rules](../build/bazel/rules/ameide_oci_image_build_python.bzl)