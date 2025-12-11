# RollingSync lint report helper

This note explains how to regenerate and interpret `backlog/387-argocd-waves-v2-lint.md`.

- **Entry point:** `./wave_lint.sh` in repo root. It wraps the original Python lint inline and runs it with `python3`.
- **Inputs:**  
  - `components_root` (positional; defaults to `gitops/ameide-gitops/environments/dev/components`)  
  - `--appset <path>` to parse RollingSync steps for the report (default: `gitops/ameide-gitops/environments/dev/argocd/apps/ameide.yaml`).  
  - `--format markdown|json` (markdown default), `--check` (lint-only, no output), `--output <file>` (write report), `--help`.
- **What it checks:** required component fields, numeric `rolloutPhase`, absence of `dependsOn`, and existence of chart/value files relative to the gitops repo root.
- **What it prints:**  
  - RollingSync steps with `maxUpdate` and the phases each step matches (from the ApplicationSet).  
  - Components grouped by phase with their `domain/project/namespace` and relative path.  
  - JSON mode emits the phase → components map for automation.
- **Usage examples:**  
  - `./wave_lint.sh --output backlog/387-argocd-waves-v2-lint.md`  
  - `./wave_lint.sh --format json > /tmp/waves.json`  
  - `./wave_lint.sh --check` to fail fast without printing the inventory.

The script does **not** hardcode any backlog details; it only reflects what’s in the repo. Compare the generated lint report with `backlog/387-argocd-waves-v2.md` or the example doc manually.
