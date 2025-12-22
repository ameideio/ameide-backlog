# 586 — WorkRequests Execution Queues (Kafka topics) Inventory

**Status:** Draft (inventory + naming decision pending finalization)  
**Audience:** Platform/GitOps, runtime owners (Process/Integration/Agent), operators  
**Scope:** Canonical inventory of **execution queue topics** used by the WorkRequests substrate (527 WP‑B) so we can:
1) keep names consistent with `backlog/509-proto-naming-conventions.md`, and  
2) avoid accidental collisions as additional capabilities add their own WorkRequests queues.

> **Not in scope:** canonical domain/process fact streams (e.g., `transformation.*.facts.v1`) and their envelopes; those remain specified in 527 docs and protos.

## 1) Background (why these exist)

WorkRequests are an execution seam: Process requests work → Domain persists → an execution backend runs tools/agents and records evidence/outcomes.

KEDA scales by Kafka consumer group lag, so we use **dedicated queue topics that contain only WorkRequested work items** (not mixed “domain facts”) to avoid scaling on unrelated events.

References:
- `backlog/527-transformation-capability.md` (§2.1 execution substrate)
- `backlog/527-transformation-integration.md` (§1.0.3 runner execution backend)
- `backlog/527-transformation-implementation-migration.md` (WP‑B GitOps inventory)

## 2) Naming policy (recommendation)

We should keep queue topics **namespaced by capability** (or by a shared `workrequests` context if/when WorkRequests becomes a cross-capability platform primitive) and keep the `.v<major>` suffix aligned to the payload’s major version per `backlog/509-proto-naming-conventions.md`.

Recommended pattern for execution queues:

```text
<capability>.<executor_class>.<executor_kind>.v<major>
```

Example (Transformation):
- `transformation.toolrun.verify.v1`
- `transformation.toolrun.generate.v1`
- `transformation.agentwork.coder.v1`

**Current state (implemented today):** the topics are global/un-prefixed (`toolrun.verify.v1`, …). This inventory tracks them and the eventual rename plan.

## 3) Inventory (current GitOps implementation)

Enabled environments:
- `local`: ✅ topics + workbench + ScaledJobs enabled
- `dev`: ✅ topics + workbench enabled, ScaledJobs disabled
- `staging`/`production`: ❌ disabled (no rollout posture defined yet)

| Queue purpose | Kafka topic (current) | KafkaTopic resource | KEDA ScaledJob | Consumer group | Notes |
|---|---|---|---|---|---|
| Tool runs (verify) | `toolrun.verify.v1` | `KafkaTopic/toolrun-verify-v1` | `ScaledJob/workrequests-toolrun-verify` | `workrequests-toolrun-verify-v1` | Placeholder container until real consumer exists |
| Tool runs (generate) | `toolrun.generate.v1` | `KafkaTopic/toolrun-generate-v1` | `ScaledJob/workrequests-toolrun-generate` | `workrequests-toolrun-generate-v1` | Placeholder container until real consumer exists |
| Agent work (coder) | `agentwork.coder.v1` | `KafkaTopic/agentwork-coder-v1` | `ScaledJob/workrequests-agentwork-coder` | `workrequests-agentwork-coder-v1` | Placeholder container until real consumer exists |

GitOps sources (authoritative):

- Topics:
  - `sources/values/_shared/data/data-kafka-workrequests-topics.yaml`
  - `environments/_shared/components/data/core/kafka-workrequests-topics/component.yaml`
- ScaledJobs + workbench:
  - `sources/values/_shared/apps/workrequests-runner.yaml`
  - `environments/_shared/components/apps/runtime/workrequests-runner/component.yaml`

## 4) Operational defaults (current)

- Kafka topic config:
  - `partitions=1`, `replicationFactor=1` (local/dev)
  - `retention.ms="604800000"` (7d; string to avoid scientific notation)
- KEDA Kafka scaler:
  - `bootstrapServers`: namespace-qualified (e.g., `kafka-kafka-bootstrap.ameide-local:9092`) because scaler runs in `keda-system`
  - `lagThreshold="1"`, `offsetResetPolicy=latest`

## 5) Rename plan (when we choose to namespace)

We can rename without data migration concerns because these queues are transport-only and are currently not fed by any production consumer.

Suggested migration steps:

1. Create new KafkaTopic resources for the namespaced topics.
2. Update ScaledJobs to point at the new topics (and keep old topics temporarily if any producers exist).
3. Update producers to publish to the new topics.
4. Delete old topics when empty + unused.

Mapping (proposal):

- `toolrun.verify.v1` → `transformation.toolrun.verify.v1`
- `toolrun.generate.v1` → `transformation.toolrun.generate.v1`
- `agentwork.coder.v1` → `transformation.agentwork.coder.v1`

## 6) Follow-ups (to track explicitly)

- Decide whether WorkRequests queues are:
  - **capability-namespaced** (`transformation.*`), or
  - **platform-namespaced** (`workrequests.*` with `capability` carried in payload).
- Define the queue message payload contract (`WorkRequestQueueMessage` or similar) and lock `.v1` semantics.
- Define staging/production rollout posture (security, secrets, TLS/SASL, NetworkPolicy/RBAC per executor class).
