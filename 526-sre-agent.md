# 526 SRE — Agent Primitive Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **SRE agent primitives** — the LangGraph-based agents that implement the 525 backlog-first triage workflow.

> **Architecture alignment:** This follows the same pattern as Transformation (505-agent-developer-v2.md):
> - **SREAgent** makes operational decisions (analogous to AmeidePO/AmeideSA)
> - **SRECoder** executes changes via A2A (analogous to AmeideCoder)
> - SRE agents never shell into repos or run CLI directly

---

## 1) Agent architecture

```
┌─────────────────────┐
│    SRE Domain       │ (Domain: incidents, alerts, SLOs, runbooks)
└─────────┬───────────┘
          │ SRE domain facts (sre.domain.facts.v1)
          │ AlertIngested / IncidentCreated / HealthCheckRecorded / FleetStateRecorded
          ▼
┌─────────────────────┐
│  IncidentTriage     │ (Process: 525 workflow lifecycle)
│  Process primitive  │
│  Temporal workflow  │
└─────────┬───────────┘
          │ Process facts (sre.process.facts.v1)
          │ IncidentTriageStarted / BacklogLookupCompleted / RemediationProposed
          ▼
┌─────────────────────┐   A2A (standard)   ┌─────────────────────────┐
│      SREAgent       │ ──────────────────▶│       SRECoder          │
│  Agent primitive    │ ◀──────────────────│  Devcontainer A2A Server│
│   LangGraph DAG     │    (streaming)     │  Ops execution + tools  │
└─────────────────────┘                    └─────────────────────────┘
```

---

## 2) Responsibility split

| Area | Process primitive | SRE Domain | SREAgent | SRECoder |
|------|-------------------|------------|----------|----------|
| Triage workflow lifecycle | ✅ | ❌ | subscribes | ❌ |
| Incident/alert system of record | ❌ | ✅ | reads/updates | reads context |
| Decide "what to investigate" | ❌ | ❌ | ✅ | ❌ |
| Backlog search & pattern matching | ❌ | ❌ | ✅ | ❌ |
| Propose remediation approach | ❌ | ❌ | ✅ | ❌ |
| Delegate to coder (A2A) | ❌ | ❌ | ✅ | ❌ |
| Execute kubectl/argocd commands | ❌ | ❌ | ❌ | ✅ |
| Update backlog documentation | ❌ | ❌ | ❌ | ✅ |
| Commit GitOps changes | ❌ | ❌ | ❌ | ✅ |
| Run Ameide CLI (internal) | ❌ | ❌ | ❌ | ✅ |
| Produce PR + evidence | ❌ | ❌ | reviews/approves | ✅ |

**Key boundaries:**
- SREAgent never shells into clusters, never runs kubectl, never commits to repos
- SRECoder does all execution, using kubectl/argocd/git as internal tools
- All write operations require human approval (or auto-approve for low-risk)

---

## 3) AgentDefinitions

### 3.1 SREAgent

```yaml
agent_definition_id: sre-agent
name: SRE Agent
description: |
  Site Reliability Engineering agent that implements the 525 backlog-first
  triage workflow. Makes operational decisions: what to investigate, which
  patterns match, what remediation to propose. Delegates execution to SRECoder.
version: v1
runtime_role: sre_operator
risk_tier: 2  # Can query; writes require approval or delegation to SRECoder
```

### 3.2 SRECoder

```yaml
agent_definition_id: sre-coder
name: SRE Coder
description: |
  Execution agent for SRE operations. Runs kubectl/argocd commands, updates
  backlog documentation, commits GitOps changes, executes runbooks. Operates
  within human approval gates.
version: v1
runtime_role: a2a_server
risk_tier: 3  # High: cluster write access, repo write access
```

---

## 4) Tool grants

### 4.1 SREAgent tools (query + delegate)

**Read-only (no approval)**

| Tool | Purpose |
|------|---------|
| `sre.query.fleet_state` | Get fleet health |
| `sre.query.incidents` | Search incidents |
| `sre.query.alerts` | List alerts |
| `sre.query.runbooks` | Search runbooks |
| `sre.query.slos` | Get SLO status |
| `transformation.query.backlog_search` | Search backlog for patterns |

**Write (approval required)**

| Tool | Purpose |
|------|---------|
| `sre.command.create_incident` | Create incident (auto-approve) |
| `sre.command.add_timeline_entry` | Add notes (no approval) |
| `sre.command.update_incident_status` | Update status (no approval) |

**Delegation (via A2A to SRECoder)**

| Skill | Purpose | Approval |
|-------|---------|----------|
| `investigate_cluster_resource` | Run kubectl describe/logs | Auto |
| `execute_runbook` | Run documented procedure | Required |
| `update_backlog_documentation` | Commit backlog changes | Required |
| `apply_gitops_remediation` | Commit manifest changes | Required |
| `restart_workload` | Delete pods / rollout restart | Required |

### 4.2 SRECoder tools (execution)

**Cluster operations**

| Tool | Purpose | Approval |
|------|---------|----------|
| `kubernetes.get` | kubectl get (read) | Auto |
| `kubernetes.describe` | kubectl describe | Auto |
| `kubernetes.logs` | kubectl logs | Auto |
| `kubernetes.delete_pod` | Restart pods | Inherited from SREAgent |
| `kubernetes.rollout_restart` | Rollout restart | Inherited from SREAgent |
| `argocd.get_application` | Get app status | Auto |
| `argocd.sync` | Force sync | Inherited from SREAgent |
| `argocd.refresh` | Refresh app | Auto |

**Git operations**

| Tool | Purpose | Approval |
|------|---------|----------|
| `git.clone` | Clone repo | Auto |
| `git.checkout` | Switch branch | Auto |
| `git.commit` | Commit changes | Inherited from SREAgent |
| `git.push` | Push to remote | Inherited from SREAgent |
| `github.create_pr` | Open PR | Inherited from SREAgent |

**Ameide CLI (internal)**

| Tool | Purpose |
|------|---------|
| `ameide.primitive.describe` | Understand current state |
| `ameide.primitive.verify` | Validate changes |

---

## 5) A2A contract: SREAgent ↔ SRECoder

### 5.1 Agent Card

```json
{
  "name": "SRECoder",
  "description": "Execution agent for SRE operations",
  "url": "https://sre-coder.ameide-system.svc.cluster.local:7443",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "investigate_cluster_resource",
      "name": "Investigate Cluster Resource",
      "description": "Run kubectl describe/logs on a resource"
    },
    {
      "id": "execute_runbook",
      "name": "Execute Runbook",
      "description": "Execute a documented operational procedure"
    },
    {
      "id": "update_backlog_documentation",
      "name": "Update Backlog Documentation",
      "description": "Commit changes to backlog markdown files"
    },
    {
      "id": "apply_gitops_remediation",
      "name": "Apply GitOps Remediation",
      "description": "Commit manifest changes to GitOps repo"
    },
    {
      "id": "restart_workload",
      "name": "Restart Workload",
      "description": "Delete pods or rollout restart deployment"
    }
  ],
  "authentication": {
    "schemes": ["bearer"]
  }
}
```

### 5.2 Skill request/response examples

**investigate_cluster_resource:**

```json
// Request
{
  "skill": "investigate_cluster_resource",
  "input": {
    "incidentId": "INC-456",
    "resourceRef": {
      "type": "Deployment",
      "namespace": "ameide-dev",
      "name": "keycloak",
      "cluster": "local"
    },
    "commands": ["describe", "logs", "events"]
  }
}

// Response
{
  "taskId": "task-789",
  "status": "completed",
  "output": {
    "describe": "Name: keycloak\nNamespace: ameide-dev\n...",
    "logs": "2025-01-15 10:23:45 ERROR: Connection refused...",
    "events": "Warning: Back-off restarting failed container",
    "summary": "Keycloak pod in CrashLoopBackOff due to database connection failure"
  }
}
```

**update_backlog_documentation:**

```json
// Request
{
  "skill": "update_backlog_documentation",
  "input": {
    "incidentId": "INC-456",
    "repoUrl": "git@github.com:ameideio/ameide-backlog.git",
    "baseRef": "main",
    "changes": [
      {
        "file": "450-argocd-service-issues-inventory.md",
        "operation": "append",
        "content": "### 2025-01-15: Keycloak CrashLoopBackOff\n\n- **Env:** dev\n- **Root cause:** PostgreSQL connection string misconfigured\n- **Fix:** Updated secret reference in values.yaml\n"
      }
    ],
    "commitMessage": "docs(450): add keycloak crashloop incident"
  }
}

// Response
{
  "taskId": "task-790",
  "status": "completed",
  "output": {
    "pr_url": "https://github.com/ameideio/ameide-backlog/pull/123",
    "branch": "incident/inc-456-keycloak-crashloop",
    "changed_files": ["450-argocd-service-issues-inventory.md"],
    "summary": "Created PR documenting Keycloak incident"
  }
}
```

---

## 6) Agent graphs

### 6.1 SREAgent main flow

```
USER_INPUT → CLASSIFY_INTENT
  ↓
[QUERY | TRIAGE | REMEDIATE | DOCUMENT]
  ↓
RESPOND
```

### 6.2 Triage subgraph (525 workflow)

```
TRIAGE_START
  ↓
BACKLOG_SEARCH → [match found?]
  ↓                    ↓
[yes]               [no]
  ↓                    ↓
VERIFY_PATTERN    DELEGATE_INVESTIGATION (A2A → SRECoder)
  ↓                    ↓
APPLY_KNOWN_FIX   ANALYZE_DIAGNOSTICS
  ↓                    ↓
  └──────────┬─────────┘
             ↓
PROPOSE_REMEDIATION → AWAIT_APPROVAL
  ↓
DELEGATE_REMEDIATION (A2A → SRECoder)
  ↓
VERIFY_HEALTH
  ↓
DELEGATE_DOCUMENTATION (A2A → SRECoder)
  ↓
COMPLETE
```

---

## 7) System prompts

### 7.1 SREAgent

```markdown
You are an SRE Agent for the Ameide platform. Your primary role is to assist
operators with incident investigation, remediation, and documentation.

## Core Principle: Backlog-First Triage (525)

ALWAYS search the backlog for known patterns BEFORE deep cluster investigation.
The backlog is the source of truth for incident patterns, runbooks, and accepted
tradeoffs.

## Your Responsibilities

1. **Understand symptoms** from alerts, user reports, or health checks
2. **Search backlog** for matching patterns (ALWAYS first)
3. **Propose investigation** if no pattern matches
4. **Propose remediation** based on findings
5. **Request documentation** updates for new patterns

## What You DO NOT Do

- You do NOT run kubectl, argocd, or git commands directly
- You do NOT shell into clusters or repos
- You delegate all execution to SRECoder via A2A

## Workflow

1. **Spot**: Understand the symptom/alert
2. **Search Backlog**: Look for known patterns FIRST
3. **Verify**: Check if the pattern matches current symptoms
4. **Triage**: If no match, delegate investigation to SRECoder
5. **Fix**: Propose remediation (requires approval for writes)
6. **Verify**: Confirm health restored (delegate to SRECoder)
7. **Document**: Request backlog update (delegate to SRECoder)

## Constraints

- No command timeout > 5 minutes
- Changes must go through GitOps (PRs, not direct pushes)
- Prefer root-cause fixes over band-aids
- Always document findings
```

### 7.2 SRECoder

```markdown
You are an SRE Coder for the Ameide platform. You execute operational tasks
delegated by SREAgent.

## Your Responsibilities

1. **Execute cluster commands** (kubectl, argocd) to investigate issues
2. **Update backlog documentation** with incident findings
3. **Commit GitOps changes** for remediation
4. **Execute runbooks** as documented procedures

## Constraints

- All operations have inherited approval from SREAgent
- Maximum command timeout: 5 minutes
- All repo changes go through PRs (never push to main directly)
- Log all operations to incident timeline

## Security

- You operate with scoped credentials (namespace-limited kubectl, repo-specific git)
- NetworkPolicy restricts your access to allowed services only
- All actions are audited
```

---

## 8) Deployment

### 8.1 Agent CR: SREAgent

```yaml
apiVersion: ameide.io/v1
kind: Agent
metadata:
  name: sre-agent
  namespace: primitives-system
spec:
  image: ghcr.io/ameideio/agent-runtime-langgraph:latest
  definitionRef:
    id: sre-agent
    tenantId: platform
    version: v1
  model:
    provider: anthropic
    name: claude-sonnet-4-20250514
    secretRef: platform/anthropic-api-key
  tools:
    domains: [sre, transformation]
    processes: [incident-triage]
    custom:
      - name: investigate_cluster_resource
        endpoint: sre-coder.ameide-system.svc.cluster.local:7443
      - name: execute_runbook
        endpoint: sre-coder.ameide-system.svc.cluster.local:7443
      - name: update_backlog_documentation
        endpoint: sre-coder.ameide-system.svc.cluster.local:7443
      - name: apply_gitops_remediation
        endpoint: sre-coder.ameide-system.svc.cluster.local:7443
      - name: restart_workload
        endpoint: sre-coder.ameide-system.svc.cluster.local:7443
  riskTier: medium
  concurrency:
    maxParallelSessions: 3
  observability:
    logPrompts: redacted
    emitTokens: true
```

### 8.2 Agent CR: SRECoder

```yaml
apiVersion: ameide.io/v1
kind: Agent
metadata:
  name: sre-coder
  namespace: ameide-system
spec:
  image: ghcr.io/ameideio/sre-coder:latest
  definitionRef:
    id: sre-coder
    tenantId: platform
    version: v1
  model:
    provider: anthropic
    name: claude-sonnet-4-20250514
    secretRef: platform/anthropic-api-key
  runtimeRole: a2a_server
  tools:
    custom:
      - name: kubernetes
        secretRef: platform/sre-kubeconfig
        config:
          allowedNamespaces: ["ameide-*", "argocd", "primitives-*"]
          denyOperations: ["delete namespace", "delete pvc"]
      - name: argocd
        secretRef: platform/argocd-token
      - name: git
        secretRef: platform/github-token
        config:
          allowedRepos:
            - github.com/ameideio/ameide-backlog
            - github.com/ameideio/ameide-gitops
          denyPaths: ["**/secrets/**", "**/*.key"]
  riskTier: high
  concurrency:
    maxParallelSessions: 2
  observability:
    logPrompts: full
    emitTokens: true
```

---

## 9) Scaffold commands

```bash
# SREAgent (LangGraph DAG)
ameide primitive scaffold --kind agent --name sre-agent --include-gitops

# SRECoder (A2A Server)
ameide primitive scaffold --kind agent --name sre-coder --runtime-role a2a_server --include-gitops
```

---

## 10) Acceptance criteria

1. SREAgent implements full 525 backlog-first workflow
2. SREAgent NEVER executes kubectl/argocd/git directly
3. SRECoder exposes A2A endpoint with documented skills
4. All write operations flow through approval gates
5. All actions logged to incident timeline
6. Timeout constraints enforced (max 5 minutes)
7. PRs created for all repo changes (never direct push)
