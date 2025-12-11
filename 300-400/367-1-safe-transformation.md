# backlog/367-1-safe-transformation – SAFe / Iteration Governance Profile

**Parent backlog:** [backlog/367-elements-transformation-automation.md](./367-elements-transformation-automation.md)

## Purpose
Model the Scaled Agile Framework (SAFe) constructs—Portfolio Epics, Program Increments (PIs), ART Iterations, PI Objectives, and cross-team dependencies—within the Transformation service, leveraging the shared methodology framework.

## Scope
- `profiles/safe.yaml` describing SAFe terminology, cadences (PI, Iteration), role matrix (Product Manager, Release Train Engineer, System Architect, Developers) via the shared `role_map`, and policy rules.
- Data model additions for `program_increments`, `iteration_goals`, `pi_objectives`, `cross_team_dependencies`, and ART/team associations, all linked to `methodology_profile_id`.
- Planning workflows: PI Planning event capture (Objectives, capacity), Iteration planning with commitment tracking, System Demo/Solution Demo evidence capture.
- Governance hooks for WSJF prioritization, guardrails (e.g., budget, guardrails compliance), and dependency risk visualization.
- Flow analytics at PI + Iteration levels (throughput, predictability measure, dependency aging).
- Protos & lexicon – `ameide.safe.v1` defines PIs, Iterations, PI Objectives, and Dependencies so UI surfaces can display SAFe terminology without relying on a neutral shared layer.
- Lifecycle DSL definition for Portfolio/Epic/Feature states, PI Objective commitment/acceptance, and guard predicates expressed as policy bundles.
- Nested timebox support: Sprints inherit from Iterations, which belong to PIs; Transformation UI exposes the hierarchy and allows WorkItems to contribute to multiple parent Timeboxes simultaneously.

### Graph vs. transformation runtime
- **Transformation runtime** executes SAFe ceremonies, enforces PI/Iteration policies, stores WSJF/predictability data, and manages role permissions in real time.
- **Graph service** remains the knowledge graph of architectural state. Every PI, Iteration, Objective, dependency, and evidence update must be projected as `graph.elements`/relationships so portfolio analytics and cross-methodology reports can reason over the same data, but the graph itself does not run SAFe workflows.

## Deliverables
1. **Schema/proto updates** – Entities for PI, Iteration, Objectives, Dependencies, WSJF fields; ability to associate backlog items with SAFe tiers (Portfolio -> Solution -> Program -> Team).
2. **Profile configuration & validation** – YAML + schema for SAFe rules, including required ceremonies and evidence expectations.
3. **Service/SDK APIs** – Neutral RPCs (e.g., `SetTimeboxGoal`, `AddDependency`, `AdvanceLifecycleState`) plus SAFe-specific helpers layered on top for PI Planning, objective scoring, and predictability reporting.
4. **UI updates** – PI planning board, dependency board, System Demo acceptance screen, inspectable ART/team lanes.
5. **Automation hooks** – Mapping automation plans/runs to PI Objectives and Iteration commitments so agents contribute to PI predictability metrics.

## Non-goals
- Scrum-specific boards (covered in the Scrum doc).
- TOGAF/enterprise architecture deliverables.

## Dependencies
- Methodology profile framework + Stage 0 data ingestion.
- Unified metamodel foundations from [backlog/300-ameide-metamodel.md](./300-ameide-metamodel.md) / [backlog/303-elements.md](./303-elements.md) so portfolio/program artifacts remain element-based.
- Org/tenant data to map teams/ARTs.

## Exit Criteria
- ARTs can execute SAFe ceremonies entirely within Transformation, with Product Managers owning backlogs, RTEs tracking dependencies/impediments, and System Architects reviewing automation outputs as part of PI/Solution Demos.
- Predictability metrics (planned vs. actual objectives) generated per PI.
- Coding/architecture agents register as actors (e.g., Solution Team) and their runs feed objective evidence automatically.

## Implementation guide
- **Schema**: add ProgramIncrement/Iteration tables (Timebox subtypes), PI Objectives (Goal subtype), WSJF fields, dependency mappings, ART/team associations, predictability metrics store.
- **Proto/contracts**: extend Buf proto set with PI/Iteration/Objective RPCs + dependency messages; regenerate SDKs and bump Transformation/portal clients.
- **Service**: Transformation gains APIs for PI Planning (objective CRUD, commitment), dependency management, predictability reporting; role map enforces PM/RTE/System Architect permissions.
- **UI**: PI Planning board (capacity, WSJF), dependency board, Iteration planning views, System/Solution Demo evidence panels.
- **Automation**: solution architect agent attaches dependency/evidence objects automatically; coding agent runs tag PI Objective IDs for predictability metrics.
- **Analytics**: predictability dashboard (planned vs achieved), dependency aging heatmap, ART-level flow metrics.
