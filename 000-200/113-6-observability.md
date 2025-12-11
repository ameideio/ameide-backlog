# Artifact-Centric Graph Architecture - Observability

## Distributed Tracing with Tempo

```python
from opentelemetry import trace, propagate
from opentelemetry.trace import SpanKind, Link

tracer = trace.get_tracer(__name__)

class GraphProjector:
    async def process_event(self, record: ConsumerRecord):
        # Extract trace context from Kafka headers
        ctx = propagate.extract(carrier=dict(record.headers))
        
        # Create span with link to original command span
        links = [Link(ctx)] if ctx else []
        
        with tracer.start_as_current_span(
            "graph.projection.apply",
            kind=SpanKind.CONSUMER,
            links=links,
            attributes={
                "messaging.system": "kafka",
                "messaging.destination": record.topic,
                "graph.id": record.key,
                "event.type": record.value.get("type"),
                "event.seq": record.value.get("seq")
            }
        ) as span:
            # Process projection
            with tracer.start_as_current_span("neo4j.merge", attributes={
                "db.system": "neo4j",
                "graph.query_template_id": "artifact_upsert"
            }):
                await self.apply_projection(record.key, record.value)
```

## Metrics for Grafana

```python
from prometheus_client import Counter, Histogram, Gauge

# Low-cardinality metrics with exemplars
projection_events = Counter('projection_events_total',
    'Events applied to graph', ['service', 'env'])

projection_latency = Histogram('projection_apply_latency_seconds',
    'Time to apply projection', ['query_template_id'])

projection_lag = Gauge('projection_lag_seconds', 
    'Lag between event and projection')

# Top-N graph lag without cardinality explosion
projection_lag_top = Gauge('projection_lag_top_seconds',
    'Top 10 graph lags', ['rank'])

# ChangeSet-specific metrics
changeset_created = Counter('changeset_created_total',
    'ChangeSets created', ['graph_id', 'ai_generated'])

changeset_by_status = Gauge('changeset_by_status',
    'Current changesets by status', ['graph_id', 'status'])

changeset_approval_time = Histogram('changeset_approval_seconds',
    'Time from submission to approval', ['graph_id'],
    buckets=(60, 300, 900, 1800, 3600, 7200, 14400))  # 1m, 5m, 15m, 30m, 1h, 2h, 4h

changeset_conflict_rate = Counter('changeset_conflicts_total',
    'ChangeSets with conflicts', ['graph_id', 'conflict_type'])

changeset_rebase_success = Counter('changeset_rebase_success_total',
    'Successful rebases', ['graph_id', 'auto_rebase'])

changeset_batch_size = Histogram('changeset_batch_size',
    'Number of streams in batch commit', ['graph_id'],
    buckets=(1, 2, 5, 10, 20, 50, 100))

# Example usage
async def record_metrics(graph_id: str, event: dict, duration: float):
    projection_events.labels(service='graph-projector', env='prod').inc()
    projection_latency.labels(query_template_id='artifact_upsert').observe(duration)
    
    # Update top-N lag metrics
    lag = time.time() - event['timestamp']
    await update_top_n_lag(graph_id, lag)

async def record_changeset_metrics(changeset: ChangeSet, operation: str):
    if operation == 'created':
        changeset_created.labels(
            graph_id=changeset.graph_id,
            ai_generated=str(changeset.ai_generated)
        ).inc()
    elif operation == 'approved':
        approval_time = (changeset.approved_at - changeset.submitted_at).total_seconds()
        changeset_approval_time.labels(graph_id=changeset.graph_id).observe(approval_time)
    elif operation == 'conflicted':
        changeset_conflict_rate.labels(
            graph_id=changeset.graph_id,
            conflict_type='version_mismatch'
        ).inc()
    elif operation == 'rebased':
        changeset_rebase_success.labels(
            graph_id=changeset.graph_id,
            auto_rebase=str(changeset.auto_rebased)
        ).inc()
```

## Structured Logs to Loki

```python
import structlog

logger = structlog.get_logger()

# JSON logs with trace correlation
logger.info("projection_applied",
    trace_id=span.get_span_context().trace_id,
    tenant_id=tenant_id,
    graph_id=graph_id,
    event_seq=event["seq"],
    event_type=event["type"],
    query_template_id="artifact_upsert",
    rows_affected=1
)
```

Example log entry:
```json
{
  "timestamp": "2024-01-20T10:30:45.123Z",
  "level": "info",
  "msg": "projection_applied",
  "trace_id": "7b3a4e2f9c1d5a8b",
  "tenant_id": "tenant-123",
  "graph_id": "repo-456",
  "event.seq": 8421,
  "event.type": "ArtifactCreated",
  "query_template_id": "artifact_upsert",
  "rows_affected": 1
}
```

## Grafana Dashboards

### Projection Overview
- Event processing rate (events/sec)
- Projection lag by graph (p50, p95, p99)
- Error rate by event type
- DLQ accumulation
- Consumer group lag

### Neo4j Health
- Cypher latency by query_template_id
- Connection pool utilization
- Error rates by error code
- Cache hit ratio
- Active transactions

### ChangeSet Dashboard
```yaml
panels:
  - title: ChangeSet Queue Depth
    query: sum by (status) (changeset_by_status)
    visualization: stat
    
  - title: Approval Latency (p95)
    query: histogram_quantile(0.95, rate(changeset_approval_seconds_bucket[5m]))
    unit: seconds
    
  - title: AI vs Human Proposals
    query: sum by (ai_generated) (rate(changeset_created_total[1h]))
    visualization: pie
    
  - title: Conflict Rate
    query: rate(changeset_conflicts_total[5m])
    visualization: timeseries
    
  - title: Rebase Success Rate
    query: |
      sum(rate(changeset_rebase_success_total[1h])) /
      sum(rate(changeset_conflicts_total[1h]))
    unit: percentunit
    
  - title: Batch Commit Size Distribution
    query: histogram_quantile(0.5, rate(changeset_batch_size_bucket[5m]))
    visualization: heatmap
    
  - title: Review Queue Age
    query: time() - min by (graph_id) (changeset_created_timestamp)
    unit: seconds
```

### SLO Board
- Projection lag SLO (p95 ≤ 5s) - current vs target
- Query latency SLO (p95 ≤ 250ms) - burn rate
- Availability (99.9%) - error budget remaining
- Throughput (2k events/sec) - capacity utilization
- ChangeSet approval SLO (p95 ≤ 1h) - current performance

## Alerts with Runbooks

```yaml
alerts:
  - name: projection_lag_high
    expr: histogram_quantile(0.95, projection_lag_seconds) > 10
    for: 5m
    severity: warning
    runbook: |
      1. Check Kafka consumer lag:
         kubectl exec -n ameide deployment/graph-projector -- kafka-consumer-groups.sh --describe
      2. If high: scale consumers
         kubectl scale deployment/graph-projector --replicas=3
      3. If normal: check Neo4j latency
         MATCH (n) RETURN count(n) // warm cache
      4. Inspect traces with high neo4j.merge duration
  
  - name: dlq_spike
    expr: rate(dlq_events_total[5m]) > 10
    for: 5m
    severity: warning
    runbook: |
      1. Sample DLQ messages:
         kafka-console-consumer --topic artifacts.events.dlq --max-messages 5
      2. Check for schema issues or missing entities
      3. Fix event shape or projector logic
      4. Replay from DLQ once patched:
         graphctl replay-dlq --topic artifacts.events.dlq
  
  - name: neo4j_latency_high
    expr: histogram_quantile(0.95, neo4j_query_duration_seconds) > 1
    for: 10m
    severity: critical
    runbook: |
      1. Check query patterns:
         CALL dbms.listQueries() YIELD query, elapsedTimeMillis WHERE elapsedTimeMillis > 1000
      2. Review indexes:
         SHOW INDEXES
      3. Check for missing typed edges on hot paths
      4. Consider query optimization or caching

  - name: changeset_queue_depth_high
    expr: sum(changeset_by_status{status="pending_review"}) > 50
    for: 30m
    severity: warning
    runbook: |
      1. Check reviewer availability:
         SELECT user_id, COUNT(*) FROM changeset_reviews GROUP BY user_id
      2. Identify bottlenecks:
         - Large changesets blocking queue
         - AI-generated changesets needing extra review
      3. Consider auto-approval for low-risk changes
      4. Alert reviewers via Slack/email

  - name: changeset_approval_slo_breach
    expr: histogram_quantile(0.95, changeset_approval_seconds) > 3600
    for: 15m
    severity: warning
    runbook: |
      1. Check for stuck changesets:
         SELECT * FROM changesets WHERE status='pending_review' 
         AND created_at < NOW() - INTERVAL '1 hour'
      2. Look for conflict resolution delays
      3. Review auto-rebase failures
      4. Consider adjusting SLO or adding reviewers

  - name: changeset_conflict_spike
    expr: rate(changeset_conflicts_total[5m]) > 0.1
    for: 10m
    severity: info
    runbook: |
      1. Identify conflicting streams:
         SELECT stream_id, COUNT(*) FROM conflicts GROUP BY stream_id
      2. Check for:
         - Concurrent editors on same artifacts
         - Stale changesets needing rebase
         - Race conditions in batch commits
      3. Review optimistic locking configuration
      4. Consider advisory locks for hot artifacts
```

## Cardinality Guards

**Metrics labels** - Only use:
- `service` (graph-projector, neo4j-client)
- `env` (dev, staging, prod)
- `query_template_id` (artifact_upsert, relation_create)

**Never use as labels:**
- tenant_id
- graph_id
- artifact_id
- user_id

**Loki labels** - Keep minimal:
```yaml
{service="graph-projector", env="prod", level="info"}
```

Dynamic values as fields, not labels.

## Kill Switch & Operational Controls

```python
# Feature flags
GRAPH_READS_ENABLED = os.getenv("GRAPH_READS_ENABLED", "true") == "true"
ENABLE_TYPED_EDGES = os.getenv("ENABLE_TYPED_EDGES", "true") == "true"
PROJECTION_BATCH_SIZE = int(os.getenv("PROJECTION_BATCH_SIZE", "100"))

# Kill switch implementation
async def handle_query(request):
    if not GRAPH_READS_ENABLED:
        return Response(status=501, body="Graph reads temporarily disabled")
    
    # Normal query processing
    return await execute_graph_query(request)
```

## Rebuild Command

```bash
#!/bin/bash
# graphctl rebuild script

graphctl rebuild --tenant <tenant-id> --graph <repo-id>

# Implementation
async def rebuild_graph(tenant_id: str, graph_id: str):
    """Rebuild graph from event store."""
    
    # 1. Mark graph as rebuilding
    await set_rebuild_flag(tenant_id, graph_id, True)
    
    # 2. Clear existing graph data
    await neo4j.run("""
        MATCH (r:Repository {id: $repoId})-[:OWNS]->(n)
        DETACH DELETE n
    """, {'repoId': graph_id})
    
    # 3. Reset projection state
    await reset_projection_state(tenant_id, graph_id)
    
    # 4. Replay all events
    events = await event_store.get_all_events(
        f"graph-{graph_id}-*"
    )
    
    for batch in chunk(events, 1000):
        await projector.apply_batch(batch)
    
    # 5. Clear rebuild flag
    await set_rebuild_flag(tenant_id, graph_id, False)
    
    logger.info("rebuild_complete",
        tenant_id=tenant_id,
        graph_id=graph_id,
        events_processed=len(events)
    )
```

## Dark Reads for Validation

```python
async def validate_graph_consistency():
    """Compare graph results with document store."""
    
    sample_queries = [
        ("get_artifact", {"id": "test-artifact-1"}),
        ("find_by_kind", {"kind": "bpmn::Process"}),
        ("trace_impact", {"id": "test-artifact-2", "depth": 2})
    ]
    
    for query_type, params in sample_queries:
        # Execute in both stores
        graph_result = await graph_store.execute(query_type, params)
        doc_result = await document_store.execute(query_type, params)
        
        # Compare results
        if not results_match(graph_result, doc_result):
            logger.warning("consistency_mismatch",
                query_type=query_type,
                params=params,
                graph_count=len(graph_result),
                doc_count=len(doc_result)
            )
            
            # Record metric
            consistency_mismatches.labels(query_type=query_type).inc()
```

## Performance Monitoring

```sql
-- Slow query log
CREATE OR REPLACE FUNCTION log_slow_queries() RETURNS trigger AS $$
BEGIN
  IF NEW.duration > interval '1 second' THEN
    INSERT INTO slow_query_log (query, duration, timestamp)
    VALUES (NEW.query, NEW.duration, NOW());
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## CI/CD Validation

```bash
# Check no cross-graph edges
neo4j-admin cypher-shell --database neo4j --non-interactive \
  "MATCH (r1:Repository)-[:OWNS]->(a1:Artifact)-[e:RELATES]-(a2:Artifact)<-[:OWNS]-(r2:Repository) 
   WHERE r1 <> r2 
   RETURN count(e)" | grep "0"

# Verify projection lag
curl -s http://localhost:9090/api/v1/query \
  -d 'query=histogram_quantile(0.95, projection_lag_seconds[5m])' | \
  jq '.data.result[0].value[1]' | \
  awk '{if ($1 > 5) exit 1}'

# Test kill switch
GRAPH_READS_ENABLED=false curl -s http://localhost:8080/api/artifacts/test | \
  grep "501"
```