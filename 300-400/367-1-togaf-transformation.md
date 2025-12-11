# backlog/367-1-togaf-transformation – TOGAF / ADM Governance Profile

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)

## Purpose
Enable Transformation initiatives to follow TOGAF’s Architecture Development Method (ADM) and governance artifacts (Architecture Vision, Business/Data/Application/Technology architecture deliverables, Architecture Contracts) using the shared methodology framework.

## Scope
- `profiles/togaf.yaml` capturing ADM phases (Prelim, A–H), required deliverables per phase, approval roles (Architecture Board, Enterprise Architect, Sponsor) through the `role_map`, and compliance checkpoints.
- Schema additions for `adm_phases`, `adm_deliverables`, `architecture_contracts`, and `compliance_reviews`, each tied to `methodology_profile_id` and linked to elements/backlog items.
- Workflow automation: phase gate progression, Architecture Board review packages, contract sign-off, deviation/dispensation tracking.
- UI experiences tailored to ADM (phase dashboards, deliverable templates, review agendas).
- Integration with automation plans so agents can produce specific TOGAF deliverables (e.g., EA agent generating reference models, coding agent delivering implementation proof).
- Protos & role mapping – `ameide.togaf.v1` defines ADM Phases, Deliverables, Architecture Contracts, Board decisions, and roles (Architecture Board, Enterprise Architect, Sponsor) directly. Role maps simply bind users to these first-class messages.
- Lifecycle & guards – phase transitions, deliverable requirements, and approvals live in the TOGAF service + watcher instead of a shared DSL; profiles ship configuration the runtime evaluates.

### Graph vs. transformation runtime
- **Transformation runtime** hosts ADM lifecycles, approvals, contract workflows, and compliance automation.
- **Graph** keeps the canonical representation of ADM deliverables, contracts, and relationships to the rest of the architecture. Every phase transition, deliverable submission, and contract update should emit the matching element/relationship mutations so reports and graph analytics remain authoritative, without overloading the graph service with runtime-specific logic.

## Deliverables
1. **Schema/proto** – Tables/messages for phases, deliverables, contracts, review outcomes, exception logs.
2. **Profile config & validation** – YAML describing deliverable templates, required evidence, approval lattices, and risk classification.
3. **Service/SDK APIs** – RPCs to advance ADM phases, submit deliverables, record Architecture Board decisions, manage contracts.
4. **UI/Docs** – Phase dashboards, deliverable templates, review automation (notifications, agendas, minutes capture).
5. **Compliance analytics** – Tracking adherence to ADM timelines, deviations granted, and contract status.

## Non-goals
- Agile/Scrum/SAFe-specific ceremonies.
- Release governance (handled later).

## Dependencies
- Stage 0 elements with methodology metadata.
- Element/relationship schema decisions from [backlog/300-ameide-metamodel.md](./300-ameide-metamodel.md) to ensure ADM deliverables and contracts remain first-class elements.
- Enterprise org hierarchy (EA lead, Architecture Board roster).

## Exit Criteria
- Architecture teams can execute TOGAF ADM entirely inside Transformation: from Vision through Implementation Governance, with agents contributing deliverables and Architecture Board approvals logged.
- Phase progression enforces deliverable completion/evidence per profile while still allowing exceptions (with rationale) captured in the compliance log.

## Implementation guide
- **Schema**: tables for ADM phases, deliverables, Architecture Contracts, compliance reviews, exception/dispensation logs; link `togaf.v1` deliverables/contracts back to graph elements.
- **Proto/contracts**: add Buf proto messages/RPCs for ADM phases, deliverables, board actions, and exceptions; regenerate SDKs.
- **Service**: RPCs to advance phases, submit deliverables (Evidence references), record Architecture Board decisions, manage contracts; role map for Architecture Board vs EA actions.
- **UI**: phase dashboard, deliverable templates, Board meeting agenda/minutes tooling, deviation tracking.
- **Automation**: EA agent templates deliverables, attaches Evidence to phases/contracts, raises impediments when guardrails fail.
- **Policy**: phase gate guards defined in lifecycle DSL; release gating checks ADM evidence before allowing promotion.
