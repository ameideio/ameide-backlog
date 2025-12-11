# Kubernetes Deployment Preflight Checks v2

## Executive Summary

This document outlines a refined approach to preflight checks for the AMEIDE platform's Kubernetes deployments. Based on comprehensive feedback, we focus on **runtime and infrastructure concerns** that can only be validated at deployment time, leveraging existing CNCF tooling and maintaining clear separation between CI/CD responsibilities and deployment validation.

## Guiding Principles

1. **Runtime Focus**: Only validate what cannot be checked in CI/CD
2. **Ecosystem First**: Leverage proven CNCF tools over custom implementations
3. **Environment Aware**: Support multi-cluster deployments from dev to prod
4. **Fast Feedback**: Blocking checks < 45s, async checks run in parallel
5. **Actionable Errors**: Every failure includes remediation steps

## Current State

The deployment process (`bazel run //:platform-deploy`) lacks:
- Infrastructure prerequisite validation
- Runtime health verification
- Multi-environment support
- Structured error reporting

## Architecture

### Clean Separation of Concerns

```yaml
# CI/CD Pipeline (NOT in preflight)
- Code quality (linting, formatting)
- Type checking
- Unit tests
- Static security analysis
- Build artifact creation

# Deployment Preflight (THIS proposal)
- Cluster connectivity
- Runtime dependencies
- Resource availability
- Infrastructure health
- Security policies
```

### Modular Plugin Architecture

```python
# tools/preflight/plugin_interface.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Optional, Dict

@dataclass
class CheckResult:
    name: str
    status: str  # "passed", "failed", "warning", "skipped"
    message: str
    remediation: Optional[str] = None
    details: Optional[Dict] = None
    duration_ms: int = 0

class PreflightPlugin(ABC):
    """Base interface for all preflight plugins."""
    
    @abstractmethod
    def check(self, context: Dict) -> CheckResult:
        """Execute the check with provided context."""
        pass
    
    @property
    @abstractmethod
    def timeout_seconds(self) -> int:
        """Maximum time this check should run."""
        pass
```

### Container-Based Execution

```dockerfile
# tools/preflight/Dockerfile
FROM python:3.11-slim

# Install CNCF tools
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Install kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
    && chmod +x kubectl \
    && mv kubectl /usr/local/bin/

# Install other tools
RUN curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
RUN curl -sfL https://github.com/FairwindsOps/polaris/releases/latest/download/polaris_linux_amd64.tar.gz | tar -xz -C /usr/local/bin

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . /app
WORKDIR /app

ENTRYPOINT ["python", "-m", "preflight"]
```

## Preflight Check Categories

### Phase 1: Critical Infrastructure (Blocking, <45s)

#### 1.1 Cluster Connectivity
- **Tool**: Native kubectl
- **Checks**:
  - API server reachable
  - Authentication valid
  - Target namespace exists
  - RBAC permissions sufficient
- **Implementation**: `plugins/cluster_connectivity.py`

#### 1.2 Registry Health
- **Tool**: curl/skopeo
- **Checks**:
  - Registry endpoint responsive
  - Push/pull permissions
  - Storage availability
- **Implementation**: `plugins/registry_health.py`

#### 1.3 Core Operators
- **Tool**: kubectl with CRD validation
- **Checks**:
  - Required operators running
  - CRDs installed
  - Webhook endpoints healthy
- **Implementation**: `plugins/operator_health.py`

### Phase 2: Resource Validation (Blocking, <30s)

#### 2.1 Compute Resources
- **Tool**: Kubernetes Metrics API
- **Checks**:
  - Requested vs allocatable resources
  - Node pressure conditions
  - Quota headroom
- **Implementation**: `plugins/resource_capacity.py`

#### 2.2 Storage Prerequisites
- **Tool**: kubectl + CSI validation
- **Checks**:
  - StorageClasses available
  - CSI drivers healthy
  - PVC capacity
- **Implementation**: `plugins/storage_validation.py`

### Phase 3: Security & Compliance (Async, <5m)

#### 3.1 Image Security
- **Tool**: Trivy
- **Checks**:
  - CVE scanning (configurable thresholds)
  - License compliance
  - SBOM generation
- **Implementation**: `plugins/image_security.py`

#### 3.2 Policy Compliance
- **Tool**: Open Policy Agent (OPA) / Gatekeeper
- **Checks**:
  - Security policies
  - Resource policies
  - Network policies
- **Implementation**: `plugins/policy_compliance.py`

#### 3.3 Supply Chain
- **Tool**: Cosign/Sigstore
- **Checks**:
  - Image signatures
  - Attestations
  - Provenance
- **Implementation**: `plugins/supply_chain.py`

## Implementation Plan

### Week 1: Foundation
1. Create plugin interface and orchestrator
2. Implement Phase 1 checks using existing tools
3. Container image with all dependencies
4. Integration with `bazel run //:platform-deploy`

### Week 2: Resource & Dependencies
1. Phase 2 resource validation
2. Multi-cluster context support
3. Environment-specific thresholds
4. Structured JSON/YAML reporting

### Week 3: Security & Operations
1. Async security scanning
2. Policy engine integration
3. Metrics and alerting
4. Documentation and runbooks

## Configuration

### Environment-Aware Settings

```yaml
# tools/preflight/config/environments.yaml
environments:
  dev:
    cluster_context: k3d-ameide
    resource_thresholds:
      cpu_percent: 90
      memory_percent: 90
    security:
      cve_threshold: high
      
  staging:
    cluster_context: aks-staging
    resource_thresholds:
      cpu_percent: 80
      memory_percent: 80
    security:
      cve_threshold: medium
      require_signatures: true
      
  production:
    cluster_context: aks-prod
    resource_thresholds:
      cpu_percent: 70
      memory_percent: 70
    security:
      cve_threshold: low
      require_signatures: true
      require_attestations: true
```

### Override Governance

```python
# Auditable override mechanism
def run_with_override(check_name: str, ticket: str, reason: str):
    """
    Skip a check with proper audit trail.
    
    Usage:
    preflight --override cluster_connectivity \
              --ticket JIRA-123 \
              --reason "Known issue with dev cluster"
    """
    audit_log.warning(
        f"Check '{check_name}' overridden",
        extra={
            "ticket": ticket,
            "reason": reason,
            "user": os.getenv("USER"),
            "timestamp": datetime.utcnow().isoformat()
        }
    )
```

## Success Metrics

### Quantitative
- **Blocking checks runtime**: p95 < 45 seconds
- **Full suite runtime**: p95 < 5 minutes  
- **Deployment success rate**: 10% improvement from baseline
- **MTTD (Mean Time to Detect)**: < 30 seconds
- **MTTR (Mean Time to Remediate)**: 50% reduction

### Qualitative
- **Actionable errors**: 100% include remediation steps
- **Plugin addition complexity**: < 50 LoC for new check
- **Override usage**: < 5% of deployments

## Integration Points

### CI/CD Pipeline
```yaml
# .github/workflows/deploy.yaml
- name: Preflight Checks
  uses: docker://ameide/preflight:latest
  with:
    environment: ${{ github.event.inputs.environment }}
    fail_fast: true
    
- name: Deploy
  if: steps.preflight.outcome == 'success'
  run: bazel run //:deploy
```

### GitOps Integration
```yaml
# Kubernetes CRD for Argo CD
apiVersion: preflight.ameide.io/v1
kind: PreflightCheck
metadata:
  name: production-deploy
spec:
  environment: production
  checks:
    - cluster-connectivity
    - resource-capacity
    - security-scan
status:
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2024-01-15T10:00:00Z"
```

## Testing Strategy

### Unit Tests
```python
# tests/test_cluster_connectivity.py
def test_cluster_unreachable():
    """Verify check fails gracefully when cluster is down."""
    plugin = ClusterConnectivityPlugin()
    context = {"kubeconfig": "/tmp/invalid"}
    
    result = plugin.check(context)
    
    assert result.status == "failed"
    assert "Cannot connect to" in result.message
    assert "kubectl cluster-info" in result.remediation
```

### Integration Tests
```yaml
# tests/fixtures/bad-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraKubeadmConfig:
      apiServer:
        extraArgs:
          enable-admission-plugins: "AlwaysDeny"
```

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|---------|------------|
| Tool version drift | False positives/negatives | Pin versions in container image, automated updates |
| Excessive permissions | Security exposure | Read-only ServiceAccount, no cluster-admin |
| Secret leakage | Credential exposure | Output sanitization, no env var echoing |
| Network dependencies | Flaky checks | Retry logic, offline mode for air-gapped |
| Performance regression | Slow deployments | Parallel execution, caching, timeout enforcement |

## Maintenance & Evolution

### Documentation
- Auto-generated from plugin docstrings
- Runbook links in error messages
- Architectural decision records (ADRs)

### Observability
```python
# Metrics exposed
preflight_check_duration_seconds{check="cluster_connectivity", status="passed"} 2.3
preflight_check_total{check="resource_capacity", status="failed"} 1
preflight_override_total{check="security_scan", ticket="JIRA-123"} 1
```

### Plugin Lifecycle
1. Propose via RFC
2. Implement with tests
3. Beta flag for 2 weeks
4. Graduate to stable
5. Deprecation requires 1 month notice

## Conclusion

This refined approach focuses preflight checks on their core value: validating runtime and infrastructure concerns that cannot be caught earlier. By leveraging proven CNCF tools, maintaining clean architectural boundaries, and providing clear escape hatches, we can improve deployment reliability without adding friction to the developer experience.