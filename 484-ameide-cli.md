# 484 – Ameide CLI (Entry Point)

This file exists as a **stable link target** for backlog docs that reference `484-ameide-cli.md`.

Canonical CLI documentation is split across:

- `backlog/484-ameide-cli-overview.md` (overview / index)
- `backlog/484a-ameide-cli-primitive-workflows.md` (human/agent workflows)
- `backlog/484b-ameide-cli-proto-contract.md` (proto/SDK/codegen contract)
- `backlog/484c-ameide-cli-repo-gitops.md` (repo + GitOps interactions)
- `backlog/484d-ameide-cli-migration.md` (migration)
- `backlog/484e-ameide-cli-industry-patterns.md` (industry comparisons)
- `backlog/484f-ameide-cli-scaffold-implementation.md` (scaffold implementation notes)

Architecture note: the v2 primitive stack standardizes on **`buf generate` as the canonical generator runner** and treats any CLI as an orchestrator/observer around standard gates; see `backlog/520-primitives-stack-v2.md` and `backlog/472-ameide-information-application.md` §2.9.
