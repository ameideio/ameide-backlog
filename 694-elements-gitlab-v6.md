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
