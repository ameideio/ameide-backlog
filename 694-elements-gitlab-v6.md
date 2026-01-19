---
title: "694 – Elements / Enterprise Repository in GitLab (v6: Git-backed canonical content)"
status: draft
owners:
  - platform
  - transformation
created: 2026-01-18
supersedes:
  - 694-elements-gitlab.md
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 527-transformation-v6-index.md
  - 695-gitlab-configuration-gitops.md
---

# 694 – Elements / Enterprise Repository in GitLab (v6: Git-backed canonical content)

This v6 file exists to keep the v6 “Git-backed Enterprise Repository” posture stable and easy to reference from other backlogs.

## Git provider posture (v6)

In the current platform posture, “Git-backed” concretely means:

- **GitLab is deployed in-cluster** as a platform component and managed by GitOps.
- GitLab is **not optional**: it is part of the default platform deployment.
- GitLab is **internal/private** to the platform: tenants do not log into GitLab; platform services use a bot identity to operate Git.

To avoid losing detail, the full decision text (including the “How this evolves prior work” paragraphs and external links) is preserved in:

- `backlog/694-elements-gitlab.md` (historical record; superseded by this v6 pointer)

Normative cross-references for the v6 story:

- Integration/EDA posture: `backlog/496-eda-principles-v6.md`
- Runtime posture: `backlog/520-primitives-stack-v6.md`
- Transformation v6 spine: `backlog/527-transformation-v6-index.md`

## Default tenant Enterprise Repository layout (v6)

This is a recommended, minimal repository layout that keeps “canonical truth is files” concrete without forcing a single file format:

```text
<tenant-enterprise-repo>/
  elements/                         # canonical authored artifacts (docs/diagrams/config)
  relationships/                    # optional relationship files (in addition to inline links)
  processes/                        # design-time governed ProcessDefinitions (BPMN as files)
    <module>/<process_key>/v<major>/
      process.bpmn
      bindings.yaml                 # optional
      README.md                     # optional
```

Normative rule: **`main` is the published baseline**, and “published” is a commit SHA on `main` (optionally tagged).

- GitLab deployment (GitOps) tracking: `backlog/695-gitlab-configuration-gitops.md`

## Alignment note (694 ↔ 695)

694 defines the **governance/tenancy posture**: GitLab is a platform-owned subsystem and the platform remains the enforcement surface (tenants do not get arbitrary GitLab accounts by default).

695 defines the **workload configuration and operations posture** (GitOps wiring, endpoints, secrets, smokes). If external hostnames exist (e.g., `gitlab.<env>.ameide.io`), they are for **operators/platform services** and should be network-restricted (private DNS/internal Gateway and/or allowlists + SSO), consistent with the “private subsystem” stance.

Progress note: GitLab is deployed as a standard GitOps component (`platform-gitlab`) and verified via PostSync smoke jobs (`platform-gitlab-smoke`), per `backlog/695-gitlab-configuration-gitops.md`.

## Governance enforcement surface (GitLab controls)

694’s governance posture assumes GitLab is configured so that “raw Git pushes” cannot bypass platform policy. Record (and keep consistent) the concrete GitLab controls relied upon, such as:

- Protected default branch (`main`): restrict who can push/merge.
- Require merge requests (no direct pushes) for protected branches.
- Require pipeline success (status checks) prior to merge.
- Limit who can approve/merge (bot/governance role vs operators), as applicable.

## Audit linkage contract (platform ↔ Git evidence)

Record the minimal immutable evidence pointers the platform stores for each accepted change, for example:

- Repository/project id + MR IID/URL (human traceability)
- Pipeline id + status (CI evidence)
- Merge commit SHA (immutable content anchor)
- Optional tag/release marker (published/baseline anchor)
