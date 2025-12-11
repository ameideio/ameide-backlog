> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# ArgoCD Digest-Based Deployment Requirements

**Status**: 📝 Documentation  
**Date**: 2025-08-08  
**Related**: 098-sha-latest.md, ArgoCD GitOps

## Executive Summary

ArgoCD needs specific configuration to deploy images using digests instead of tags. This document outlines the exact requirements for production-grade, immutable deployments using industry-standard digest references.

## What ArgoCD Needs

### 1. Helm Chart Support for Digests Only

**Philosophy**: Production should ONLY use immutable digests. Tags are for development only.

**Current State**: ❌ Charts use tags
```yaml
# Current (problematic)
image: "{{ .Values.image.graph }}:{{ .Values.image.tag }}"
```

**Required Change**: ✅ Use digest as the primary identifier
```yaml
# In templates/deployment.yaml
image: "{{ .Values.image.graph }}@{{ .Values.image.digest }}"
```

**Why digest-only?**
- Industry standard terminology (OCI, Docker, Kubernetes)
- No ambiguity between tag and digest
- Forces immutability in production
- Algorithm-agnostic (supports sha256, sha512, etc.)
- Prevents accidental tag usage in production

### 2. Values Files Structure

**Development** (uses Tilt, not ArgoCD):
```yaml
# values.yaml (default for development)
image:
  graph: docker.io/ameide/core-cqrs-command
  digest: ""  # Empty for development, Tilt injects its own
```

**Production** (ArgoCD managed):
```yaml
# values-production.yaml
image:
  graph: registry.example.com/core-cqrs-command
  digest: "sha256:93b60ebb48f2cb68f72a01e73ea4ee5ed3a5c8596aecf8c43ef7a7b5990f541e"
```

Note: No `tag` field at all - we only use digests for production.

### 3. GitOps Repository Structure

**Requirement**: Separate GitOps graph for deployment configurations.

**Recommended Structure**:
```
ameide-gitops/
├── environments/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   └── values/
│   │       ├── core-cqrs-command.yaml
│   │       ├── core-cqrs-query.yaml
│   │       └── ...
│   ├── staging/
│   │   └── values/
│   │       └── ... (with SHA digests)
│   └── production/
│       └── values/
│           └── ... (with SHA digests)
├── applications/
│   ├── infrastructure/
│   │   ├── redis.yaml
│   │   ├── kafka.yaml
│   │   └── postgres.yaml
│   ├── platform/
│   │   ├── core-cqrs-command.yaml
│   │   ├── core-cqrs-query.yaml
│   │   └── core-cqrs-subscription.yaml
│   └── frontend/
│       ├── www-ameide.yaml
│       └── www-ameide-portal.yaml
└── app-of-apps.yaml
```

### 4. ArgoCD Application Manifests

**Requirement**: Applications configured to use environment-specific values.

**Example Application**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: core-cqrs-command
  namespace: argocd
spec:
  project: ameide
  
  source:
    # Points to main repo with Helm charts
    repoURL: https://github.com/your-org/ameide-core
    targetRevision: main
    path: infra/kubernetes/charts/platform/core-cqrs-command
    helm:
      releaseName: core-cqrs-command
      
      # Multiple values files pattern
      valueFiles:
      - values.yaml  # Base values from chart
      
      # Additional values from GitOps repo
      valuesObject:
        image:
          graph: registry.example.com/core-cqrs-command
          # Digest injected by CI/CD
          digest: "sha256:PLACEHOLDER_WILL_BE_UPDATED_BY_CI"
  
  destination:
    server: https://kubernetes.default.svc
    namespace: ameide
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 5. CI/CD Pipeline Integration (Improved Pattern)

**Requirement**: CI/CD updates GitOps repo with SHA digests.

**Improved Pipeline with Direct Digest Resolution**:

```bash
#!/bin/bash
# ci-deploy.sh - Build, push, and update GitOps with digests

set -euo pipefail

SERVICE=$1
GIT_SHA=${GITHUB_SHA:-$(git rev-parse HEAD)}

# 1) Build OCI image with Bazel
bazel build //services/${SERVICE}:${SERVICE}_image

# 2) Push with crane (using OCI layout directory, not tarball)
IMAGE_DIR=$(bazel info bazel-bin)/services/${SERVICE}/${SERVICE}_image
crane push "${IMAGE_DIR}" \
  "registry.example.com/${SERVICE}:${GIT_SHA}" \
  --insecure  # Remove for production with proper TLS

# 3) Get the digest directly (no need to capture from push output)
DIGEST=$(crane digest "registry.example.com/${SERVICE}:${GIT_SHA}")

# 4) Update GitOps graph values
cd ../ameide-gitops
yq -i ".image.graph=\"registry.example.com/${SERVICE}\" |
       .image.digest=\"${DIGEST}\" |
       del(.image.tag)" \
  environments/production/values/${SERVICE}.yaml

# 5) Commit and push for ArgoCD sync
git add .
git commit -m "Deploy ${SERVICE}@${DIGEST:0:12} (${GIT_SHA:0:7})"
git push

echo "✅ Deployed ${SERVICE} with digest: ${DIGEST}"
```

**GitHub Actions Example**:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service:
          - core-cqrs-command
          - core-cqrs-query
          - core-cqrs-subscription
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Bazel
      uses: bazel-contrib/setup-bazel@v3
    
    - name: Setup crane
      uses: imjasonh/setup-crane@v0.3
    
    - name: Build and Push
      run: |
        # Build OCI image
        bazel build //services/${{ matrix.service }}:${{ matrix.service }}_image
        
        # Push to registry with Git SHA tag
        IMAGE_DIR=$(bazel info bazel-bin)/services/${{ matrix.service }}/${{ matrix.service }}_image
        crane push "${IMAGE_DIR}" \
          "${{ secrets.REGISTRY }}/${{ matrix.service }}:${{ github.sha }}"
    
    - name: Get Digest
      id: digest
      run: |
        DIGEST=$(crane digest "${{ secrets.REGISTRY }}/${{ matrix.service }}:${{ github.sha }}")
        echo "digest=${DIGEST}" >> $GITHUB_OUTPUT
    
    - name: Update GitOps
      uses: actions/checkout@v4
      with:
        graph: your-org/ameide-gitops
        token: ${{ secrets.GITOPS_TOKEN }}
        path: gitops
    
    - name: Update Values
      run: |
        cd gitops
        yq -i ".image.graph=\"${{ secrets.REGISTRY }}/${{ matrix.service }}\" |
               .image.digest=\"${{ steps.digest.outputs.digest }}\" |
               del(.image.tag)" \
          environments/production/values/${{ matrix.service }}.yaml
        
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "Deploy ${{ matrix.service }}@${{ steps.digest.outputs.digest }}"
        git push
```

**Key Improvements**:
1. Use `crane digest` to get digest directly (cleaner than parsing push output)
2. Delete `.image.tag` when setting `.image.digest` (avoid confusion)
3. Include both digest and Git SHA in commit message for traceability
4. Matrix build for parallel service deployments

## ArgoCD Configuration

### 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Create ArgoCD Project

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ameide
  namespace: argocd
spec:
  description: AMEIDE Platform
  
  sourceRepos:
  - 'https://github.com/your-org/ameide-core'
  - 'https://github.com/your-org/ameide-gitops'
  
  destinations:
  - namespace: 'ameide'
    server: https://kubernetes.default.svc
  - namespace: 'argocd'
    server: https://kubernetes.default.svc
  
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
```

### 3. Configure Image Updater (Optional)

ArgoCD Image Updater can automatically update image tags/digests:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd
data:
  registries.conf: |
    registries:
    - name: ameide
      prefix: registry.example.com
      api_url: https://registry.example.com
      credentials: secret:argocd/registry-creds#creds
      default: true
```

## Migration Path

### Phase 1: Update Helm Charts (Immediate)
```bash
# Update all deployment.yaml templates to support digest field
# Test with manual digest override
helm upgrade core-cqrs-command ./charts/platform/core-cqrs-command \
  --set image.digest=sha256:abc123...
```

### Phase 2: Setup GitOps Repository
```bash
# Create new graph: ameide-gitops
# Structure as shown above
# Add ArgoCD application manifests
```

### Phase 3: Configure CI/CD Pipeline
```bash
# Add GitHub Actions workflows
# Integrate crane for pushing
# Add GitOps repo update step
```

### Phase 4: Deploy ArgoCD
```bash
# Install ArgoCD in cluster
# Configure projects and applications
# Test with manual sync first
```

### Phase 5: Enable Auto-Sync
```bash
# Enable automated sync
# Monitor for successful deployments
# Verify SHA digests are being used
```

## Verification

### Check Deployed Image Digest

```bash
# Get the actual image being used
kubectl get deployment core-cqrs-command -n ameide -o json | \
  jq -r '.spec.template.spec.containers[0].image'

# Should show: registry.example.com/core-cqrs-command@sha256:...
# NOT: registry.example.com/core-cqrs-command:latest
```

### Verify ArgoCD Sync

```bash
# Check application status
argocd app get core-cqrs-command

# Check sync history
argocd app history core-cqrs-command
```

## Benefits of This Approach

1. **Immutable Deployments**: SHA digests guarantee exact image version
2. **Audit Trail**: Git history shows exactly what was deployed when
3. **Rollback Safety**: Can rollback to exact previous state
4. **Security**: Prevents tag poisoning attacks
5. **GitOps Native**: All changes tracked in Git
6. **Environment Parity**: Same image SHA across all environments

## Common Pitfalls

### ❌ Don't: Mix Tags and Digests
```yaml
# This is confusing and error-prone
image:
  graph: registry.example.com/service
  tag: v1.2.3
  digest: sha256:abc123...  # Which one wins?
```

### ❌ Don't: Update Production Manually
```bash
# Never do this in production
kubectl set image deployment/core-cqrs-command ...
```

### ✅ Do: Let ArgoCD Handle Everything
```bash
# All updates through Git
git commit -m "Update image digest"
git push
# ArgoCD auto-syncs
```

## Example: Complete ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: core-cqrs-command-production
  namespace: argocd
  annotations:
    # Image updater annotations (if using)
    argocd-image-updater.argoproj.io/image-list: |
      core-cqrs-command=registry.example.com/core-cqrs-command
    argocd-image-updater.argoproj.io/core-cqrs-command.update-strategy: digest
    argocd-image-updater.argoproj.io/core-cqrs-command.allow-tags: regexp:^main-[0-9a-f]{7}$
spec:
  project: ameide
  
  sources:
  # Chart source
  - repoURL: https://github.com/your-org/ameide-core
    targetRevision: main
    path: infra/kubernetes/charts/platform/core-cqrs-command
    helm:
      releaseName: core-cqrs-command
      valueFiles:
      - values.yaml
  
  # Values source (GitOps repo)
  - repoURL: https://github.com/your-org/ameide-gitops  
    targetRevision: main
    path: environments/production/values
    helm:
      valueFiles:
      - core-cqrs-command.yaml
  
  destination:
    server: https://kubernetes.default.svc
    namespace: ameide
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - RespectIgnoreDifferences=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  ignoreDifferences:
  # Ignore replica count changes (HPA managed)
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

## Summary

ArgoCD needs:
1. **Helm charts that support `image.digest` field**
2. **GitOps graph with environment-specific values**
3. **CI/CD pipeline that updates digests in GitOps repo**
4. **ArgoCD applications configured to use both repos**
5. **Proper sync policies and health checks**

This provides complete immutability and auditability for production deployments while keeping development velocity with Tilt.
