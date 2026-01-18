# 586 — WorkRequests Execution Queues (Kafka topics) Inventory

**Status:** Draft (inventory + naming decision pending finalization)  
**Audience:** Platform/GitOps, runtime owners (Process/Integration/Agent), operators  
**Scope:** Canonical inventory of **execution queue topics** used by the WorkRequests substrate (527 WP‑B) so we can:
1) keep names consistent with `backlog/509-proto-naming-conventions-v6.md`, and  
2) avoid accidental collisions as additional capabilities add their own WorkRequests queues.

**Fleet index:** `backlog/587-kafka-topics-and-queues-inventory.md` (all Kafka topics/queues)

> **Not in scope:** canonical domain/process fact streams (e.g., `transformation.*.facts.v1`) and their envelopes; those remain specified in 527 docs and protos.

## 1) Background (why these exist)

WorkRequests are an execution seam: Process requests work → Domain persists → an execution backend runs tools/agents and records evidence/outcomes.

KEDA scales by Kafka consumer group lag, so we use **dedicated execution queue topics that contain only execution intents** (e.g., `WorkExecutionRequested`), not mixed domain fact streams, to avoid scaling on unrelated events.

References:
- `backlog/527-transformation-capability.md` (§2.1 execution substrate)
- `backlog/527-transformation-integration.md` (§1.0.3 runner execution backend)
- `backlog/527-transformation-implementation-migration.md` (WP‑B GitOps inventory)

## 2) Naming policy (recommendation)

We should keep queue topics **namespaced by capability** (or by a shared `workrequests` context if/when WorkRequests becomes a cross-capability platform primitive) and keep the `.v<major>` suffix aligned to the payload’s major version per `backlog/509-proto-naming-conventions-v6.md`.

Recommended pattern for execution queues:

```text
<capability>.work.queue.<executor_class>.<executor_kind>.v<major>
```

Example (Transformation):
- `transformation.work.queue.toolrun.verify.v1`
- `transformation.work.queue.toolrun.generate.v1`
- `transformation.work.queue.agentwork.coder.v1`

**Current state (implemented today):** the topics are capability-prefixed and 509-aligned (`transformation.work.queue.*.v1`).

## 3) Inventory (current GitOps implementation)

Enabled environments:
- `local`: ✅ topics + workbench + ScaledJobs enabled
- `dev`: ✅ topics + workbench + ScaledJobs enabled (currently capped: `maxReplicaCount: 1`)
- `staging`/`production`: ❌ disabled (no rollout posture defined yet)

| Queue purpose | Kafka topic (current) | KafkaTopic resource | KEDA ScaledJob | Consumer group | Notes |
|---|---|---|---|---|---|
| Tool runs (verify) | `transformation.work.queue.toolrun.verify.v1` | `KafkaTopic/transformation-work-queue-toolrun-verify-v1` | `ScaledJob/workrequests-toolrun-verify` | `transformation-work-queue-toolrun-verify-v1` | KEDA schedules an executor-image Job that consumes `WorkExecutionRequested` intents from this queue and records outcomes/evidence back to Domain |
| Tool runs (generate) | `transformation.work.queue.toolrun.generate.v1` | `KafkaTopic/transformation-work-queue-toolrun-generate-v1` | `ScaledJob/workrequests-toolrun-generate` | `transformation-work-queue-toolrun-generate-v1` | KEDA schedules an executor-image Job that consumes `WorkExecutionRequested` intents from this queue and records outcomes/evidence back to Domain |
| Agent work (coder) | `transformation.work.queue.agentwork.coder.v1` | `KafkaTopic/transformation-work-queue-agentwork-coder-v1` | `ScaledJob/workrequests-agentwork-coder` | `transformation-work-queue-agentwork-coder-v1` | KEDA schedules an executor-image Job that consumes `WorkExecutionRequested` intents from this queue and records outcomes/evidence back to Domain |

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

## 5) Rename plan (if we ever change names again)

We can rename without data migration concerns because these queues are transport-only; correctness is backed by Domain persistence + evidence storage.

If we rename again:

1. Create new KafkaTopic resources for the new names.
2. Update ScaledJobs/producers to point at the new topics.
3. Keep old topics temporarily only if there are active producers/consumers.
4. Delete old topics when empty + unused.

## 6) Follow-ups (to track explicitly)

- Decide whether WorkRequests queues are:
  - **capability-namespaced** (`transformation.*`), or
  - **platform-namespaced** (`workrequests.*` with `capability` carried in payload).
- Lock the execution queue message payload contract:
  - for Transformation: `WorkExecutionRequested` (`ameide_core_proto.transformation.work.v1`)
  - for other capabilities: follow `backlog/509-proto-naming-conventions-v6.md` (execution queues are operational and intent-only).
- Define staging/production rollout posture (security, secrets, TLS/SASL, NetworkPolicy/RBAC per executor class).
