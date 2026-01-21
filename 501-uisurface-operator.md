# 501 – UISurface Operator

**Status:** Active (Phase 1 Complete)
**Audience:** Platform engineers implementing UISurface operator
**Scope:** Implementation tracking, development phases, acceptance criteria

> **Related**:
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – Go patterns & reference implementation

---

## Grounding & cross-references

- **Architecture grounding:** Implements the UISurface primitive described in `470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, and `477-primitive-stack.md` as a first-class operator alongside Domain, Process, and Agent primitives.  
- **Operator ecosystem:** Shares CRD/status/condition conventions with `498-domain-operator.md`, `499-process-operator.md`, `500-agent-operator.md`, the shared guidance in `495-ameide-operators.md`, and the unified packaging in `503-operators-helm-chart.md`.  
- **Runtime relationships:** UISurfaces typically front Domain/Process/Agent APIs that follow the vertical slices in `502-domain-vertical-slice.md`, `499-process-operator.md`, and `504-agent-vertical-slice.md`; dependency wiring in this operator reflects those contracts.  
- **Scrum stack usage:** Any Scrum-oriented UI (e.g., backlog views or PO dashboards) that visualizes the artifacts from `300-400/367-1-scrum-transformation.md` / `508-scrum-protos.md` or the events from `506-scrum-vertical-v2.md` is hosted via this UISurface operator, even though Scrum semantics themselves are owned by those backlogs.

## 1. Overview

The UISurface operator manages the lifecycle of **UISurface primitives** – Next.js web applications serving the Ameide frontend. Each `UISurface` CR results in:

- Next.js application Deployment
- Service (ClusterIP)
- HTTPRoute (Gateway API) with host/path routing
- Keycloak client configuration for auth
- Feature flags ConfigMap

**Key insight**: The operator wires routing + auth; the UISurface image implements the UI using Ameide TS SDK.

**Canvas/layout configuration is runtime data consumed by the UISurface app; it is not encoded in CRDs and is not interpreted by the operator.**

---

## 2. CRD Summary

```yaml
apiVersion: ameide.io/v1
kind: UISurface
metadata:
  name: www-ameide-platform
spec:
  image: ghcr.io/ameide/www_ameide_platform:3.0.1
  imagePullPolicy: IfNotPresent # GitOps: keep boring; do not deploy mutable tags (see 602/603)
  host: app.ameide.com
  pathPrefix: /
  auth:
    scopes:
      - l2o:read
      - orders:write
  dependencies:
    domains:
      - platform
      - sales
    processes:
      - l2o
  features:
    flags:
      enableTransformation: true
  resources:
    cpu: "500m"
    memory: "1Gi"
status:
  conditions:
    - type: Ready
    - type: RoutingReady
    - type: AuthConfigured
    - type: DependenciesReady
  observedGeneration: 4
```

---

## 3. Reconcile Flow

```
1. Fetch UISurface CR
2. Handle deletion (finalizer cleanup – remove Keycloak client)
3. Ensure finalizer present
4. Validate spec (dependencies exist and Ready)
5. reconcileAuth()      ← Keycloak client, redirect URIs
6. reconcileRouting()   ← HTTPRoute with host/path
7. reconcileWorkload()  ← Deployment, Service, ConfigMap
8. Update status (conditions, observedGeneration)
```

---

## 4. CRD Types (Go)

```go
type UISurfaceSpec struct {
    Image        string                      `json:"image"`
    ImagePullPolicy corev1.PullPolicy        `json:"imagePullPolicy,omitempty"`
    Host         string                      `json:"host"`
    PathPrefix   string                      `json:"pathPrefix,omitempty"`
    Auth         UISurfaceAuthConfig         `json:"auth,omitempty"`
    Dependencies UISurfaceDependencies       `json:"dependencies,omitempty"`
    Features     UISurfaceFeatures           `json:"features,omitempty"`
    Resources    corev1.ResourceRequirements `json:"resources,omitempty"`
}

type UISurfaceAuthConfig struct {
    Scopes []string `json:"scopes,omitempty"`
}

type UISurfaceDependencies struct {
    Domains   []string `json:"domains,omitempty"`
    Processes []string `json:"processes,omitempty"`
    Agents    []string `json:"agents,omitempty"`
}

type UISurfaceFeatures struct {
    Flags map[string]bool `json:"flags,omitempty"`
}

type UISurfaceStatus struct {
    Conditions         []metav1.Condition `json:"conditions,omitempty"`
    ObservedGeneration int64              `json:"observedGeneration,omitempty"`
}
```

---

## 5. Development Phases

### Phase 1: Core Reconciliation ✅ IMPLEMENTED

> **Implementation**: See [operators/uisurface-operator/](../operators/uisurface-operator/)

| Task | Description | Status |
|------|-------------|--------|
| **CRD types** | Define `UISurface`, `UISurfaceSpec`, `UISurfaceStatus` | ✅ `api/v1/uisurface_types.go` |
| **Basic reconciler** | 7-step reconcile skeleton | ✅ `internal/controller/uisurface_controller.go` |
| **Finalizer handling** | Add/remove finalizer | ✅ Implemented |
| **Deployment** | Create Next.js Deployment | ✅ `internal/controller/reconcile_workload.go` |
| **Service** | Create ClusterIP Service | ✅ Implemented |
| **Status conditions** | Set `Ready` condition | ✅ `internal/controller/conditions.go` |

**Helm chart**: Operators deployable via unified chart at `operators/helm/`

### Phase 2: Routing (Gateway API)

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **HTTPRoute creation** | Create HTTPRoute with host/path | Route appears in `kubectl get httproute` |
| **Host binding** | Set `spec.hostnames` from `spec.host` | Route matches correct host |
| **Path prefix** | Set `spec.rules[].matches[].path` | Route matches correct path |
| **Backend ref** | Point to UISurface Service | Traffic routes to pods |
| **TLS handling** | Reference existing Gateway TLS | HTTPS works (cert managed externally) |
| **RoutingReady condition** | Check HTTPRoute accepted | Condition reflects route status |

### Phase 3: Auth (Keycloak Integration)

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Keycloak client creation** | Create OIDC client for UISurface | Client appears in Keycloak |
| **Redirect URIs** | Set valid redirect URIs from host/path | OAuth flow works |
| **Scopes assignment** | Assign scopes from `spec.auth.scopes` | User gets correct scopes |
| **Client secret** | Create K8s Secret with client credentials | App can authenticate |
| **AuthConfigured condition** | Check Keycloak client exists | Condition reflects auth status |
| **Client cleanup** | Delete Keycloak client on UISurface delete | No orphan clients |

### Phase 4: Dependencies & Features

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Dependency validation** | Check all referenced Domains/Processes/Agents Ready | `DependenciesReady` condition |
| **Feature flags ConfigMap** | Create ConfigMap with `spec.features.flags` | ConfigMap contains flags |
| **ConfigMap mounting** | Mount as env or file in Deployment | App reads flags at startup |
| **SDK config injection** | Set base URLs for domains/processes | Ameide TS SDK can call backends |
| **Dependency annotations** | Add dependency info to Deployment | Observability tools see deps |

### Phase 5: Production Readiness

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **HPA** | Create HPA based on CPU/memory | HPA scales pods on load |
| **Health probes** | Configure liveness/readiness for Next.js | Unhealthy pods restarted |
| **Resource defaults** | Apply default resources if not specified | Pods have resource limits |
| **ServiceMonitor** | Create ServiceMonitor for Next.js metrics | Metrics scraped |
| **Audit events** | Emit K8s events on reconcile actions | Events visible in describe |

### Phase 6: Testing & Validation

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Unit tests** | Mock Keycloak client, mock k8s client | Tests pass without real services |
| **envtest tests** | Create UISurface, verify HTTPRoute | Integration tests pass |
| **E2E tests** | Full cluster with Gateway + Keycloak | App accessible via HTTPS |
| **CLI integration** | `ameide primitive verify` checks UISurface | CLI reports UISurface health |

---

## 6. Key Implementation Details

### 6.1 HTTPRoute Template

```go
func constructHTTPRoute(surface *amv1.UISurface) *gatewayv1.HTTPRoute {
    return &gatewayv1.HTTPRoute{
        ObjectMeta: metav1.ObjectMeta{
            Name:      surface.Name,
            Namespace: surface.Namespace,
        },
        Spec: gatewayv1.HTTPRouteSpec{
            Hostnames: []gatewayv1.Hostname{gatewayv1.Hostname(surface.Spec.Host)},
            ParentRefs: []gatewayv1.ParentReference{{
                Name:      "ameide-gateway",
                Namespace: ptr.To(gatewayv1.Namespace("ameide-system")),
            }},
            Rules: []gatewayv1.HTTPRouteRule{{
                Matches: []gatewayv1.HTTPRouteMatch{{
                    Path: &gatewayv1.HTTPPathMatch{
                        Type:  ptr.To(gatewayv1.PathMatchPathPrefix),
                        Value: ptr.To(surface.Spec.PathPrefix),
                    },
                }},
                BackendRefs: []gatewayv1.HTTPBackendRef{{
                    BackendRef: gatewayv1.BackendRef{
                        BackendObjectReference: gatewayv1.BackendObjectReference{
                            Name: gatewayv1.ObjectName(serviceName(surface)),
                            Port: ptr.To(gatewayv1.PortNumber(3000)),
                        },
                    },
                }},
            }},
        },
    }
}
```

### 6.2 Keycloak Integration Options

**Option A: Keycloak Operator CRD**
```go
func (r *UISurfaceReconciler) reconcileAuth(ctx context.Context, surface *amv1.UISurface) error {
    client := &keycloakv1.KeycloakClient{
        ObjectMeta: metav1.ObjectMeta{
            Name:      surface.Name,
            Namespace: surface.Namespace,
        },
        Spec: keycloakv1.KeycloakClientSpec{
            RealmRef:     "ameide",
            ClientID:     surface.Name,
            Protocol:     "openid-connect",
            RedirectURIs: []string{fmt.Sprintf("https://%s/*", surface.Spec.Host)},
            // ...
        },
    }
    return controllerutil.SetControllerReference(surface, client, r.Scheme)
}
```

**Option B: Direct Keycloak Admin API**
```go
func (r *UISurfaceReconciler) reconcileAuth(ctx context.Context, surface *amv1.UISurface) error {
    return r.KeycloakAdmin.CreateOrUpdateClient(ctx, keycloak.ClientRepresentation{
        ClientID:     surface.Name,
        RedirectURIs: []string{fmt.Sprintf("https://%s/*", surface.Spec.Host)},
        // ...
    })
}
```

### 6.3 Reconciler Struct

```go
type UISurfaceReconciler struct {
    client.Client
    Scheme        *runtime.Scheme
    Recorder      record.EventRecorder
    KeycloakAdmin keycloak.AdminClient // or watch KeycloakClient CRD
    GatewayClass  string               // which Gateway to attach routes
}
```

### 6.4 Next.js Deployment Template

```go
func mutateNextDeployment(deploy *appsv1.Deployment, surface *amv1.UISurface, configMapName string) {
    // NOTE: avoid overwriting whole PodSpec (defaulted fields will cause endless updates).
    idx := -1
    for i := range deploy.Spec.Template.Spec.Containers {
        if deploy.Spec.Template.Spec.Containers[i].Name == "nextjs" {
            idx = i
            break
        }
    }
    if idx == -1 {
        deploy.Spec.Template.Spec.Containers = append(deploy.Spec.Template.Spec.Containers, corev1.Container{Name: "nextjs"})
        idx = len(deploy.Spec.Template.Spec.Containers) - 1
    }
    c := &deploy.Spec.Template.Spec.Containers[idx]
    c.Image = surface.Spec.Image
    c.Ports = []corev1.ContainerPort{{ContainerPort: 3000}}
    c.Env = []corev1.EnvVar{
        {Name: "NEXT_PUBLIC_API_URL", Value: "https://api.ameide.com"},
        {Name: "KEYCLOAK_CLIENT_ID", Value: surface.Name},
        // Scopes and feature flags from ConfigMap
    }
    c.EnvFrom = []corev1.EnvFromSource{{
        ConfigMapRef: &corev1.ConfigMapEnvSource{
            LocalObjectReference: corev1.LocalObjectReference{Name: configMapName},
        },
    }}
    c.LivenessProbe = &corev1.Probe{/* Next.js health check */}
    c.ReadinessProbe = &corev1.Probe{/* Next.js health check */}
    c.Resources = surface.Spec.Resources
}
```

### 6.5 Owned Resources

```go
ctrl.NewControllerManagedBy(mgr).
    For(&amv1.UISurface{}).
    Owns(&appsv1.Deployment{}).
    Owns(&corev1.Service{}).
    Owns(&corev1.ConfigMap{}).           // feature flags
    Owns(&gatewayv1.HTTPRoute{}).        // routing
    Owns(&keycloakv1.KeycloakClient{}).  // auth (if using CRD)
    Complete(r)
```

---

## 7. Non-Goals

| What | Why |
|------|-----|
| UI component logic | Lives in UISurface image |
| SSR rendering | Next.js responsibility |
| Gateway/TLS management | Separate Gateway + cert-manager |
| Keycloak realm setup | Platform bootstrap, not per-UISurface |

---

## 8. Dependencies

| Dependency | Purpose |
|------------|---------|
| **Gateway API** | HTTPRoute for ingress routing |
| **Keycloak** | OIDC authentication |
| **Domain/Process/Agent operators** | Dependencies must be Ready |
| **cert-manager** | TLS certificates (managed at Gateway level) |

---

## 9. Open Questions

| Question | Status |
|----------|--------|
| Keycloak Operator CRD vs Admin API? | TBD – depends on Keycloak deployment model |
| Multi-path UISurfaces? | Out of scope – one path prefix per UISurface |
| CSP headers injection? | TBD – could be HTTPRoute filter or app-level |

---

## 10. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [446-namespace-isolation.md](446-namespace-isolation.md) | Operator deploys once per cluster |
| [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Cluster-scoped deployment via ApplicationSet |
| [503-operators-helm-chart.md](503-operators-helm-chart.md) | Helm chart for operator deployment |
| [495-ameide-operators.md](495-ameide-operators.md) | UISurface operator responsibilities (§4) |
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Go patterns & adaptation (§10.9) |
| [477-primitive-stack.md](477-primitive-stack.md) | UISurface in primitive architecture |
| [473-ameide-technology.md](473-ameide-technology.md) | Next.js as UI technology choice |
