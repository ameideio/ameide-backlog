# Governance Entity UI Extensions - Artifact Lifecycle Integration

## Overview

This document extends [220-governance-entity-model-ui.md](220-governance-entity-model-ui.md) with governance screens that surface artifact lifecycle context, stewardship controls, retention policies, scenario metadata, trace link visibility, and auto-classification oversight. These extensions close critical gaps where artifact-side features exist ([220-artifact-entity-model-ui.md](220-artifact-entity-model-ui.md)) but governance teams lack visibility or management controls.

## Navigation Context
```
┌─────────────────────────────────────────────────────────────────────┐
│ ameide > Governance                                                  │
│ ┌──────────────────────────────────────────────────────────────────┐
│ │ [Reviews] [Policies] [Checks] [Waivers] [Compliance] [Analytics] │
│ └──────────────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Review Case Detail - Extended Context

**Purpose**: Adds stewardship, retention, scenario, and trace link context to review case detail screen to give reviewers complete lifecycle visibility during approval workflows.

**Extends**: [220-governance-entity-model-ui.md#2-review-case-detail](220-governance-entity-model-ui.md#L75-L152)

```
╔══════════════════════════════════════════════════════════════════════╗
║ Review Case: 2025 Digital Transformation Vision v1.1    [Actions ▾]  ║
╠══════════════════════════════════════════════════════════════════════╣
║  [Overview] [Checks] [Decisions] [Context] [Evidence] [Activity]    ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  📋 CONTEXT TAB - NEW                                                 ║
║                                                                       ║
║  STEWARDSHIP & GOVERNANCE OWNERSHIP                                   ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Artifact Owner: Alice (Enterprise Architect)                       │ ║
║  │ Stewardship Group: [Enterprise Architecture Board]             │ ║
║  │                                                                 │ ║
║  │ Current Governance Owner Group: EA Board                        │ ║
║  │ ├─ Lead: David (Governance Lead)                                │ ║
║  │ ├─ Reviewers: Bob, Bob2, Carol (3)                              │ ║
║  │ ├─ Delegates: Eve (on-call backup)                              │ ║
║  │ │                                                                │ ║
║  │ │ Quorum Rules:                                                 │ ║
║  │ │ • Required Approvals: 2/3 reviewers                           │ ║
║  │ │ • Abstain Handling: Count toward quorum                       │ ║
║  │ │ • Tie-Breaking: Lead casts deciding vote                      │ ║
║  │ │                                                                │ ║
║  │ │ Escalation Chain:                                             │ ║
║  │ │ • Level 1 (24h): Primary reviewers (Bob, Bob2, Carol)         │ ║
║  │ │ • Level 2 (48h): Governance Lead (David)                      │ ║
║  │ │ • Level 3 (72h): Fallback delegates (Eve)                     │ ║
║  │                                                                 │ ║
║  │ [View Group Details] [View Escalation History]                 │ ║
║  │                                                                 │ ║
║  │ ⚠ Note: Governance defaults were inherited from graph     │ ║
║  │   settings. Artifact-level overrides: None                         │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  RETENTION & LIFECYCLE CONTEXT                                        ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Retention State: [● Active]                                     │ ║
║  │ Retention Policy: Repository Default (7 years after retirement) │ ║
║  │ Expiry Date: N/A (active artifact, no expiry)                      │ ║
║  │                                                                 │ ║
║  │ Archival Policy: Azure Archive after 90 days of Retired state  │ ║
║  │ Retention Hold: ☐ Not enabled                                   │ ║
║  │                                                                 │ ║
║  │ ℹ Reviewer Guidance:                                            │ ║
║  │ • Approve only if retention requirements are understood         │ ║
║  │ • Flag if artifact should be exempt from auto-archival             │ ║
║  │ • Request retention hold if regulatory freeze needed            │ ║
║  │                                                                 │ ║
║  │ [View Repository Retention Policy] [Request Retention Hold]    │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  SCENARIO CONTEXT                                                     ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Scenario Tag: [🏛 Baseline]                                     │ ║
║  │ Effective Window: Jan 15, 2025 → Always Active                  │ ║
║  │                                                                 │ ║
║  │ ℹ This artifact represents the current baseline architecture.      │ ║
║  │   Target scenario artifacts (future state) have separate review   │ ║
║  │   workflows and may reference or supersede this baseline.       │ ║
║  │                                                                 │ ║
║  │ Related Scenario Artifacts:                                        │ ║
║  │ • Target: 2025 Target Architecture Vision v2.0 (Draft)          │ ║
║  │ • Transition: Migration Roadmap v1.3 (In Review)                │ ║
║  │                                                                 │ ║
║  │ [View Scenario Toggle Documentation]                           │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  TRACE LINK COVERAGE & DOWNSTREAM IMPACT                              ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Trace Policy Compliance: [✓ Compliant]                          │ ║
║  │                                                                 │ ║
║  │ Required Coverage:                                              │ ║
║  │ ✓ All deliverables must satisfy ≥1 requirement (2 satisfied)   │ ║
║  │ ✓ Strategic artifacts must reference ≥2 standards (3 referenced)  │ ║
║  │                                                                 │ ║
║  │ Trace Links Summary: (8 total)                                 │ ║
║  │ ┌────────────────────────────────────────────────────────────┐ │ ║
║  │ │ SATISFIES (2)                                              │ │ ║
║  │ │ → REQ-001: Transform customer experience                   │ │ ║
║  │ │ → REQ-005: Modernize digital capabilities                  │ │ ║
║  │ │                                                            │ │ ║
║  │ │ REFERENCES (3 Standards)                                   │ │ ║
║  │ │ → Cloud-First Principle                                    │ │ ║
║  │ │ → Data Governance Standard v2.1                            │ │ ║
║  │ │ → Security Architecture Standard                           │ │ ║
║  │ │                                                            │ │ ║
║  │ │ BUILDS ON (1)                                              │ │ ║
║  │ │ → Enterprise Capability Model v3.0                         │ │ ║
║  │ │                                                            │ │ ║
║  │ │ IMPACTS (2 Capabilities)                                   │ │ ║
║  │ │ → Customer Experience (Business Capability)                │ │ ║
║  │ │ → Digital Platform (Business Capability)                   │ │ ║
║  │ └────────────────────────────────────────────────────────────┘ │ ║
║  │                                                                 │ ║
║  │ ⚠ Downstream Impact Analysis:                                  │ ║
║  │ Initiatives Consuming This Artifact: 2                             │ ║
║  │ ├─ Modernization Pilot (Baseline Reference)                     │ ║
║  │ └─ Customer Experience Platform (Mandated Standard)             │ ║
║  │                                                                 │ ║
║  │ ⚠ If this review is REJECTED, 2 transformations will be blocked    │ ║
║  │                                                                 │ ║
║  │ [View Full Trace Graph] [View Initiative Impact] [Export]     │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  CLASSIFICATION CONTEXT                                               ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Current Classifications: (2)                                    │ ║
║  │ ├─ 📚 Landscape.Baseline (Manual, Enforced Rule)                │ ║
║  │ └─ 🎯 Architecture.Vision (Auto-Suggested, Accepted)            │ ║
║  │                                                                 │ ║
║  │ Auto-Classification Status:                                     │ ║
║  │ ✓ All suggestions reviewed by artifact owner                       │ ║
║  │ ✓ No pending auto-classification exceptions                     │ ║
║  │                                                                 │ ║
║  │ [View Classification Details]                                  │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 2. Stewardship & Owner Group Administration

**Purpose**: Central management interface for governance owner groups, membership rosters, escalation chains, and quorum rules. Addresses the gap where owner groups appear as static labels without management controls.

**New Screen** (referenced from Repository Settings or Governance nav)

```
╔══════════════════════════════════════════════════════════════════════╗
║ Governance Owner Groups                           [+ New Group]      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ACTIVE GROUPS (5)                                                    ║
║                                                                       ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 👥 Enterprise Architecture Board            [● Active]          │ ║
║  │ ──────────────────────────────────────────────────────────────  │ ║
║  │ Scope: Repository-wide governance                               │ ║
║  │ Default for Classifications: (12) including Landscape.*         │ ║
║  │ Active Review Cases: 3 │ P95 Cycle Time: 42h                    │ ║
║  │                                                                 │ ║
║  │ MEMBERSHIP ROSTER                                               │ ║
║  │ ┌────────────────────────────────────────────────────────────┐ │ ║
║  │ │ Lead (1)                                                   │ │ ║
║  │ │ • David (Governance Lead)                   [Edit] [Remove]│ │ ║
║  │ │   Joined: Jan 1, 2024 │ Active cases: 1                    │ │ ║
║  │ │   Repository Role: Admin (auto-granted)                    │ │ ║
║  │ │                                                            │ │ ║
║  │ │ Reviewers (3)                                              │ │ ║
║  │ │ • Bob (Senior Architect)                    [Edit] [Remove]│ │ ║
║  │ │   Joined: Jan 1, 2024 │ Active cases: 2                    │ │ ║
║  │ │   Repository Role: Contributor (auto-granted)              │ │ ║
║  │ │ • Bob2 (Principal Architect)                [Edit] [Remove]│ │ ║
║  │ │   Joined: Jan 1, 2024 │ Active cases: 1                    │ │ ║
║  │ │ • Carol (Solutions Architect)               [Edit] [Remove]│ │ ║
║  │ │   Joined: Feb 15, 2024 │ Active cases: 0                   │ │ ║
║  │ │                                                            │ │ ║
║  │ │ Delegates (1)                                              │ │ ║
║  │ │ • Eve (On-Call Backup)                      [Edit] [Remove]│ │ ║
║  │ │   Joined: Mar 1, 2024 │ Escalation role only               │ │ ║
║  │ │                                                            │ │ ║
║  │ │ [+ Add Member]                                             │ │ ║
║  │ └────────────────────────────────────────────────────────────┘ │ ║
║  │                                                                 │ ║
║  │ QUORUM RULES                                                    │ ║
║  │ ┌────────────────────────────────────────────────────────────┐ │ ║
║  │ │ Required Approvals: [2 ▾] of [3] reviewers                │ │ ║
║  │ │ Abstention Handling: ⦿ Count toward quorum                 │ │ ║
║  │ │                      ○ Do not count (block if too many)    │ │ ║
║  │ │ Tie-Breaking: ⦿ Lead casts deciding vote                   │ │ ║
║  │ │               ○ Escalate to next level                     │ │ ║
║  │ │                                                            │ │ ║
║  │ │ [Test Quorum Scenarios]                                    │ │ ║
║  │ └────────────────────────────────────────────────────────────┘ │ ║
║  │                                                                 │ ║
║  │ ESCALATION CHAIN (3 Levels)                                     │ ║
║  │ ┌────────────────────────────────────────────────────────────┐ │ ║
║  │ │ Level 1: Primary Reviewers           Trigger: 0h (immediate)│ │ ║
║  │ │ • Notify: Bob, Bob2, Carol                                 │ │ ║
║  │ │ • Action: Assign case, send review request                 │ │ ║
║  │ │                                                            │ │ ║
║  │ │ Level 2: Governance Lead             Trigger: 24h overdue  │ │ ║
║  │ │ • Notify: David (Lead)                                     │ │ ║
║  │ │ • Action: Send escalation alert, request prioritization    │ │ ║
║  │ │                                                            │ │ ║
║  │ │ Level 3: Fallback Delegates          Trigger: 48h overdue  │ │ ║
║  │ │ • Notify: Eve (Delegate)                                   │ │ ║
║  │ │ • Action: Auto-assign to fallback, create SLA breach event │ │ ║
║  │ │                                                            │ │ ║
║  │ │ [Edit Chain] [Test Escalation]                             │ │ ║
║  │ └────────────────────────────────────────────────────────────┘ │ ║
║  │                                                                 │ ║
║  │ REPOSITORY ACCESS AUTO-GRANT                                    │ ║
║  │ ┌────────────────────────────────────────────────────────────┐ │ ║
║  │ │ ☑ Auto-grant graph access to members                  │ │ ║
║  │ │   Lead Role:     [Admin ▾]                                 │ │ ║
║  │ │   Reviewer Role: [Contributor ▾]                           │ │ ║
║  │ │   Delegate Role: [Viewer ▾]                                │ │ ║
║  │ │                                                            │ │ ║
║  │ │ Current Grants: 5 active (tracked in Membership table)     │ │ ║
║  │ └────────────────────────────────────────────────────────────┘ │ ║
║  │                                                                 │ ║
║  │ [View Details] [Edit Group] [View Active Cases] [Analytics]   │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 👥 Standards Governance Team                 [● Active]          │ ║
║  │ ──────────────────────────────────────────────────────────────  │ ║
║  │ Scope: Standards recertification & compliance                  │ ║
║  │ Default for Classifications: (3) including SIB.Standards       │ ║
║  │ Active Review Cases: 1 │ P95 Cycle Time: 28h                    │ ║
║  │                                                                 │ ║
║  │ Members: 4 (2 Leads, 2 Reviewers) │ Quorum: 2/2                 │ ║
║  │ [View Details] [Edit]                                          │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 👥 Security & Privacy Council               [● Active]          │ ║
║  │ Scope: Security, privacy, compliance policies                  │ ║
║  │ Default for: Security & Privacy Policy v1.1                    │ ║
║  │ Members: 6 │ Quorum: 3/5 │ Active Cases: 2                      │ ║
║  │ [View Details] [Edit]                                          │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 3. Auto-Classification Exception Queue

**Purpose**: Governance oversight for auto-classification suggestions, exceptions, and validation loops. Allows reviewers to audit automated placement decisions and flag high-confidence suggestions that need manual review before governance approval.

**New Screen** (accessible from Governance Analytics or Checks tab)

```
╔══════════════════════════════════════════════════════════════════════╗
║ Auto-Classification Exception Queue              [Configure Rules]   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  Filters: [All Confidence ▾] [All Actions ▾] [All Classifications ▾] ║
║                                                                       ║
║  EXCEPTIONS REQUIRING REVIEW (8)                                      ║
║                                                                       ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 🤖 API Gateway Blueprint v2.1                  [⚠ High Priority]│ ║
║  │ ──────────────────────────────────────────────────────────────  │ ║
║  │ Suggested Classification: Landscape.Target                      │ ║
║  │ Confidence: 91% │ Reason: Keywords match + architecture pattern │ ║
║  │ Exception Reason: [⚠ High-confidence baseline override]         │ ║
║  │                                                                 │ ║
║  │ Current State: Pending artifact owner review                       │ ║
║  │ Owner Action: Alice has not yet accepted/rejected suggestion    │ ║
║  │ Classification Mode: Suggested (graph default)             │ ║
║  │                                                                 │ ║
║  │ ⚠ Governance Alert:                                             │ ║
║  │ This suggestion would OVERRIDE existing Landscape.Baseline tag. │ ║
║  │ Review required before accepting to avoid governance conflicts. │ ║
║  │                                                                 │ ║
║  │ Trace Impact: 2 transformations consuming this artifact as baseline    │ ║
║  │                                                                 │ ║
║  │ [View Artifact] [View Suggestion Details] [Flag for Manual Review]│ ║
║  │ [Approve Suggestion] [Reject Suggestion] [Request Owner Review]│ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 🤖 Customer Data Model v3.2                   [○ Low Priority] │ ║
║  │ ──────────────────────────────────────────────────────────────  │ ║
║  │ Suggested Classification: Reference.DataModel                   │ ║
║  │ Confidence: 67% │ Reason: Document structure analysis           │ ║
║  │ Exception Reason: [○ Below confidence threshold (75%)]          │ ║
║  │                                                                 │ ║
║  │ Current State: Auto-queued for manual review                    │ ║
║  │ Owner: Carol │ Assigned: 2 hours ago                            │ ║
║  │                                                                 │ ║
║  │ [View Artifact] [View Suggestion] [Escalate to Governance]        │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  RECENT APPROVALS & REJECTIONS (LAST 7 DAYS)                          ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Approved: 34 suggestions │ Avg Confidence: 88%                  │ ║
║  │ Rejected: 5 suggestions  │ Avg Confidence: 72%                  │ ║
║  │ Pending Owner Review: 8                                         │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  AUTO-CLASSIFICATION GOVERNANCE RULES                                 ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Confidence Threshold: [75% ▾]                                   │ ║
║  │ Below threshold → Queue for manual review                       │ ║
║  │                                                                 │ ║
║  │ Exception Triggers:                                             │ ║
║  │ ☑ Suggestion overrides existing baseline classification         │ ║
║  │ ☑ Suggestion changes governance owner group                     │ ║
║  │ ☑ Artifact has active governance case                              │ ║
║  │ ☑ Classification has enforcement-level placement rules          │ ║
║  │                                                                 │ ║
║  │ Learning Mode: ☑ Update confidence model from user corrections  │ ║
║  │                                                                 │ ║
║  │ [Edit Rules] [View Training Data] [Export Exception Log]       │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 4. Governance Analytics - Extended with Trace Coverage & Classification Metrics

**Purpose**: Extends [220-governance-entity-model-ui.md#8-governance-analytics-dashboard](220-governance-entity-model-ui.md#L478-L533) with trace coverage metrics and auto-classification performance to give governance teams visibility into impact-sensitive approvals and automated placement quality.

**Extends**: Governance Analytics Dashboard

```
╔══════════════════════════════════════════════════════════════════════╗
║ Governance Analytics                          [Export] [Date Range▾] ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  [Key Metrics] [Cycle Time] [Outcomes] [Trace Coverage] [Auto-Class]║
║                                                                       ║
║  📊 TRACE COVERAGE TAB - NEW                                          ║
║                                                                       ║
║  TRACE POLICY COMPLIANCE (LAST 30 DAYS)                               ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Total Reviews: 287                                              │ ║
║  │ Trace Policy Violations: 12 (4.2%)           [⚡ Action Needed] │ ║
║  │                                                                 │ ║
║  │ Violation Breakdown:                                            │ ║
║  │ • Missing requirement links: 7 cases                            │ ║
║  │ • Insufficient standard references: 4 cases                     │ ║
║  │ • Orphaned impact links: 1 case                                 │ ║
║  │                                                                 │ ║
║  │ Resolution Status:                                              │ ║
║  │ • Resolved via waiver: 3 cases                                  │ ║
║  │ • Resolved via remediation: 5 cases                             │ ║
║  │ • Pending: 4 cases (in rework)                                  │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  TRACE COVERAGE BY CLASSIFICATION                                     ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Classification            │ Avg Links │ Policy Compliance      │ ║
║  │ ─────────────────────────────────────────────────────────────  │ ║
║  │ Landscape.Baseline        │    8.2    │ 98.3% [✓ Excellent]   │ ║
║  │ Landscape.Target          │    6.7    │ 94.1% [✓ Good]        │ ║
║  │ Architecture.Vision       │   12.4    │ 99.2% [✓ Excellent]   │ ║
║  │ SIB.Standards             │    4.1    │ 87.3% [○ Fair]        │ ║
║  │ Reference.DataModel       │    5.8    │ 91.7% [✓ Good]        │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  HIGH-IMPACT REVIEWS (DOWNSTREAM CONSUMPTION)                         ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ ⚠ These reviews affect multiple transformations. Prioritize.       │ ║
║  │                                                                 │ ║
║  │ 📄 Cloud Security Standard v3.1          [◐ In Review]         │ ║
║  │    Downstream Impact: 8 transformations consuming (Mandated Std)    │ ║
║  │    Trace Links: 14 (satisfies 6 requirements)                   │ ║
║  │    SLA Status: 36h remaining (on track)                         │ ║
║  │    [View Case]                                                  │ ║
║  │                                                                 │ ║
║  │ 📄 Enterprise Capability Model v3.1      [◐ In Review]         │ ║
║  │    Downstream Impact: 12 transformations (Baseline Reference)       │ ║
║  │    Trace Links: 23 (builds foundation for 18 artifacts)            │ ║
║  │    SLA Status: [⚠ At Risk] 8h remaining                         │ ║
║  │    [View Case] [Escalate]                                       │ ║
║  │                                                                 │ ║
║  │ [View All High-Impact Reviews]                                 │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  📊 AUTO-CLASSIFICATION TAB - NEW                                     ║
║                                                                       ║
║  AUTO-CLASSIFICATION PERFORMANCE (LAST 30 DAYS)                       ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Suggestions Generated:  127                                     │ ║
║  │ Accepted by Owners:      89 (70.1%)                             │ ║
║  │ Rejected by Owners:      15 (11.8%)                             │ ║
║  │ Pending Review:          23 (18.1%)                             │ ║
║  │                                                                 │ ║
║  │ Avg Confidence: 84.3%                                           │ ║
║  │ Acceptance Rate by Confidence:                                  │ ║
║  │ • 90-100%: 95.2% acceptance (42 suggestions)                    │ ║
║  │ • 80-89%:  78.4% acceptance (51 suggestions)                    │ ║
║  │ • 75-79%:  52.1% acceptance (23 suggestions)                    │ ║
║  │ • Below 75%: Queued for manual review (11 suggestions)          │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  GOVERNANCE EXCEPTION QUEUE TRENDS                                    ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │  12   ┤                                                         │ ║
║  │       │                                                         │ ║
║  │  10   ┤     ●                                                   │ ║
║  │       │                 ●                                       │ ║
║  │   8   ┤ ●       ●           ●───●                               │ ║
║  │       │                             ●───●                       │ ║
║  │   6   ┤                                     ●                   │ ║
║  │       │                                         ●               │ ║
║  │   4   ●─┴───┴───┴───┴───┴───┴───┴───┴──>                       │ ║
║  │       W1  W2  W3  W4  W5  W6  W7  W8  W9                       │ ║
║  │                                                                 │ ║
║  │ Exceptions Pending Review: Trending down (Good!)                │ ║
║  │ Learning mode improvements reducing queue backlog.              │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  TOP EXCEPTION TRIGGERS (LAST 30 DAYS)                                ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Baseline override warnings:      18  █████████░░░░░░░░         │ ║
║  │ Confidence below threshold:      11  ██████░░░░░░░░░░░         │ ║
║  │ Governance owner group change:    7  ███░░░░░░░░░░░░░░         │ ║
║  │ Active case conflict:             3  █░░░░░░░░░░░░░░░░         │ ║
║  │ Enforcement-level rule conflict:  2  █░░░░░░░░░░░░░░░░         │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  CLASSIFICATION QUALITY METRICS                                       ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Artifact Misclassification Rate: 2.1% (Target: < 5%)  [✓ Healthy] │ ║
║  │ • Detected via governance review rejections                     │ ║
║  │ • Corrected within avg 18 hours                                 │ ║
║  │                                                                 │ ║
║  │ Auto-Suggestion Correction Rate: 11.8%                          │ ║
║  │ • Owners rejecting suggestions (improves learning model)        │ ║
║  │ • Confidence model updated weekly from feedback                 │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 5. Retention Policy Enforcement Dashboard

**Purpose**: Governance workspace for monitoring retention state, expiry dates, archival triggers, and retention holds across artifacts. Allows reviewers to enforce purge and hold rules during governance cases.

**New Screen** (accessible from Governance > Compliance or Analytics)

```
╔══════════════════════════════════════════════════════════════════════╗
║ Retention & Archival Enforcement                 [Configure Policy]  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  [Active Artifacts] [Pending Expiry] [Retention Holds] [Archive Queue] ║
║                                                                       ║
║  PENDING EXPIRY & ARCHIVAL (NEXT 90 DAYS)                             ║
║                                                                       ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Artifacts Approaching Retention Expiry: 12                         │ ║
║  │ Artifacts in Retired State (90+ days): 5                            │ ║
║  │ Retention Holds Active: 3                                        │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  ASSETS PENDING ARCHIVAL (5)                                          ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 📄 Legacy Payment Gateway Model v1.0        [○ Retired 94 days]│ ║
║  │ ──────────────────────────────────────────────────────────────  │ ║
║  │ Lifecycle State: Retired                                        │ ║
║  │ Retired Date: Oct 18, 2024 (94 days ago)                        │ ║
║  │ Archival Threshold: 90 days (EXCEEDED by 4 days)                │ ║
║  │                                                                 │ ║
║  │ Retention Policy: Repository Default (7 years after retirement) │ ║
║  │ Expiry Date: Oct 18, 2031 (6 years remaining)                   │ ║
║  │ Archival Target: Azure Archive (cold storage)                   │ ║
║  │                                                                 │ ║
║  │ Downstream Impact Check:                                        │ ║
║  │ ✓ No active transformations consuming this artifact                    │ ║
║  │ ✓ Superseded by: Payment Gateway v2.0 (Approved)                │ ║
║  │ ✓ Trace links archived                                          │ ║
║  │                                                                 │ ║
║  │ ⚠ Action Required:                                              │ ║
║  │ Artifact eligible for cold storage archival. Approve transfer?     │ ║
║  │                                                                 │ ║
║  │ [Approve Archival] [Extend Retention] [Request Hold] [Purge]   │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 📄 2020 Digital Strategy v3.2                [○ Retired 102d]  │ ║
║  │ Retired: Jul 10, 2024 (102 days) │ Expiry: Jul 10, 2031        │ ║
║  │ Archival Threshold: 90 days (EXCEEDED by 12 days)               │ ║
║  │                                                                 │ ║
║  │ [○] Retention Hold Active (Regulatory Freeze)                   │ ║
║  │ Hold Reason: Legal discovery request (Case-2024-456)            │ ║
║  │ Hold Owner: Legal Team │ Expires: Mar 1, 2025                   │ ║
║  │                                                                 │ ║
║  │ ⚠ HOLD PREVENTS ARCHIVAL - Review required before Mar 1        │ ║
║  │                                                                 │ ║
║  │ [View Hold Details] [Extend Hold] [Release Hold]               │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  RETENTION HOLDS (3 ACTIVE)                                           ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Hold ID: RH-2024-789                        [● Active]          │ ║
║  │ Artifact: 2020 Digital Strategy v3.2                               │ ║
║  │ Reason: Legal discovery request (Case-2024-456)                 │ ║
║  │ Owner: Legal Team │ Created: Dec 1, 2024 │ Expires: Mar 1, 2025│ ║
║  │ Impact: Blocks archival, purge, and modification                │ ║
║  │ [View Details] [Extend] [Release]                              │ ║
║  ├────────────────────────────────────────────────────────────────┤ ║
║  │ Hold ID: RH-2024-823                        [● Active]          │ ║
║  │ Artifact: Payment Processing Standards v1.1                        │ ║
║  │ Reason: Regulatory audit (SOX Compliance)                       │ ║
║  │ Owner: Compliance Officer │ Expires: Feb 15, 2025               │ ║
║  │ [View Details] [Extend] [Release]                              │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  UPCOMING EXPIRIES (NEXT 90 DAYS)                                     ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 12 artifacts will reach retention expiry in next 90 days           │ ║
║  │                                                                 │ ║
║  │ 📄 Legacy API Documentation v2.0             Expires: Feb 1     │ ║
║  │ 📄 Old Security Policy v1.3                  Expires: Feb 14    │ ║
║  │ 📄 Deprecated Data Model v4.1                Expires: Mar 5     │ ║
║  │ ... (9 more)                                                    │ ║
║  │                                                                 │ ║
║  │ [View All Expiring Artifacts] [Bulk Extend] [Configure Alerts]    │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  RETENTION POLICY SUMMARY                                             ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Repository Default Policy:                                      │ ║
║  │ • Active Artifacts: Indefinite retention                           │ ║
║  │ • Retired Artifacts: 7 years after retirement                      │ ║
║  │ • Evidence Artifacts: 5 years                                   │ ║
║  │ • Archival Threshold: 90 days in Retired state                  │ ║
║  │ • Cold Storage Tier: Azure Archive                              │ ║
║  │                                                                 │ ║
║  │ Purge Approval Required: ☑ Governance approval for all purges   │ ║
║  │ Purge Schedule: Monthly (1st of month)                          │ ║
║  │                                                                 │ ║
║  │ [Edit Repository Policy] [View Audit Log]                      │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 6. Scenario Context Management

**Purpose**: Governance interface for managing scenario tags (baseline, target, transition) across artifacts and ensuring reviewers understand scenario context during approval workflows. Addresses gap where scenario toggles are visible to artifact editors but not to governance reviewers.

**New Screen** (accessible from Governance > Analytics or Repository Settings)

```
╔══════════════════════════════════════════════════════════════════════╗
║ Scenario Context Management                      [Documentation]     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  [Baseline Artifacts] [Target Artifacts] [Transition Artifacts] [Conflicts]  ║
║                                                                       ║
║  SCENARIO TAG DISTRIBUTION                                            ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Baseline Artifacts:    247  ████████████████████░░ 78%             │ ║
║  │ Target Artifacts:       52  █████░░░░░░░░░░░░░░░░ 16%              │ ║
║  │ Transition Artifacts:   18  ██░░░░░░░░░░░░░░░░░░░ 6%               │ ║
║  │                                                                 │ ║
║  │ Total Artifacts: 317                                               │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  BASELINE ASSETS (247)                                                ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 🏛 2025 Digital Transformation Vision v1.1   [● Approved]       │ ║
║  │ ──────────────────────────────────────────────────────────────  │ ║
║  │ Scenario: Baseline │ Effective: Jan 15, 2025 → Always Active    │ ║
║  │ Classification: Landscape.Baseline                              │ ║
║  │                                                                 │ ║
║  │ Related Scenario Artifacts:                                        │ ║
║  │ • Target: 2025 Target Architecture Vision v2.0 (Draft)          │ ║
║  │ • Transition: Migration Roadmap v1.3 (In Review)                │ ║
║  │                                                                 │ ║
║  │ Downstream Consumption:                                         │ ║
║  │ • 2 transformations using as Baseline Reference                     │ ║
║  │ • Referenced by 3 target artifacts (transition planning)           │ ║
║  │                                                                 │ ║
║  │ ℹ Governance Guidance:                                          │ ║
║  │ This artifact represents CURRENT STATE. Changes to baseline artifacts │ ║
║  │ may impact active transformations. Ensure backward compatibility.   │ ║
║  │                                                                 │ ║
║  │ [View Artifact] [View Related Scenarios] [View Impact]            │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  TARGET ASSETS (52)                                                   ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ 🎯 2025 Target Architecture Vision v2.0      [○ Draft]          │ ║
║  │ ──────────────────────────────────────────────────────────────  │ ║
║  │ Scenario: Target │ Effective: Q3 2025 → Q4 2025                 │ ║
║  │ Classification: Landscape.Target                                │ ║
║  │                                                                 │ ║
║  │ Related Scenario Artifacts:                                        │ ║
║  │ • Baseline: 2025 Digital Transformation Vision v1.1 (Approved)  │ ║
║  │ • Transition: Migration Roadmap v1.3 (In Review)                │ ║
║  │                                                                 │ ║
║  │ Gap Analysis:                                                   │ ║
║  │ • 12 new capabilities defined (not in baseline)                 │ ║
║  │ • 5 baseline capabilities deprecated                            │ ║
║  │ • 8 capabilities enhanced                                       │ ║
║  │                                                                 │ ║
║  │ ℹ Governance Guidance:                                          │ ║
║  │ This artifact represents FUTURE STATE. Ensure alignment with       │ ║
║  │ strategic objectives and migration feasibility.                 │ ║
║  │                                                                 │ ║
║  │ [View Artifact] [View Gap Analysis] [Compare to Baseline]         │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  SCENARIO CONFLICTS & WARNINGS (2)                                    ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ ⚠ API Gateway Blueprint v2.1                 [⚠ Conflict]       │ ║
║  │ ──────────────────────────────────────────────────────────────  │ ║
║  │ Issue: Auto-classification suggests BASELINE, but artifact content │ ║
║  │        describes FUTURE STATE capabilities (target scenario)    │ ║
║  │                                                                 │ ║
║  │ Current Scenario Tag: Baseline (manual assignment)              │ ║
║  │ Auto-Suggested Tag: Target (91% confidence)                     │ ║
║  │                                                                 │ ║
║  │ ⚠ Resolution Required:                                          │ ║
║  │ Owner (Alice) should review scenario tag and update if needed.  │ ║
║  │ Governance review BLOCKED until scenario conflict resolved.     │ ║
║  │                                                                 │ ║
║  │ [View Artifact] [Notify Owner] [Override to Target] [Mark Valid]  │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  SCENARIO TOGGLE DOCUMENTATION                                        ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │ Scenario Tags Purpose:                                          │ ║
║  │ • Baseline: Current state / as-is architecture                  │ ║
║  │ • Target: Future state / to-be architecture                     │ ║
║  │ • Transition: Migration paths / interim states                  │ ║
║  │                                                                 │ ║
║  │ Governance Implications:                                        │ ║
║  │ • Baseline changes affect active transformations (high impact)      │ ║
║  │ • Target artifacts require strategic alignment review              │ ║
║  │ • Transition artifacts must link to baseline + target              │ ║
║  │                                                                 │ ║
║  │ [View Full Documentation] [View Journey 05: Landscape Toggle]  │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Visual Design Tokens

### Scenario Tags
```
🏛 Baseline    Current state architecture
🎯 Target      Future state architecture
🔄 Transition  Migration/interim state
```

### Trace Link Types
```
→ Satisfies     Requirement satisfaction
→ References    Standard/guideline reference
→ Builds On     Foundation dependency
→ Impacts       Downstream capability/element impact
```

### Retention States
```
[● Active]      Artifact in active use, no expiry
[○ Retired]     Artifact retired, retention window active
[⚠ Expiring]    Approaching retention expiry date
[🔒 Hold]       Retention hold (legal/regulatory freeze)
[❄ Archived]    Cold storage archival
```

### Auto-Classification Status
```
🤖 Auto-Suggested      AI-generated suggestion pending review
✓ Auto-Accepted        Owner accepted auto-suggestion
⚠ Exception Queued     Requires manual governance review
○ Manual Only          Auto-classification disabled
```

---

## Integration Points

### With Artifact UI ([220-artifact-entity-model-ui.md](220-artifact-entity-model-ui.md))
1. **Artifact Detail Page**: Stewardship, retention, scenario tags flow into Review Case Context tab
2. **Trace Link Visualizer**: Coverage data surfaces in Governance Analytics Trace Coverage tab
3. **Classification Manager**: Auto-suggestions feed Exception Queue for governance oversight
4. **Version Timeline**: Lifecycle events inform retention expiry calculations

### With Repository UI ([220-graph-entity-model-ui.md](220-graph-entity-model-ui.md))
1. **Owner Group Administration**: Links to Repository Access Control for auto-grant configuration
2. **Retention Policies**: Extends Repository General Settings retention configuration
3. **Classification Mappings**: Auto-classification exceptions reference external taxonomy mappings

### With Existing Governance UI ([220-governance-entity-model-ui.md](220-governance-entity-model-ui.md))
1. **Review Case Detail**: Adds Context tab with stewardship, retention, scenario, trace links
2. **Governance Analytics**: Adds Trace Coverage and Auto-Classification tabs
3. **Policy Manager**: Trace policies reference coverage metrics in analytics
4. **SLA Breach Monitor**: Escalation chains leverage Owner Group escalation configuration

---

## Summary of Gaps Addressed

| Gap | Solution Screen | Integration |
|-----|-----------------|-------------|
| Stewardship defaults only show current assignee without management controls | **Screen 2: Stewardship & Owner Group Administration** | Membership rosters, escalation chains, quorum rules, auto-grant configuration |
| Retention state/expiry tracked in model but not visible in governance workspaces | **Screen 1: Review Case Context Tab** + **Screen 5: Retention Enforcement Dashboard** | Retention context in review cases, archival queue monitoring, hold management |
| Scenario tags absent from case metadata | **Screen 1: Review Case Context Tab** + **Screen 6: Scenario Context Management** | Baseline/target/transition tags in review context, conflict detection, documentation |
| Governance screens omit trace-link visibility | **Screen 1: Review Case Context Tab** + **Screen 4: Analytics Trace Coverage** | Trace link summary in review cases, coverage metrics, high-impact review prioritization |
| Auto-classification suggestions lack governance oversight | **Screen 3: Auto-Classification Exception Queue** + **Screen 4: Analytics Auto-Class Tab** | Exception queue for governance review, confidence metrics, learning mode tracking |

---

## Next Steps

1. **Implement Context Tab in Review Case Detail** to unblock governance teams needing lifecycle visibility during approvals
2. **Deploy Auto-Classification Exception Queue** to provide oversight for automated placement decisions before they affect governance workflows
3. **Extend Governance Analytics** with Trace Coverage and Auto-Classification tabs to enable data-driven prioritization of impact-sensitive reviews
4. **Add Retention Enforcement Dashboard** for compliance teams managing archival, purge, and hold workflows
5. **Integrate Scenario Context Management** to prevent baseline/target conflicts and improve reviewer understanding of architecture evolution

All screens designed for seamless integration with existing governance, artifact, and graph UI patterns established in earlier wireframe documents.
