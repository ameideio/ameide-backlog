# 587 — Kafka Topics & Queues Inventory (Fleet)

**Status:** Draft (inventory bootstrap)  
**Audience:** Platform/GitOps, runtime owners (Domain/Process/Projection/Integration/Agent), operators  
**Scope:** A single place to track **all Kafka topics used by Ameide**, split into:

1) **Contract topic families** (domain intents/facts, process facts) — semantic, proto-defined per `backlog/509-proto-naming-conventions.md`.  
2) **Execution queues** (work queues / KEDA scale targets) — operational topics used to trigger Jobs based on lag.

This backlog is the *fleet-level index*. Capability-specific deep dives (e.g., WorkRequests queues) should link from here.

## 1) Naming policy (baseline)

Topic names are logical and must remain stable across environments.

- **Contract topic families** (per 509):
  - `<capability>.<class>.<kind>.v<major>`
  - where `class ∈ {domain, process}`, `kind ∈ {intents, facts}`
  - examples: `scrum.domain.facts.v1`, `transformation.knowledge.domain.facts.v1`
- **Execution queues** (recommended; to avoid collisions across capabilities):
  - `<capability>.<executor_class>.<executor_kind>.v<major>`
  - examples (Transformation): `transformation.toolrun.verify.v1`, `transformation.agentwork.coder.v1`

Hard rule: execution queues MUST NOT be “mixed streams” of unrelated facts; they should contain only `WorkRequested`-class work items so scaling reacts only to explicit work requests.

## 2) Inventory: GitOps-provisioned Kafka topics (Strimzi `KafkaTopic`)

This table is the **actual** set of topics created by GitOps (Strimzi Topic Operator) today.

| Topic | Category | GitOps source | Enabled envs | Notes |
|---|---|---|---|---|
| `toolrun.verify.v1` | execution queue | `sources/values/_shared/data/data-kafka-workrequests-topics.yaml` | `local`, `dev` | WorkRequests queue (verify class); see 586 |
| `toolrun.generate.v1` | execution queue | `sources/values/_shared/data/data-kafka-workrequests-topics.yaml` | `local`, `dev` | WorkRequests queue (generate class); see 586 |
| `agentwork.coder.v1` | execution queue | `sources/values/_shared/data/data-kafka-workrequests-topics.yaml` | `local`, `dev` | WorkRequests queue (agent work); see 586 |

Capability inventories:

- WorkRequests execution queues: `backlog/586-workrequests-execution-queues-inventory.md`

## 3) Inventory: contract topic families (semantic; proto-defined)

These are not necessarily provisioned as Strimzi `KafkaTopic` objects yet, but they are the logical topics that define runtime seams.

Transformation (527) examples:

- `transformation.domain.intents.v1`
- `transformation.domain.facts.v1`
- `transformation.process.facts.v1`
- `transformation.knowledge.domain.intents.v1`
- `transformation.knowledge.domain.facts.v1`

Scrum (508) examples:

- `scrum.domain.intents.v1`
- `scrum.domain.facts.v1`
- `scrum.process.facts.v1`

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
- If namespacing changes (e.g., `toolrun.verify.v1` → `transformation.toolrun.verify.v1`), capture a migration plan and timebox the old names.
