Purpose

Build a two-part AI development assistant where Architect (in the Vercel app) turns user intent into a precise plan, and Coder (in a container with repo access) executes the plan via diffs and tests. Focus is on clear thinking, decisive control points, and a smooth human-in-the-loop—not tooling minutiae.

Outcomes

Fast planning → actionable plan JSON users can approve or tweak.

Safe code changes via reviewable diffs and mandatory approvals at risky steps.

Continuous visibility: everything streams (tokens, tool status, test output).

Resumable runs with deterministic checkpoints.

Artifacts-first UX: Architect outputs (stories, epics, ACs, risks, sprint goals) appear as cards in the Next.js Chatbot Artifacts UI, editable inline.

Core Principles

Plan is the contract: Architect produces a machine-readable plan that drives the run.

Interrupts at control points: Human approves/edits before irreversible actions (applying patches, pushing commits).

Stream everything: Status, partial outputs, and logs are surfaced live.

Idempotent + resumable: Each step can be retried or rolled back deterministically.

Least privilege: Coder’s environment has the minimum rights needed for the task.

Scope & Non‑Goals

In scope

Single-user flows, one repo at a time.

Planning, patch generation, application, test run, and result summarization.

Approvals for patch apply and optional push/PR creation.

Not in scope (v0)

Multi-tenant policy, project scaffolding from scratch, full IDE replacement, advanced sandboxing/VM orchestration.

Actors & Responsibilities

User: Provides intent, approves decisions, can edit the plan/patch.

Architect (Vercel): Converts intent → structured plan; emits artifacts (epics, stories, ACs, risks, sprint goal) to the Next.js Chatbot Artifacts UI; clarifies risks; proposes acceptance tests.

Coder (Container): Clones repo, generates patch per task, pauses for approval, applies patch, runs tests, streams logs.

System Overview (solution sketch)

User ──(prompt)──▶ Architect (Vercel)
  ▲                         │
  │        plan JSON        ▼
  └── approve/edit ◀─── UI streams

Architect ──▶ Coder (container)
    plan, repo, thread id
          │
          ├─ clone ▶ generate diff ▶ INTERRUPT(apply?) ▶ apply ▶ test ▶ result
          │                           ▲                     │
          └─────────── stream status/logs ──────────────────┘

Scrum/Agile organization (Architect duties)

Architect outputs are shaped to Scrum/Agile so the plan doubles as a lightweight backlog:

Product goal & Sprint goal: concise, outcome-focused.

Epics: grouped outcomes with rationale.

User stories (INVEST): As a  I want  so that .

Acceptance criteria: Gherkin-style Given/When/Then per story.

Priority & estimate: MoSCoW priority + story points (with confidence).

Dependencies & risks: per story.

Definition of Ready/Done: short checklists per story.

Sprint candidates: suggested set sized to team capacity.

Architect also performs a quick backlog refinement pass: flags ambiguity, adds/adjusts ACs, and proposes small spikes when uncertainty is high.

Architect artifacts in the Next.js Chatbot Artifacts UI

Principle: artifacts are first-class; the plan is a snapshot derived from them.

Artifact types

Plan summary (product goal, sprint goal)

Epic

User story (INVEST)

Acceptance criteria set (Gherkin)

Risk

Sprint selection / capacity

Minimal artifact shape

{ "id":"US-001", "type":"story", "title":"/health endpoint", "body":"As an SRE...", "relations":{"epic_id":"E-001"}, "ac":["Given... When... Then..."], "priority":"Must", "estimate":{"points":1,"confidence":0.8}, "selected_for_sprint": true }

Streaming (Architect → UI)

{ "type":"artifact", "op":"add|update|remove", "item": { /* artifact */ } }

User actions (thinking-focused)

Approve / edit stories and ACs

Prioritize, mark sprint candidates, adjust capacity

Send to Coder → materialize Plan v2 snapshot and start the run

Source of truth

The artifacts store (per thread_id) in the Next.js app is canonical.

Plan v2 and the Coder run payload are deterministic snapshots of that store at send-time.

Data Contracts (thin but decisive)

Plan v2 (Agile/Scrum Plan; Architect → UI/Container)

{
  "product_goal": "Expose health and sum endpoints with tests",
  "sprint": { "goal": "Operational endpoints in place", "capacity_points": 5, "stories_selected": ["US-001","US-002"] },
  "epics": [
    { "id": "E-001", "title": "Observability & Health", "why": "Detect outages early" }
  ],
  "backlog": [
    {
      "id": "US-001",
      "epic_id": "E-001",
      "story": "As an SRE, I want a /health endpoint so that monitoring can detect outages.",
      "acceptance_criteria": [
        "Given the service is running, When I GET /health, Then I receive 200 and body {\"ok\": true}"
      ],
      "priority": "Must",
      "estimate": { "points": 1, "confidence": 0.8 },
      "dependencies": [],
      "risk": "None",
      "definition_of_ready": ["Story unambiguous", "ACs in Gherkin", "Owner set"],
      "definition_of_done": ["Tests pass", "Lint/format", "Endpoint documented"],
      "tasks": [
        { "id": "T1", "desc": "Implement /health in app/main.py", "tests": ["tests/test_health.py"] }
      ],
      "sprint_candidate": true
    },
    {
      "id": "US-002",
      "epic_id": "E-001",
      "story": "As a developer, I want a /sum endpoint so that clients can compute a total on server.",
      "acceptance_criteria": [
        "Given numbers [1,2,3], When I POST /sum, Then response is 6 with 200"
      ],
      "priority": "Should",
      "estimate": { "points": 2, "confidence": 0.7 },
      "dependencies": ["US-001"],
      "risk": "Input validation edge cases",
      "definition_of_done": ["Happy + error paths tested"],
      "tasks": [
        { "id": "T2", "desc": "Implement /sum in app/main.py", "tests": ["tests/test_sum.py"] }
      ],
      "sprint_candidate": true
    }
  ]
}

Notes

Architect validates INVEST and AC quality; if missing, it proposes ACs.

Coder consumes stories_selected → flattens to tasks in order.

Each task maps to exactly one proposed unified diff + minimal tests.

Start Coder Run (Vercel → Container)

{
  "thread_id": "uuid-123",
  "repo": {"git_url":"https://github.com/org/repo","ref":"feature-branch"},
  "plan": { /* Plan v2 snapshot from Artifacts UI */ }
}

Streamed Events (Container → Vercel UI)

{ "type":"status", "stage":"checkout|codegen|apply|test", "message":"..." }
{ "type":"patch",  "task":"T1", "unified_diff":"--- a/..." }
{ "type":"interrupt", "interrupt_id":"abc", "payload": { "kind":"apply_patch", "diff_preview":"..." } }
{ "type":"result", "summary":"ok|failed", "details": { "tests_passed": true, "stats": {"passed": 8, "failed": 0} } }

Resume (UI → Container)

{ "thread_id":"uuid-123", "value":"approve|reject|edit", "edit": {"unified_diff":"..."} }

State & Control Flow (LangGraph in the Coder)

Nodes (linear for v0): checkout → codegen → interrupt(apply) → apply → test → summarize (for selected sprint stories only)

checkout: clone/checkout ref; detect stack; stream repo summary.

codegen: for each selected story’s tasks, emit proposed unified diff aligned to the story’s ACs.

interrupt(apply): block and surface the diff preview per story/task. Options: approve, reject, or edit.

apply: git apply with safety flags; on conflict, surface conflict hunks as an interrupt.

test: run tests; stream lines and a final summary; map pass/fail back to each story’s ACs.

summarize: report sprint goal coverage: which ACs met, which stories remain.

Retry policy

Any node can be retried with the same inputs (idempotent). On repeated failure, escalate to user with diagnosis.

Human-in-the-Loop
 (where thinking matters)

Before code is touched: User sees the plan and can edit tasks/acceptance.

Before apply: Mandatory approval; user can inline-edit the diff.

On failures: The system proposes one minimal follow-up patch; user decides to continue or stop.

Failure Modes & Responses

Patch doesn’t apply: Show conflict hunks; offer auto-rebase attempt or smaller patch split.

Tests fail: Surface failing tests + traceback; propose targeted fix with a new diff; pause.

Ambiguous repo layout: Ask a single clarifying question or fall back to scaffolding minimal files (behind approval).

Security & Trust (essentials only)

Scope: Coder has repo-scoped token and ephemeral workspace; no access to user secrets beyond what’s needed.

Controls: Only whitelisted commands (git, test runner, formatter). No network egress by default (configurable).

Audit: Persist plan, diffs, approvals, and test summaries per thread_id.

Performance Targets (v0)

Planning: < 5s perceived (stream tokens early).

First patch: < 20s for small edits (< 10 files).

Test window: soft cap 90s; stream partial output.

Milestones

MVP: Plan JSON → single patch → approve → apply → run tests → summarize.

v1: Multi-task sequencing, partial applies, better error recovery.

v2: Repo-aware context (indexing), AST edits fallback, PR creation.

Open Questions

Do we enforce a single mandatory apply-approval, or also require approval to run tests (CI load)?

How do we represent cross-file refactors in the plan (task graph vs. flat list)?

Where should edited diffs be stored (temp artifact vs. branch commit)?


