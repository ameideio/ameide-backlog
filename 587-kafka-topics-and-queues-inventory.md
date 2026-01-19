# 587 — Kafka Topics & Queues Inventory (Fleet)

**Status:** Draft (inventory bootstrap)  
**Audience:** Platform/GitOps, runtime owners (Domain/Process/Projection/Integration/Agent), operators  
**Scope:** A single place to track **all Kafka topics used by Ameide**, split into:

1) **Semantic identities** (facts/intents) — contract-first, proto-governed per `backlog/496-eda-principles-v6.md` and `backlog/509-proto-naming-conventions-v6.md` (CloudEvents `type` is the contract; topics are delivery).  
2) **Execution queues** (work queues / KEDA scale targets) — operational topics used to trigger Jobs based on lag.

This backlog is the *fleet-level index*. Capability-specific deep dives (e.g., WorkRequests queues) should link from here.

## 1) Naming policy (baseline)

Kafka topic names are operational and must remain stable across environments.

**Contract surface (normative):**

- CloudEvents `type` is the semantic identity of the message (`io.ameide.*`), per `backlog/496-eda-principles-v6.md`.
- Topic naming may use `topic == ce.type` (simple default), or topic families (operational grouping), but topics are not the contract.

- **Legacy contract topic families** (historical; do not treat as the contract under v6):
  - `<capability>.<class>.<kind>.v<major>`
  - where `class ∈ {domain, process}`, `kind ∈ {intents, facts}`
  - examples: `scrum.domain.facts.v1`, `transformation.knowledge.domain.facts.v1`
- **Execution queues** (recommended; 509-aligned extension):
  - `<capability>.work.queue.<executor_class>.<executor_kind>.v<major>`
  - examples (Transformation): `transformation.work.queue.toolrun.verify.v1`, `transformation.work.queue.agentwork.coder.v1`

Hard rule: execution queues MUST NOT be “mixed streams” of unrelated facts; they should contain only **execution intent** work items (e.g., `WorkExecutionRequested`) so scaling reacts only to explicitly requested work and never to domain fact streams.

## 2) Inventory: GitOps-provisioned Kafka topics (Strimzi `KafkaTopic`)

This table is the **actual** set of topics created by GitOps (Strimzi Topic Operator) today.

| Topic | Category | GitOps source | Enabled envs | Notes |
|---|---|---|---|---|
| `transformation.work.queue.toolrun.verify.v1` | execution queue | `sources/values/_shared/data/data-kafka-workrequests-topics.yaml` | `local`, `dev` | WorkRequests queue (verify class); see 586 |
| `transformation.work.queue.toolrun.generate.v1` | execution queue | `sources/values/_shared/data/data-kafka-workrequests-topics.yaml` | `local`, `dev` | WorkRequests queue (generate class); see 586 |
| `transformation.work.queue.agentwork.coder.v1` | execution queue | `sources/values/_shared/data/data-kafka-workrequests-topics.yaml` | `local`, `dev` | WorkRequests queue (agent work); see 586 |

Capability inventories:

- WorkRequests execution queues: `backlog/586-workrequests-execution-queues-inventory.md`

## 3) Inventory: contract topic families (semantic; proto-defined)

These are not necessarily provisioned as Strimzi `KafkaTopic` objects yet.

Under v6, the **semantic identities** (CloudEvents `type`) define runtime seams; topics may map `topic == ce.type` or group multiple types under topic families.

Transformation (527) examples:

- Facts (examples): `io.ameide.transformation.fact.<name>.v1`
- Intents (examples): `io.ameide.transformation.intent.<name>.v1`

Scrum (508) examples:

- Facts (examples): `io.ameide.scrum.fact.<name>.v1`
- Intents (examples): `io.ameide.scrum.intent.<name>.v1`

## 4) Operational defaults (guidance; evolve per environment)

Execution queue defaults (local/dev):

- retention: short (1–7 days)
- cleanup: delete
- partitions: sized for expected parallelism (local may be 1)

Managed cluster defaults (staging/prod):

- TLS/SASL as required by cluster policy
- dedicated service accounts and least-privilege networking per executor class
- explicit retention + compaction policies (Kafka is transport, not evidence)

## 5) Change control (how this stays current)

When a PR adds/renames any Kafka topic (either as a Strimzi `KafkaTopic` or as a new logical topic family in contracts):

- Update this backlog (587) to keep the fleet index correct.
- Update the relevant capability backlog (e.g., 586 for WorkRequests queues).
- If namespacing changes, capture a migration plan and timebox the old names.
