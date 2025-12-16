# 521e — Internal Verification Baseline (generation drift)

Internal verification answers: “Are Buf/plugins and generated-only outputs consistent and up to date?”

**Baseline inputs**
- Internal generation baseline: `backlog/521b-internal-generation-baseline.md`
- Internal generation improvements log: `backlog/521c-internal-generation-improvements.md`

---

## What we verify (baseline)

### Buf correctness (proto hygiene)
- `buf lint` (source-of-truth proto lint)
- `buf breaking` (API compatibility)

### Generated artifacts freshness (“codegen drift”)
- Generated roots exist (missing outputs are a hard failure when codegen is enforced)
- Generated artifacts match the current Buf templates (diff-based for TS; staleness heuristics for Go/Python)

---

## Where it is enforced

### CLI
`ameide primitive verify` includes internal-verification checks such as `BufLint`, `BufBreaking`, and `Codegen`.

### CI
CI remains the canonical gate for regen drift + compile/tests. The CLI may wrap these locally, but CI is authoritative.

