# 042 – Refactor **core-platform-coder** agent to BPMN / wfdesc / Tool pattern

Date: 2025-07-25

Author: modelling-guild

## Context

Our evaluation of `packages/ameide_core-platform-coder` (Backlog #041) highlighted that the refactoring agent is currently expressed directly as LangGraph Python code.  This bypasses the model-driven toolchain used elsewhere in the platform, e.g. in *workflows-sample-order* where BPMN tasks with `impl="wfdesc"` are decomposed into AI agents via the code-generation builders.

Aligning the coder agent with the same **BPMN for deterministic wfdesc → builder** pipeline will give us:

* Uniform representation of all AI units (risk analysis, code refactor, etc.)
* Automatic generation of LangGraph scaffolding, tool wrappers and CI harnesses
* Easier visual editing / governance of the agent’s control-flow (gate, retry, etc.)

## Goal

Represent the test-refactoring agent as:

1. A **BPMN 2.0** process model describing the high-level control flow (load → analyse → plan → rewrite → gate → decision).  Use `ServiceTask` elements with `impl="tool"` for deterministic helpers and `impl="wfdesc"` for adaptive LLM calls.
2. A **wfdesc JSON-LD** file defining each AI “tool” (analysis LLM, planning LLM, rewriting LLM, review LLM).  Inputs/outputs correspond to message payloads in the LangGraph.
3. Tool definitions that reference reusable builder IDs (`codex-cli`, `claude-cli`, `openai‐threads`) so the code generator can stitch them in.

The existing python implementation will be replaced by generated code; any bespoke logic should be moved to *ToolAdapters* that the builder can import.

## Deliverables

1. `packages/ameide_core-platform-coder/resources/coder.bpmn` – BPMN model **(delivered 2025-07-25)**
2. `packages/ameide_core-platform-coder/resources/coder_agent.wfdesc.json` – wfdesc definitions **(delivered 2025-07-25)**
3. Updated builder configuration so that `build/codegen` emits the LangGraph agent and CLI wrapper from the models. *(open)*
4. Migration notes and README updates. *(open)*

## Tasks

### 1 – Draft BPMN model
* Map each current LangGraph node to a `ServiceTask` in a linear flow.
* Add an exclusive gateway after *gate* with retry branch (max 3) and success branch.
* Use BPMN extension `ameide:impl` (or Camunda’s `camunda:class`) to mark tool vs wfdesc.

### 2 – Create wfdesc definitions
* For each adaptive step (analysis, planning, review, rewrite) define a `wfdesc:Step` with `wfdesc:hasAgent` pointing at a generic builder (`gpt‐4` / `anthropic`).
* Expose input/output shapes so the generator can weld them to Python dataclasses.

### 3 – Extend builders
* Add templates that read the coder BPMN, resolve `impl="wfdesc"`, pull wfdesc, and emit LangGraph runnable + state classes.
* Ensure reuse of existing helper library for tool wrappers (`openai_tool`, `codex_tool`).

### 4 – Remove hand-coded graph
* Deprecate `src/ameide_core_platform_coder/{graph.py,nodes.py}` once generated code is verified.

### 5 – Tests & CI
* Update unit tests to import the generated package path (likely unchanged API).
* Confirm `pytest` and `ruff` still pass.

## Acceptance Criteria

* Running `bazel build //packages/ameide_core-platform-coder:dist` triggers the builder and succeeds without manual python sources.
* The generated agent reproduces current behaviour on `sample_test.py` (same state transitions, retries, gatekeeper logic).
* BPMN + wfdesc artefacts are version-controlled and validated by the model-lint step.

## Risks / Mitigations

* **Builder gaps** – may need new template features (e.g., exclusive gateway with loop counter).  Mitigation: spike quickly, budget 2 days for template work.
* **Loss of bespoke heuristics** – ensure `compute_quality_score` and AST helpers remain as importable utils; only orchestration is generated.
* **Schema drift** – lock Pydantic models in wfdesc outputs to avoid mismatches.

## Stakeholders

* Model-driven engineering team
* Core-platform-coder maintainers
* Dev-experience guild

## Related items

* #041 – quality / robustness improvements for core-platform-coder
* workflows-sample-order package – reference implementation of BPMN + wfdesc decomposition
