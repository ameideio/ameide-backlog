> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/368 – Vercel-Style Preview → Promote Workflow

**Created:** 2026-03-25  
**Owner:** Web Platform / Release Engineering  
**Scope:** services/www_ameide_platform + frontdoor DNS/SSL

---

## 1. Motivation

- Every PR already builds on Vercel, but reviewers still copy long `feature-xyz-<hash>.vercel.app` URLs from GitHub. Discovery is poor, and some stakeholders require friendlier domains (SOC testing, customer demos).
- Production rebuilds differ from the approved preview, so last-minute rebuild drift risks regressions and invalidates approvals.
- Rollbacks are manual (redeploy latest main) instead of instant domain swaps.

We want the **“Vercel way”**: automatic preview deployments, vanity preview domains, one-click promotion of the exact preview, and instant rollback—while remaining compatible with our Terraform/Kubernetes footprints.

---

## 2. Desired Flow

1. **Auto preview** – Any push to a non-production branch gets a Vercel Preview deployment with its own URL (already true).
2. **Readable URLs** – Optionally alias previews to tenant-friendly domains (`*.previews.ameide.dev`) via one CLI command or automation job.
3. **Promote exact build** – When ready, promote the approved preview to production (vercel dashboard or CLI). No rebuild.
4. **Instant rollback** – Domain swap back to the previous deployment if needed.

---

## 3. Domain Strategy

| Need | Approach |
|------|----------|
| Human-friendly preview host | Add CNAME `previews.ameide.dev` (or `v2.ameide.dev`) in DNS and connect domain in Vercel Project → Settings → Domains. |
| Many previews | Use wildcard `*.previews.ameide.dev` so each branch auto-gets `branch.previews.ameide.dev` (Vercel Preview Deployment Suffix). |
| Dedicated vanity alias | For high-profile initiatives, map a single preview to `initiative-x.ameide.dev` using `vercel alias set`. |

CLI snippet (for manual mode):

```bash
npm i -g vercel
vercel alias set https://feature-xyz-abcdefg.vercel.app v2.ameide.dev
```

Automation hook: GitHub Action listens for `deployment_status: success`, extracts the deployment URL, and runs `vercel alias set` with PAT scoped to the project.

---

## 4. Promotion & Rollback

### Promote
- **Dashboard**: Deployments → ⋮ → Promote to Production (ensures zero-downtime CNAME flip).
- **CLI**: `vercel promote <deployment-url> --yes`.
- **Guardrails**: require green checks (Playwright, smoke, Lighthouse) before promotion. Add status check `preview-ready` the action must set.

### Rollback
- **Dashboard**: select prior production deployment → ⋮ → Assign to domain.
- **CLI**: `vercel rollback` (aliased domain swap).
- Tie events into Release Service (`platform.release.v1`) so Transformation Initiatives and Ops dashboards record promotions/rollbacks.

---

## 5. Integration Points

1. **GitHub Checks** – Extend `cd-service-images.yml` (or dedicated Vercel action) to:
   - Comment preview + vanity URL.
   - Post promotion instructions.
2. **Transformation Service** – When automation plans (backlog/367) create PRs, they can also request aliasing:
   - Control plane adds `deployment_alias` metadata.
   - Agent runtime runs `vercel alias set` once preview ready.
3. **Kubernetes Config** – `AUTH_REDIRECT_PROXY_URL` already supports dynamic preview hosts (see `infra/kubernetes/charts/platform/www-ameide-platform/templates/configmap.yaml:69`), so Next.js Auth works behind vanity domains.
4. **Tenant overlays** – Store preview domain suffix in `tenants/<tenant>/automation/overlay.yaml` so agents know whether to alias automatically.

---

## 6. Implementation Plan

| Stage | Deliverable |
|-------|-------------|
| Stage 0 | Document DNS + Vercel domain setup, provision wildcard certs. |
| Stage 1 | GitHub Action `preview-alias` triggered on successful deployments; writes alias to `branch.previews.ameide.dev`. |
| Stage 2 | Promotion workflow: slash command `/promote-preview` calls `vercel promote`. Record event in Release DB. |
| Stage 3 | Rollback workflow: `/rollback-preview <deployment-url>` with audit trail. |
| Stage 4 | Tenant automation: Transformation plans specify preview alias requirements; Codex agents execute aliasing automatically. |

---

## 7. Open Questions

1. Do we need per-environment isolation (e.g., `previews.staging.ameide.dev` vs. `previews.prod.ameide.dev`), or single domain with role-based controls?
2. Should promotions block until Kubernetes `www` helm release completes, or do we trust Vercel as the sole production surface for `www_ameide_platform`?
3. How do we surface alias + promotion metadata back to backlog elements (Elements relationships) for traceability?

---

## 8. Next Steps

1. Register `previews.ameide.dev` (or similar) in DNS, add to Vercel project.
2. Land GitHub workflow invoking `vercel alias set` using deployment payload (token stored in org secrets).
3. Pilot with a single repository (`services/www_ameide_platform`) before generalizing to other UIs/micro-sites.
