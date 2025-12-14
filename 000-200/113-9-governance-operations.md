# Artifact-Centric Graph Architecture - Governance & Operations

## Kind Registry & Governance

### Canonical Form
All kinds follow strict canonical form:
- **Format**: `namespace::kind` (lowercase)
- **Namespaces**: `common`, `structural`, `doc`, `view`, `archimate`, `bpmn`
- **Single source of truth**: `packages/ameide_core_proto/kinds_registry.yaml`

### Registry Management
```yaml
# kinds_registry.yaml
version: 1.0.0
kinds:
  common::folder:
    status: stable
    since: 2024-01-01
    description: Container for organizing artifacts
    
  bpmn::sequenceflow:  # Always lowercase in registry
    status: stable
    since: 2024-01-01
    description: BPMN sequence flow between nodes
    
  archimate::capability:
    status: beta
    since: 2024-06-01
    deprecates: archimate::business_capability  # Old name
```

### Governance Process
1. **Propose**: Submit PR to `kinds_registry.yaml`
2. **Review**: Architecture team approval required
3. **Deprecation**: 90-day warning period with migration guide
4. **Removal**: Only after all repositories migrated

### CI Enforcement
```bash
# CI check - fail if unregistered kind used
grep -r "kind:" . --include="*.proto" --include="*.py" | \
  while read line; do
    kind=$(echo $line | sed 's/.*kind: *"\([^"]*\)".*/\1/')
    if ! grep -q "$kind" kinds_registry.yaml; then
      echo "ERROR: Unregistered kind: $kind"
      exit 1
    fi
  done
```

## Event Schema Versioning

### Versioning Strategy
```python
# Event envelope with version
class EventEnvelope:
    # For Protobuf contracts, breaking changes use a new major package/topic family (`...v2`)
    # and are guarded by `buf breaking`; this numeric major is only for consumer-side upcasting.
    schema_major: int = 1

    # Optional semantic marker (string, e.g. "1.0.0"); not used as the compatibility gate.
    schema_version: str = "1.0.0"

    message_type: str
    payload: dict
    
    def upcast(self) -> dict:
        """Upgrade old events to current schema."""
        if self.schema_major == 1 and self.message_type == "ArtifactCreated":
            # v1 -> v2: Add graph_id if missing
            if 'graph_id' not in self.payload:
                self.payload['graph_id'] = extract_from_stream(self.stream_id)
        return self.payload
```

### Upcaster Chain
```python
class ProjectorWithUpcasting:
    def __init__(self):
        self.upcasters = {
            1: UpcastV1ToV2(),
            2: UpcastV2ToV3(),
        }
    
    async def process_event(self, event: dict):
        # Apply upcasters in sequence
        current_major = event.get('schema_major', 1)
        while current_major < CURRENT_SCHEMA_MAJOR:
            event = self.upcasters[current_major].upcast(event)
            current_major += 1
        
        # Process with current schema
        await self.apply_projection(event)
```

### Contract Tests
```python
async def test_rebuild_across_versions():
    """Ensure rebuild succeeds with mixed event versions."""
    # Insert v1 events
    await insert_v1_events()
    # Insert v2 events
    await insert_v2_events()
    # Rebuild should handle both
    await graphctl_rebuild()
    assert await count_artifacts() == expected_count
```

## Delete & Tombstone Semantics

### Deletion Policy
```cypher
-- Soft delete (default)
MATCH (a:Artifact {id: $id})
SET a.deleted = true,
    a.deletedAt = datetime(),
    a.deletedBy = $userId

-- All queries must exclude soft-deleted
MATCH (a:Artifact)
WHERE coalesce(a.deleted, false) = false
RETURN a

-- Hard delete after retention period
MATCH (a:Artifact)
WHERE a.deleted = true 
  AND a.deletedAt < datetime() - duration('P90D')
DETACH DELETE a
```

### Edge Deletion
```cypher
-- Soft delete edge
MATCH ()-[r:RELATES {edge_key: $edgeKey}]-()
SET r.deleted = true,
    r.deletedAt = datetime()

-- Hard delete edge
MATCH ()-[r:RELATES {edge_key: $edgeKey}]-()
WHERE r.deleted = true
DELETE r
```

### Reified Relation Deletion
```cypher
-- Soft delete reified relation
MATCH (rel:Relation {id: $relationId})
SET rel.deleted = true

-- Remove edges on hard delete
MATCH (rel:Relation {id: $relationId})
WHERE rel.deleted = true
MATCH (rel)-[e]-()
DELETE e, rel
```

## Query Guardrails & Quotas

### Default Limits
```python
QUERY_DEFAULTS = {
    'maxDepth': 5,           # Maximum traversal depth
    'maxNodes': 50_000,      # Maximum nodes returned
    'maxEdges': 150_000,     # Maximum edges traversed
    'queryTimeout': 3000,    # Milliseconds
    'maxResultRows': 10_000, # Maximum result size
}

RATE_LIMITS = {
    'requests_per_second': 100,
    'burst_size': 200,
    'per_graph_rps': 10,
}
```

### Enforcement
```python
async def execute_query(query: str, params: dict, limits: dict = None):
    limits = {**QUERY_DEFAULTS, **(limits or {})}
    
    # Add limits to Cypher
    query = f"""
    CALL apoc.cypher.runTimeboxed(
      '{query}',
      {params},
      {limits['queryTimeout']}
    ) YIELD value
    RETURN value
    LIMIT {limits['maxResultRows']}
    """
    
    # Check result size
    result = await neo4j.run(query)
    if len(result) >= limits['maxResultRows']:
        raise QueryLimitExceeded(413, "Result too large")
    
    return result
```

### Rate Limiting
```python
from asyncio_throttle import Throttler

class RateLimitedAPI:
    def __init__(self):
        self.throttler = Throttler(
            rate_limit=100,  # requests per second
            period=1.0,
            burst_size=200
        )
    
    async def handle_request(self, request):
        async with self.throttler:
            # Check per-graph limit
            repo_key = f"{request.tenant_id}:{request.graph_id}"
            if await check_repo_limit(repo_key):
                return Response(429, headers={'Retry-After': '1'})
            
            return await process_request(request)
```

## Access Control Mapping

### Keycloak to Repository Roles
```python
# Role mapping
ROLE_MAPPING = {
    'graph.viewer': ['read'],
    'graph.editor': ['read', 'write'],
    'graph.owner': ['read', 'write', 'delete', 'admin'],
}

def extract_permissions(jwt_claims: dict) -> dict:
    """Map Keycloak claims to graph permissions."""
    permissions = {}
    
    # Extract graph roles from resource_access
    for repo_id, roles in jwt_claims.get('resource_access', {}).items():
        if repo_id.startswith('repo-'):
            permissions[repo_id] = []
            for role in roles.get('roles', []):
                permissions[repo_id].extend(ROLE_MAPPING.get(role, []))
    
    return permissions
```

### Permission Cache
```python
from cachetools import TTLCache

class AuthzCache:
    def __init__(self):
        self.cache = TTLCache(maxsize=10000, ttl=60)  # 60s TTL
    
    async def check_permission(self, user_id: str, repo_id: str, action: str):
        cache_key = f"{user_id}:{repo_id}"
        
        # Check cache
        if cache_key in self.cache:
            return action in self.cache[cache_key]
        
        # Fetch from Keycloak
        permissions = await fetch_user_permissions(user_id, repo_id)
        self.cache[cache_key] = permissions
        
        return action in permissions
```

## Data Lifecycle & Compliance

### GDPR/PII Handling
```python
# PII fields encryption
PII_FIELDS = ['email', 'phone', 'ssn', 'address']

class PIIEncryptor:
    def encrypt_event(self, event: dict) -> dict:
        for field in PII_FIELDS:
            if field in event:
                event[field] = self.encrypt(event[field])
                event[f"{field}_encrypted"] = True
        return event
    
    def decrypt_for_authorized(self, event: dict, user_permissions: list) -> dict:
        if 'pii_read' not in user_permissions:
            return event
        
        for field in PII_FIELDS:
            if f"{field}_encrypted" in event:
                event[field] = self.decrypt(event[field])
        return event
```

### Right to be Forgotten
```python
async def handle_erasure_request(user_id: str):
    """GDPR Article 17 - Right to erasure."""
    
    # 1. Write compensating events
    await event_store.append(
        stream_id=f"user-{user_id}",
        event=UserErasureRequested(user_id=user_id)
    )
    
    # 2. Anonymize in projections
    await neo4j.run("""
        MATCH (a:Artifact {createdBy: $userId})
        SET a.createdBy = 'ANONYMIZED',
            a.pii_erased = true
    """, {'userId': user_id})
    
    # 3. Schedule hard delete after retention
    await schedule_hard_delete(user_id, days=30)
```

### Retention Schedules
```sql
-- Retention policy table
CREATE TABLE retention_policies (
    artifact_kind TEXT PRIMARY KEY,
    retention_days INTEGER NOT NULL,
    hard_delete BOOLEAN DEFAULT false
);

INSERT INTO retention_policies VALUES
    ('common::log', 90, true),
    ('doc::draft', 180, false),
    ('archimate::model', 2555, false);  -- 7 years

-- Cleanup job
DELETE FROM artifacts
WHERE deleted = true
  AND deletedAt < NOW() - INTERVAL '1 day' * (
    SELECT retention_days FROM retention_policies
    WHERE artifact_kind = artifacts.kind
  );
```

## Operational Limits

### Repository Quotas
```python
REPOSITORY_LIMITS = {
    'max_artifacts': 5_000_000,
    'max_edges_per_node': 50_000,
    'max_blob_size_mb': 25,
    'max_total_storage_gb': 100,
    'warning_threshold': 0.8,  # Warn at 80%
}

async def check_quotas(repo_id: str):
    """Check graph against quotas."""
    stats = await get_repo_stats(repo_id)
    
    for metric, limit in REPOSITORY_LIMITS.items():
        current = stats.get(metric, 0)
        if current > limit:
            raise QuotaExceeded(f"{metric} exceeded: {current}/{limit}")
        
        if current > limit * REPOSITORY_LIMITS['warning_threshold']:
            await notify_admins(f"Repository {repo_id} approaching {metric} limit")
```

## Failure Modes & Recovery

### Retry Strategy
```python
from tenacity import retry, stop_after_attempt, wait_exponential

class ResilientProjector:
    @retry(
        stop=stop_after_attempt(7),
        wait=wait_exponential(multiplier=0.1, max=2)  # 100ms -> 2s
    )
    async def process_with_retry(self, event):
        try:
            await self.apply_projection(event)
        except Neo4jConnectionError:
            # Circuit breaker
            if self.failure_count > 10:
                await self.pause_partition()
                raise CircuitBreakerOpen()
            self.failure_count += 1
            raise
```

### Backpressure Handling
```python
async def handle_backpressure():
    """Auto-scale based on lag."""
    lag = await get_consumer_lag()
    
    if lag > 10000:  # 10k events behind
        await scale_consumers(min(current * 2, MAX_CONSUMERS))
    elif lag < 100 and current > MIN_CONSUMERS:
        await scale_consumers(max(current // 2, MIN_CONSUMERS))
```

## Snapshots & Rebuild Optimization

### Aggregate Snapshots
```python
async def create_snapshot(stream_id: str):
    """Snapshot aggregate state periodically."""
    events = await event_store.get_events(stream_id)
    
    if len(events) > SNAPSHOT_THRESHOLD:
        state = rebuild_aggregate(events)
        await snapshot_store.save(
            stream_id=stream_id,
            version=events[-1].version,
            state=state
        )
```

### Graph Snapshots
```bash
# Nightly graph snapshot for fast rebuilds
neo4j-admin backup \
  --database=neo4j \
  --backup-dir=/backups/$(date +%Y%m%d) \
  --incremental
```

## Feature Flag Management

### Per-Repository Toggles
```sql
CREATE TABLE graph_features (
    graph_id TEXT,
    feature_name TEXT,
    enabled BOOLEAN,
    config JSONB,
    PRIMARY KEY (graph_id, feature_name)
);

-- Override precedence: repo > tenant > global
WITH feature_config AS (
    SELECT enabled, 1 as priority FROM graph_features 
    WHERE graph_id = $1 AND feature_name = $2
    UNION
    SELECT enabled, 2 FROM tenant_features
    WHERE tenant_id = $3 AND feature_name = $2
    UNION
    SELECT enabled, 3 FROM global_features
    WHERE feature_name = $2
)
SELECT enabled FROM feature_config
ORDER BY priority
LIMIT 1;
```

## Projector Upgrade Safety

### Blue-Green Deployment
```yaml
# Deploy new version alongside old
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graph-projector-v2
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: projector
        image: graph-projector:v2
        env:
        - name: CONSUMER_GROUP
          value: graph-projection-v2  # Different group
        - name: MODE
          value: shadow  # Don't commit offsets
```

### Query Template Versioning
```python
QUERY_TEMPLATES = {
    'artifact_upsert_v1': """...""",
    'artifact_upsert_v2': """...""",  # New version
}

def get_query_template(template_id: str, version: int):
    key = f"{template_id}_v{version}"
    return QUERY_TEMPLATES[key]
```
