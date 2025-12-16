# 521f — External Verification Baseline (repo wiring + test contract)

External verification answers: “Do repo-owned scaffolds and test packs satisfy the required contract?”

**Baseline inputs**
- External generation baseline: `backlog/521a-external-generation-baseline.md`
- External generation improvements log: `backlog/521d-external-generation-improvements.md`
- Integration test contract: `backlog/430-unified-test-infrastructure.md`

---

## What we verify (baseline)

### Repo-owned integrity
- Tests exist (missing tests fail verification by default).
- Scaffold markers (`AMEIDE_SCAFFOLD`) are not present in test files.

### Integration-pack contract (430)
430 defines requirements like `INTEGRATION_TEST_MODE=mock|cluster`, `run_integration_tests.sh`, `__mocks__/`, and JUnit/structured logging expectations.

This baseline is where we define which parts are enforced by:
- the CLI (`ameide primitive verify`)
- CI workflows
- shared tooling (e.g., `tools/integration-runner/**`)

