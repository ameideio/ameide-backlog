# 507 – Scrum → Process → Agent Reference Map

**Status:** Draft reference (living document)  
**Audience:** Architecture, Transformation, Process, Agent teams  
**Purpose:** Provide a single entry point that links the Scrum governance plane (Stage 1), the Process/event layer (Stage 2), and the agent execution layer (505) so teams can navigate the stack without chasing scattered backlogs.

**Authority & supersession**

- This backlog is **authoritative for navigation only**: it maps which backlogs own which parts of the Scrum → Process → Agent stack.  
- **Scrum event/intent envelopes and runtime seams** are owned by `506-scrum-vertical-v2.md` (plus the `transformation_scrum_*` protos in `508-scrum-protos.md`).  
- **PO/SA/Coder role mappings, work handover semantics, and artifacts** are owned by `505-agent-developer-v2.md` and `505-agent-developer-v2-implementation.md`.
- **Scrum data model and profiles** are owned by `300-400/367-1-scrum-transformation.md`.  
- If this file contradicts any of those, **prefer 506-v2/508/505/367-1**. Older docs (`300-400/367-0-feedback-intake.md`, `505-agent-developer.md`, `506-scrum-vertical.md`) are historical and must not override the newer contracts.

**Contract surfaces (this doc references, does not define)**

- **Domain intents/facts topics**: `scrum.domain.intents.v1`, `scrum.domain.facts.v1` (defined in 506-v2/508).  
- **Process facts topic**: `scrum.process.facts.v1` (defined in 506-v2).  
- **Optional A2A REST binding + Agent Card** (transport binding only): defined in 505-v2.
- **Operator CRDs and condition vocabulary**: defined in 495/498/499/500/501/502.

## Grounding & cross-references

- **Architecture grounding:** Builds on the primitive/EDA foundations in `470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, `477-primitive-stack.md`, and `496-eda-principles.md`.  
- **Stage 1 (Transformation Scrum profile):** Anchors to `300-400/367-1-scrum-transformation.md` and the `transformation_scrum_*` protos in `508-scrum-protos.md` as the Scrum system-of-record.  
- **Stage 2 (Process primitives):** Uses the runtime seam in `506-scrum-vertical-v2.md` and Process operator semantics from `499-process-operator.md` to place Temporal workflows in the stack.  
- **Stage 3 (Agents & tooling):** Maps AmeidePO/AmeideSA/AmeideCoder architecture in `505-agent-developer-v2.md` and implementation plan in `505-agent-developer-v2-implementation.md` onto the primitive/operator/CLI patterns in `500-agent-operator.md`, `504-agent-vertical-slice.md`, and the 484a–484f CLI backlogs.

---

## 0. Layers at a glance

| Layer | Responsibility | Authoritative backlog(s) | Key artifacts |
|-------|----------------|--------------------------|---------------|
| **Stage 0 – Intake** | Capture requests, tag methodology profile | `367-0-feedback-intake.md`, `367-1-safe-transformation.md` (for other profiles) | Feedback nodes, profile assignment, `methodology_profile_id` resolution |
| **Stage 1 – Transformation (Scrum profile)** | Store Scrum data (Product Backlog, Product Backlog Items, Sprints, Increments) and UI/policy | `367-1-scrum-transformation.md`, `508-scrum-protos.md` | Methodology YAML, `transformation_scrum_*` protos/schema updates, portal workflows |
| **Stage 2 – Process primitive(s)** | Temporal workflows for timeboxes, process fact emission | `506-scrum-vertical-v2.md`, `499-process-operator.md` | Process CRDs, Temporal workers, event contracts |
| **Stage 3 – Agent execution** | AmeidePO/AmeideSA/AmeideCoder role mappings, executor runtime | `505-agent-developer-v2.md`, `505-agent-developer-v2-implementation.md`, `500/504` | Agent Definitions, work handover topics, CLI guardrails (optional A2A binding) |
| **Support – Primitive placement & EDA** | Ensure consistent boundaries/application layout | `477-primitive-stack.md`, `496-eda-principles.md` | Placement diagrams, event invariants |
| **Support – CLI orchestrator & guardrails** | Tooling agents use internally | `484a-ameide-cli-primitive-workflows.md` … `484f`, `520-primitives-stack-v2.md`, `backlog/521c-internal-generation-improvements.md`, `backlog/521d-external-generation-improvements.md` | Orchestration wrappers (Buf under the hood), repo/gitops handling, prompts, CI guardrails |
| **Support – Operator specs** | Runtime control planes | `495-ameide-operators.md`, `500-agent-operator.md`, `498/499/501` | CRDs, helm charts, GitOps overlays |

Supporting invariants: `477-primitive-stack.md` (placement of primitives) and `496-eda-principles.md` (event semantics).

---

## 0.1 Timeline view

```
Stage 0 (367-0) ──┐
                  ▼
Stage 1 (367-1/508) ──┐ Emits Scrum domain facts (e.g., ProductBacklogItemCreated/DoneRecorded, SprintCreated/Started, SprintBacklogCommitted, IncrementUpdated)
                      ▼
Stage 2 (506-v2/499) ──┐ Emits process facts (e.g., SprintBacklogReadyForExecution, SprintBacklogItemReadyForWork, SprintTimeboxReachedEnd)
                        ▼
Stage 3 (505/500/504) ──► AmeidePO → AmeideSA → AmeideCoder → Transformation
```

Each hand-off is asynchronous (EDA intents/facts plus work handover events; optional A2A as a transport binding) and enforced by the operators defined in 495/498/499/500/501. At **runtime**, Process workflows interact with Transformation only via bus messages (`scrum.domain.intents.v1` / `scrum.domain.facts.v1`); any synchronous definition fetch by the Process operator (499) is a **control‑plane** concern, not a domain state transition.

---

## 1. Stage 0 – Intake & profile resolution (367-0 recap)

- `367-0-feedback-intake.md` captures requests, normalizes them into Transformation elements, and attaches `methodology_profile_id` metadata.  
- Resolution order (tenant → initiative → product) determines which Stage 1 profile (Scrum/SAFe/TOGAF) applies; the logic is shared across profiles.  
- Intake emits events that Stage 1 consumes to create backlog items, Product Goals, or other artifacts.

**Feeds:** Stage 1 reads profile assignments from the Transformation database; Process + agents rely on the same metadata to subscribe to the right events.

---

## 2. Data & governance (367-1 / 508 recap)

- Transformation is the **system of record** for Scrum artifacts (Product Backlog, Product Backlog Items, Sprints, Sprint Backlogs, Product Goals, Sprint Goals, Increments, Definitions of Done, plus optional impediment extensions).  
- Scrum schema/proto definitions live in `services/transformation` and `packages/ameide_core_proto/transformation/scrum/v1` (see `508-scrum-protos.md`).  
- Portal UX and profile YAML define ceremonies and optional working-agreement policies (e.g., readiness checklists), while Scrum Guide compliance centers on Definition of Done and the official artifacts/commitments.
- Stage 0 intake (`367-0-feedback-intake.md`) feeds Transformation with profile assignments; adjacent profiles (e.g., TOGAF, SAFe) live under the same 367-1 folder hierarchy.

**Feeds:** Process primitives subscribe to the Scrum domain intents/facts emitted here (see §3 and `506-scrum-vertical-v2.md`). AmeidePO uses the SDKs generated from these protos to fetch Product Backlog Item state and issue commands via intents.

---

## 3. Process + events (506-v2 recap)

- `506-scrum-vertical-v2.md` specifies the **intent vs. fact** envelopes, routing keys, and Temporal workflows for Scrum.  
- Process primitives (managed by the Process operator, backlog `499`) **consume Scrum domain facts** from `scrum.domain.facts.v1` (e.g., `SprintCreated`, `SprintStarted`, `SprintBacklogCommitted`, `ProductBacklogItemDoneRecorded`, `IncrementUpdated`) and emit **process facts** such as `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`, `SprintTimeboxReachedEnd`, `SprintEndingSoon` on `scrum.process.facts.v1`. They never mutate Transformation state directly.  
- Domain write requests such as `CreateSprintRequested`, `CommitSprintBacklogRequested`, `RecordProductBacklogItemDoneRequested`, and `RecordIncrementRequested` travel from AmeidePO/Process → Transformation as **domain intents** on `scrum.domain.intents.v1` using the domain SDKs.
- Process operator & CRD guidance lives in `499-process-operator.md` and `495-ameide-operators.md`; refer there for deployment patterns and GitOps wiring.

**Feeds:** AmeidePO subscribes primarily to process facts (e.g., `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`), AmeideSA receives delegation context through these events/commands, and the executor receives work intents/facts (optionally exposed via A2A).

---

## 4. Agent execution (505 recap)

- `505-agent-developer-v2.md` defines the Product Owner / Solution Architect / Coder mapping, with bus-native work handover and optional A2A transport binding.
- The implementation plan (`505-agent-developer-v2-implementation.md`) outlines the concrete workstreams (Process CRDs, AmeidePO/SA DAGs, executor runtime, operator updates).
- `500-agent-operator.md` and `504-agent-vertical-slice.md` cover runtime management and guardrails that AmeideCoder uses internally (CLI orchestration wrappers + CI gates).
- CLI guardrails and prompts are further detailed in the `484a-484f` backlog series; `520-primitives-stack-v2.md`, `backlog/521c-internal-generation-improvements.md`, and `backlog/521d-external-generation-improvements.md` define the “CLI orchestrates, Buf generates, CI gates” split that prevents the CLI from becoming a bespoke generator.

**Inputs:** Process events (from 506) + Transformation data (from 367-1).  
**Outputs:** Evidence (`artifact.*` payload), Transformation commands, GitHub PRs, work handover telemetry (optional A2A task telemetry).

---

## 5. Navigating the repository

| Concern | Primary location | Notes |
|---------|------------------|-------|
| Transformation schema/protos | `services/transformation`, `packages/ameide_core_proto/transformation/scrum/v1` | Update alongside 367-1 and 508 |
| Temporal workflows | `primitives/process/*`, `operators/process-operator` | Follow 506-v2 & 499 |
| Agent DAGs | `primitives/agent/ameide-po`, `ameide-sa`, `ameide-coder` | Align with 505 plan |
| Devcontainer service | `services/devcontainer_service` | Executor runtime (may expose A2A binding per 505) |
| CLI guardrails | `packages/ameide_core_cli`, `prompts/agent/*` | Orchestrator wrappers; internal generation remains `buf generate`; CI is canonical (`520-primitives-stack-v2.md`, `backlog/521c-internal-generation-improvements.md`, `backlog/521d-external-generation-improvements.md`) |
| GitOps manifests | `gitops/ameide-gitops/.../primitives/{process,agent}` | Ensure sequencing Process → PO → SA → Coder |
| Primitive placement & operators | `477-primitive-stack.md`, `495/498/500/501` | Repository layout + operator responsibilities |

---

## 6. How to use this map

1. **Starting from Transformation:** When adding Scrum data or policies, cross-check §2 to ensure the required events/commands exist for Process + agents.  
2. **Starting from Process:** When wiring new workflows, validate the event payloads with 367-1’s schema and emit telemetry per 496.  
3. **Starting from agents:** Use this map to ensure AmeidePO/AmeideSA handle the right inputs (Process events vs. Transformation commands) and to locate the underlying services they depend on.

---

## 7. Future extensions

- Add SAFe/TOGAF variants once their Stage 1/Stage 2/Stage 3 backlogs mature.  
- Integrate this reference into portal/CLI docs so contributors can jump directly to the relevant backlog.
- Expand cross-layer maps for security (risk tiers, policies), observability (metrics/logs per layer), and tenant isolation once their respective backlogs (e.g., 362, 420) converge.
