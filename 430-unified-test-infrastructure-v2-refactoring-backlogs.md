---
title: 430v2 – Unified Test Infrastructure (Backlog Alignment Inventory)
status: proposal
owners:
  - platform
  - test-infra
created: 2026-01-09
updated: 2026-01-09
parent: 430-unified-test-infrastructure-v2.md
---

## Purpose

430v2 changes the repo-wide testing contract. This document inventories backlog docs that must be updated to:

- preserve historical context (do not delete v1 content)
- clearly mark legacy patterns as legacy
- avoid conflicting “normative” guidance
- consistently reference the v2 target contract:
  - `backlog/430-unified-test-infrastructure-v2-target.md`

---

## Update rubric (apply consistently)

For each affected backlog doc:

1) Add a top-level banner:
   - “Update (2026-01): superseded by 430v2; this document is historical where it references `INTEGRATION_MODE` / packs.”
2) Replace “target state” cross-references from 430(v1) to 430v2 target:
   - `backlog/430-unified-test-infrastructure-v2-target.md`
3) Keep the legacy content intact under a clearly labeled section:
   - “Legacy (v1 dual-mode / packs)”
4) Where the doc defines CLI behavior, align it to:
   - `ameide test` as the front door
   - strict phases
   - JUnit evidence

---

## Affected backlogs (must align)

### References `INTEGRATION_MODE` (v1 dual-mode contract)

- `backlog/300-400/319-onboarding-v2.md`
- `backlog/300-400/365-buf-sdks-v2.md`
- `backlog/300-400/376-integration-test-dual-modes.md`
- `backlog/406-tests.md`
- `backlog/428-onboarding.md`
- `backlog/430-unified-test-infrastructure.md`
- `backlog/430-unified-test-infrastructure-status.md`
- `backlog/468-testing-front-door.md`
- `backlog/484c-ameide-cli-repo-gitops.md`
- `backlog/521f-external-verification-baseline.md`
- `backlog/527-transformation-implementation-migration.md`
- `backlog/527-transformation-integration.md`
- `backlog/527-transformation-process.md`
- `backlog/537-primitive-testing-discipline.md`
- `backlog/588-smoke-tests-inventory.md`
- `backlog/591-capabilities-tests.md`
- `backlog/596-onboarding-capability.md`
- `backlog/597-login-onboarding-primitives.md`
- `backlog/621-agent-local-ci-runner-code.md`
- `backlog/621-ameide-cli-inner-loop-test.md`

### References `tools/integration-runner/*` (pack runner tooling)

- `backlog/300-400/307-integration-e2e-tests.md`
- `backlog/300-400/347-service-tests-refactorings.md`
- `backlog/482-adding-new-service.md`
- `backlog/484-ameide-cli-ORIGINAL-BACKUP.md`
- `backlog/551-comparative-integration.md`
- (plus many docs in the `INTEGRATION_MODE` list above)

### References `run_integration_tests.sh` (packs as required entrypoints)

Commonly referenced by:

- `backlog/300-400/361-integration-test-suite-alignment.md`
- `backlog/484a-ameide-cli-primitive-workflows.md`
- `backlog/484d-ameide-cli-migration.md`
- `backlog/484e-ameide-cli-industry-patterns.md`
- `backlog/484f-ameide-cli-scaffold-implementation.md`
- `backlog/510-domain-primitive-scaffolding.md`
- `backlog/511-process-primitive-scaffolding.md`
- `backlog/527-transformation-*` family (multiple)
- and other “testing discipline / packs / inner loop” docs

These docs must be updated so that:
- pack scripts are explicitly **legacy**
- Phase 1/2 are executed via native tooling, orchestrated by `ameide test`

---

## CI and workflow docs (must converge)

Docs that describe CI phases and quality gates must converge on the same semantics:

- `backlog/414-quality-gate-refactor.md`
- `backlog/610-ci-rationalization.md`
- `backlog/300-400/345-ci-cd-unified-playbook.md`
- `backlog/200-300/212-progressive-quality-gate-strategy.md`
- any doc that references `ci-integration-packs`, “integration packs”, or “repo/local/cluster modes”

---

## Out-of-scope (do not change due to 430v2)

These are often adjacent but not part of the v2 contract:

- GitOps smoke tests/hook jobs in `ameide-gitops` (tracked separately; not Phase 1/2).
- Operator/controller acceptance tests (envtest/kind/kuttl) unless they were incorrectly coupled to `INTEGRATION_MODE`.
