# 030 – Coding Agent Implementation Recipe

Status: **Planning**  
Created: 2025-07-23  
Updated: 2025-07-24  

> **Superseded – 2025-02:** This standalone “coding agent recipe” is kept for historical reference only. The cluster-grade implementation, guardrails, and GitOps deployment story now live in [505-agent-developer.md](../505-agent-developer.md) (agent-developer) together with the Agent vertical slice ([504-agent-vertical-slice.md](../504-agent-vertical-slice.md)) and Transformation guardrails from the 484 backlog series. Use those documents for any new work; only refer to this file when you need to understand the earlier bare‑metal/CICD experiments.

### Relationship to current plan

* **Runtime + tooling:** agent-developer defines `runtime_type=langgraph`, `dag_ref`, and the `develop_in_container` tool backed by the Ameide devcontainer service. The ad-hoc Docker/CLI commands in this file are obsolete.
* **Process + guardrails:** All coding agents must follow the OBSERVE→REASON→ACT→VERIFY workflow codified in 504 and 484a. Manual branch orchestration and AWS Step Functions sketches below are illustrative only.
* **Operator + GitOps:** Backlog [500-agent-operator.md](../500-agent-operator.md) now owns CRD/Deployment reconciliation, so any configuration described here must be expressed as AgentDefinitions + CR manifests instead of local scripts.
* **Reference implementation:** The LangGraph-based coder agent lives in `primitives/agent/ameide-coder/src/agent.py` and uses the `develop_in_container` tool (`primitives/agent/ameide-coder/src/tools/develop_in_container.py`). Use those directories as the canonical code reference.

The remainder of this document describes the original PoC setup for posterity.

## Architecture Note

This recipe predates the SOLID refactoring of core-agent-model (see 023-agent-review.md). 
When implementing, use the new architecture:
- `AgentModel` with proper node validators
- `AgentModelProto` for serialization
- SOLID edge types (SimpleEdge, ConditionalEdge, LoopEdge, ErrorEdge)
- ValidatorFactory for extensibility

## End‑to‑end recipe (quick‑reference)

| Layer                                 | What we settled on                                                                                                                                                                                                                        |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Agent architecture**                | LangGraph sub‑graphs for **Architect → Coder ↔ Tester ↔ Reviewer**, wrapped as tools and driven by a ReAct agent.                                                                                                                         |
| **Git flow**                          | *start\_branch* node → `agent/YYYYMMDD‑slug` off `main` → work happens → human/Reviewer **LGTM** → *merge\_branch* node commits & merges back into `main`, pushes, deletes temp branch.                                                   |
| **Human touch‑points**                | Approval nodes in LangGraph, Slack/GUI buttons, real‑time dashboard cards (+ timeline, diff viewer).                                                                                                                                      |
| **Dashboard UI**                      | React component (in Canvas) that streams `/events` via SSE; shows status, messages, diff pane, cost meter.                                                                                                                                |
| **Local run (bare metal)**            | `python agent_runner.py --goal "…"`, using host Git repo and CLI credentials in `~/.config/…`.                                                                                                                                            |
| **Local run (container)**             | `bash\ndocker run -v $PWD:/work \\\n           -v $SSH_AUTH_SOCK:/ssh-agent -e SSH_AUTH_SOCK \\\n           -v ~/.config/claude-code:/root/.config/claude-code:ro \\\n           --env-file .agent.env dev-agent:latest --goal \"...\"\n` |
| **Remote CI (GitHub Actions)**        | Same image; inject Claude cookie or API key with repo secrets, checkout with `fetch-depth:0`, run script, push merge.                                                                                                                     |
| **Claude usage**                      | *We stick to the **claude CLI** interactive flow* to stay on Max (no API billing). Cookie lives in `auth.json`; mount or recreate it from secret. For paid tasks, flip to API key path.                                                   |
| **Codex usage**                       | Standard `OPENAI_API_KEY` env var, always metered.                                                                                                                                                                                        |
| **Multiple Claude accounts per user** | Store creds under `~/.claude-accounts/<account>/…`, switch by wrapping CLI with `XDG_CONFIG_HOME=<path> claude …` (or helper `claude‑as`).                                                                                                |
| **Multiple OS users**                 | Isolation is automatic—each user has their own `$HOME` tree and thus their own Claude cookie store.                                                                                                                                       |
| **Cost & safety guards**              | Budget‑check node, automated token telemetry, quota alerts, Gitleaks, branch isolation, Docker sandbox, secret scanning.                                                                                                                  |

---

### End‑to‑end flow in one picture

```
start_branch   ─→  Architect  ─→  Coder  ─↔─  Tester  ─↔─  Reviewer  ─→  merge_branch ─→  end
                   (Codex)           ↑ tests fail  ↑ changes reqd             | push, delete temp
                                      └────────────┘                          |
                                 Human GUI buttons (approve / revise) ←───────┘
```

---

### One command cheat‑sheet

```bash
# 1. interactive Claude login once (Max quota)
claude login

# 2. build everything
docker build -t dev-agent .

# 3. run a new goal inside container on a temp git branch
docker run --rm -it \
  -v $(pwd):/work -w /work \
  -v $SSH_AUTH_SOCK:/ssh-agent -e SSH_AUTH_SOCK \
  -v ~/.config/claude-code:/root/.config/claude-code:ro \
  --env OPENAI_API_KEY --env ANTHROPIC_API_KEY= \
  dev-agent --goal "Add Redis caching layer"
```

---

### When you need multiple Claude plans

```bash
# login once per plan
XDG_CONFIG_HOME=~/.claude-accounts/max-personal claude login
XDG_CONFIG_HOME=~/.claude-accounts/api-paid     claude login   # or store API key

# run agent with a specific account
XDG_CONFIG_HOME=~/.claude-accounts/max-personal docker run …          # free quota
XDG_CONFIG_HOME=~/.claude-accounts/api-paid docker run -e ANTHROPIC_API_KEY=...  # metered
```

---

### What’s left to customise

* Flesh out **tester** (coverage, static analysis) and **reviewer** (risk score).
* Add **cost dashboard** widget (`usage $ / tokens`) to React UI.
* Automate **Claude cookie refresh** every 30 days (simple GitHub secret rotation action).
* Optional: second LLM reviewer or human gate for high‑risk merges.

With these pieces you can: develop locally, run in Docker or CI, branch safely, merge on approval, and keep Claude usage inside your Max subscription unless you explicitly opt into paid API calls.




{
  "Comment": "Agent‑driven coding workflows (branch‑per‑session) in AWS States Language / WfSpec style",
  "StartAt": "CreateSessionBranch",
  "States": {
    "CreateSessionBranch": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName.$": "$.meta.functions.create_session_branch",
        "Payload.$": "$"
      },
      "Next": "Architect"
    },

    "Architect": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName.$": "$.meta.functions.architect_llm",
        "Payload.$": "$"
      },
      "Next": "HumanApproveSpec"
    },

    "HumanApproveSpec": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.approval.spec",
          "BooleanEquals": true,
          "Next": "Coder"
        }
      ],
      "Default": "Architect"
    },

    "Coder": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName.$": "$.meta.functions.coder_codex",
        "Payload.$": "$"
      },
      "Next": "TestAndLint"
    },

    "TestAndLint": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "UnitTests",
          "States": {
            "UnitTests": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName.$": "$.meta.functions.run_unit_tests",
                "Payload.$": "$"
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "StaticAnalysis",
          "States": {
            "StaticAnalysis": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName.$": "$.meta.functions.run_static_analysis",
                "Payload.$": "$"
              },
              "End": true
            }
          }
        }
      ],
      "Next": "EvaluateTests"
    },

    "EvaluateTests": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.tests_pass",
          "BooleanEquals": true,
          "Next": "Reviewer"
        }
      ],
      "Default": "Coder"
    },

    "Reviewer": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName.$": "$.meta.functions.reviewer_claude",
        "Payload.$": "$"
      },
      "Next": "ComputeRiskScore"
    },

    "ComputeRiskScore": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName.$": "$.meta.functions.compute_risk",
        "Payload.$": "$"
      },
      "Next": "RiskGate"
    },

    "RiskGate": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.risk_score",
          "NumericGreaterThan": 70,
          "Next": "HumanCriticalReview"
        }
      ],
      "Default": "HumanApproveDiff"
    },

    "HumanCriticalReview": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName.$": "$.meta.functions.human_critical_review",
        "Payload.$": "$"
      },
      "Next": "HumanApproveDiff"
    },

    "HumanApproveDiff": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.approval.diff",
          "BooleanEquals": true,
          "Next": "CommitMerge"
        }
      ],
      "Default": "Coder"
    },

    "CommitMerge": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName.$": "$.meta.functions.commit_and_merge",
        "Payload.$": "$"
      },
      "Next": "Success"
    },

    "Success": {
      "Type": "Succeed"
    }
  }
}
