# 527 Transformation - E2E Execution Sequence (Release; Implementation)

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

| Task | Process (Temporal) | Events (intent vs fact) | Infra (mechanics) | UI (what the user can see) |
| --- | --- | --- | --- | --- |
| Enter `release` | Emit `PhaseEntered(release)` | Process facts only | none | UI shows “ready to release” with checks |
| Build/publish artifacts | Activity requests publish step (or triggers CI via Integration) | Intent: publish execution intent; Facts: work lifecycle + evidence | CI/build system produces digest-pinned images | UI shows build status + produced digest(s) |
| Promote environments | Workflow waits explicitly (message callback or timer-based poll loop) and uses bounded Activities to check promotion outcome | Facts: release/promote outcomes emitted by owning domain(s) | GitOps promotion is a Git change per `backlog/611-trunk-based-main-and-gitops-environment-promotion.md` | UI shows promotion timeline + PR links |
| Close the run | Emit terminal process fact | Fact: `RunCompleted` / `RunFailed` | Infra is operational only | UI shows final status + evidence pointers |
