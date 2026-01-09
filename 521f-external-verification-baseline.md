# 521f — External Verification Baseline (repo wiring + test contract)

External verification answers: “Do repo-owned scaffolds and test packs satisfy the required contract?”

**Baseline inputs**
- External generation baseline: `backlog/521a-external-generation-baseline.md`
- External generation improvements log: `backlog/521d-external-generation-improvements.md`
- Test contract (normative): `backlog/430-unified-test-infrastructure-v2-target.md`

---

## What we verify (baseline)

### Repo-owned integrity
- Tests exist (missing tests fail verification by default).
- Scaffold markers (`AMEIDE_SCAFFOLD`) are not present in test files.

### Test contract (430v2)
430v2 defines requirements like:
- strict phases (Unit → Integration → E2E)
- native tooling per language
- no `INTEGRATION_MODE`
- no per-component `run_integration_tests.sh` packs as the canonical execution path
- JUnit XML as mandatory evidence (including synthetic JUnit on early failures)

This baseline is where we define which parts are enforced by:
- the CLI (`ameide primitive verify`)
- CI workflows
- shared tooling (for legacy compatibility only; targeted for removal as part of 430v2)
