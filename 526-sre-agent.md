# 526 SRE — Agent Primitive Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **sre-agent** primitive — the LangGraph-based agent that implements the 525 backlog-first triage workflow.

---

## 1) Agent responsibilities

The `sre-agent` primitive implements **SRE operational assistance**:

- Execute the 525 backlog-first triage methodology
- Search backlog for known incident patterns
- Propose remediation actions with human approval
- Execute runbooks (assisted or automated)
- Document incidents and update backlog
- Answer operational questions about fleet health

Agent primitives:

- Are **consumers** of facts and read models
- Are **producers** of intents/commands (gated by approval)
- Execute **AgentDefinitions** stored in Transformation domain
- Are governed by **tool grants** and **risk-tier** policies

---

## 2) AgentDefinition: SREAgent

```yaml
agent_definition_id: sre-agent
name: SRE Agent
description: |
  Site Reliability Engineering agent that implements the 525 backlog-first
  triage workflow. Assists operators with incident investigation, remediation,
  and documentation.
version: v1
risk_tier: 2  # Can execute read operations; writes require approval
```

---

## 3) Tool grants

### Read-only (no approval)

| Tool | Purpose |
|------|---------|
| `sre.query.fleet_state` | Get fleet health |
| `sre.query.incidents` | Search incidents |
| `sre.query.alerts` | List alerts |
| `sre.query.runbooks` | Search runbooks |
| `transformation.query.backlog_search` | Search backlog |
| `kubernetes.get` | kubectl get resources |
| `kubernetes.logs` | Get pod logs |
| `argocd.get_application` | Get app details |

### Write (approval required)

| Tool | Purpose |
|------|---------|
| `sre.command.create_incident` | Create incident (auto-approve) |
| `sre.command.add_timeline_entry` | Add notes (no approval) |
| `sre.command.execute_runbook` | Run runbook (approval required) |
| `transformation.command.create_backlog_item` | Create backlog (approval required) |
| `gitops.commit` | Commit changes (approval required) |
| `kubernetes.delete_pod` | Restart pods (approval required) |

---

## 4) Agent graph

### Main flow

```
USER_INPUT → CLASSIFY_INTENT → [QUERY | TRIAGE | REMEDIATE] → RESPOND
```

### Triage subgraph (525 workflow)

```
TRIAGE_START → BACKLOG_SEARCH → ANALYZE_RESULTS
  ↓
[KNOWN_PATTERN | GATHER_DIAGNOSTICS]
  ↓
IDENTIFY_ROOT_CAUSE → PROPOSE_REMEDIATION → AWAIT_APPROVAL
  ↓
APPLY_REMEDIATION → VERIFY_HEALTH → DOCUMENT
```

---

## 5) System prompt

```markdown
You are an SRE Agent for the Ameide platform. Your primary role is to assist
operators with incident investigation, remediation, and documentation.

## Core Principle: Backlog-First Triage (525)

ALWAYS search the backlog for known patterns BEFORE deep cluster investigation.

## Workflow

1. **Spot**: Understand the symptom/alert
2. **Search Backlog**: Look for known patterns FIRST
3. **Verify**: Check if the pattern matches
4. **Triage**: If no match, gather targeted diagnostics
5. **Fix**: Propose remediation (requires approval for writes)
6. **Verify**: Confirm health restored
7. **Document**: Update backlog with findings

## Constraints

- No command timeout > 5 minutes
- Changes must go through GitOps
- Prefer root-cause fixes over band-aids
- Always document findings
```

---

## 6) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind agent --name sre --include-gitops
```

### Approval handling

Write operations emit approval requests and wait for signals.

---

## 7) Acceptance criteria

1. Agent implements full 525 backlog-first workflow
2. Backlog search is ALWAYS the first step in triage
3. Write operations require human approval
4. All actions logged to incident timeline
5. Timeout constraints enforced
