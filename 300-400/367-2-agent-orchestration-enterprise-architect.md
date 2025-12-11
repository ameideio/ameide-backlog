# backlog/367-2-agent-orchestration-enterprise-architect – Enterprise Architecture Automation

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)

## Purpose
Specialize the generic agent orchestration platform for Enterprise Architect personas (e.g., TOGAF or SAFe System Architect roles). Focuses on producing architecture deliverables, reviewing models, and enforcing governance guardrails.

## Scope
- Define EA-specific repo adapters/workspace templates (modeling tools, diagram exports, compliance scripts).
- Methodology-aware prompts and guardrails (e.g., TOGAF deliverable templates, architecture principles) injected into runs.
- Evidence packaging targeted at Architecture Board reviews (diagrams, rationale docs, compliance checklists) registered as `governance.v1.Attestation` records with correct type/URI/checksum.
- Integration with graph service to fetch/update architecture elements, relationships, and reference models.
- Escalation flow for items requiring human architect approval or exception handling.

## Deliverables
1. **Agent profiles** – LangGraph/LangChain definitions for EA tasks, including tool access (model transformations, document generation).
2. **Execution policies** – Allowed repositories/paths, modeling tool automation, export formats, compliance validators.
3. **Run adapters** – Scripts to produce TOGAF deliverables (Architecture Definition Document, Roadmaps), cross-link results to elements.
4. **UI hooks** – Review panels for Architecture Board to inspect agent outputs, add comments, accept/reject deliverables.
5. **Telemetry** – Metrics on EA automation success, time saved, exception frequency, and evidence/impediment volumes tied to Architecture Board outcomes.

## Non-goals
- Software delivery automation (coding agent) or solution-level planning (covered elsewhere).

## Dependencies
- Methodology profile (TOGAF/SAFe) definitions referencing EA roles.
- Generic scheduler (367-2-agent-orchestration-generic.md).
- Graph/metamodel unification from [backlog/300-ameide-metamodel.md](./300-ameide-metamodel.md) so generated deliverables stay in the canonical element graph.

## Exit Criteria
- Enterprise Architects can delegate document/model preparation to agents with traceable outputs linked to methodology deliverables.
- Architecture Board workflows integrate agent evidence, with approvals/feedback cycling back into backlog items.
- Compliance policy violations automatically raise impediments/exceptions for EA review.

## Implementation guide
- **Agent definition**: design LangGraph flows for EA deliverables (model extraction, document generation) with access to modeling tools/API.
- **Proto/contracts**: extend Buf definitions with deliverable/Evidence metadata required for EA workflows; regenerate SDKs so services/UI share the same structures.
- **Repo adapters**: create EA-focused adapters (diagram exporters, compliance scripts) with guardrails.
- **Integration**: use graph service APIs to fetch/update architecture elements and link outputs to deliverables/contracts.
- **UI**: build Architecture Board review panel showing agent outputs, comments, accept/reject actions.
- **Policy**: map deliverables to Evidence requirements per ADM phase; auto-raise impediments when validation fails.
