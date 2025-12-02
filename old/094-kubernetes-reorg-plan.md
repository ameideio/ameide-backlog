# Kubernetes Folder Reorganization Plan

## Current Structure Problems
- Mixed approaches (Helm charts, raw manifests, operators)
- Duplicate/stale files in multiple locations
- Unclear deployment strategy
- Half-migrated state

## Proposed New Structure

```
infra/kubernetes/
├── base/                      # Base cluster setup
│   ├── namespace.yaml
│   ├── registry-secret.yaml
│   └── rbac.yaml
│
├── operators/                 # Kubernetes operators (CloudNativePG, Strimzi, etc.)
│   ├── cloudnative-pg/
│   ├── strimzi-kafka/
│   ├── cert-manager/
│   └── kustomization.yaml
│
├── infrastructure/            # Infrastructure services (databases, message queues)
│   ├── postgres/             # PostgreSQL clusters
│   ├── redis/                # Redis instances
│   ├── kafka/                # Kafka clusters
│   ├── neo4j/                # Graph database
│   ├── minio/                # Object storage
│   ├── temporal/             # Workflow engine
│   └── keycloak/             # Authentication
│
├── platform/                  # AMEIDE platform services
│   ├── helm/                 # Helm chart for all services
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── values-dev.yaml
│   │   ├── values-prod.yaml
│   │   └── templates/
│   │       ├── _helpers.tpl
│   │       ├── cqrs-command.yaml
│   │       ├── cqrs-query.yaml
│   │       ├── cqrs-subscription.yaml
│   │       ├── agent-langgraph.yaml
│   │       ├── workflows-temporal.yaml
│   │       ├── provenance-tracker.yaml
│   │       └── frontend-apps.yaml
│   │
│   └── kustomize/            # Alternative: Kustomize overlays
│       ├── base/
│       └── overlays/
│           ├── dev/
│           └── prod/
│
├── scripts/                   # Deployment and management scripts
│   ├── setup-cluster.sh
│   ├── install-operators.sh
│   ├── deploy-infra.sh
│   └── deploy-platform.sh
│
└── docs/                      # Documentation
    ├── README.md
    ├── architecture.md
    └── troubleshooting.md
```

## Migration Steps

### Phase 1: Clean Up (Immediate)
1. Archive everything in `legacy/` that's not actively used
2. Remove duplicate operator definitions
3. Consolidate manifests

### Phase 2: Reorganize (Today)
1. Move operators to unified `operators/` directory
2. Separate infrastructure from application services
3. Create single platform Helm chart

### Phase 3: Simplify Deployment (Tomorrow)
1. Update Skaffold to use new structure
2. Update Bazel rules to match
3. Test end-to-end deployment

## Benefits
- Clear separation of concerns
- Single source of truth for each component
- Easier to understand and maintain
- Supports both Helm and Kustomize approaches
- Gradual migration path