---
title: "700 – Frontends: reorganize `services/` UISurfaces + reverse-DNS naming (no compromises)"
status: draft
owners:
  - product
  - platform
  - web
created: 2026-01-17
---

# 700 – Frontends: reorganize `services/` UISurfaces + reverse-DNS naming (no compromises)

## Summary

This is primarily a **repo reorganization proposal**: rename and add **UISurface** frontends under `services/` to use a strict **reverse-DNS** identity scheme. The product split is the rationale, but the change is about folder structure, package identity, build contexts, and GitOps wiring.

We ship **one shared backend** (same domains, same APIs, same data) with **different UISurfaces**:

- `ameide.io` publishes the **Transformation Domain** as a vendor-neutral product experience.
- `com.future` publishes the public website for the **complete business application platform** (final domain TBD).
- `com.future.run` publishes the authenticated “run the business” experience (final domain TBD).

To make this explicit in the repo, we adopt a strict **reverse-DNS naming scheme** for frontend folders and packages, and we do the full rename (no parallel legacy names).

## Scope

In scope:

- UISurface frontends living under `services/` (directory names, package names, build/publish identity, GitOps wiring).

Out of scope:

- Renaming non-frontend backend services in `services/` (e.g. `services/threads`, `services/chat`, `services/platform`).
- Backend/domain refactors (Transformation remains the same backend).

## Decision

### A) Frontend folder names (repo-level truth)

Rename and add frontends under `services/`:

- `services/www_ameide` → `services/io.ameide`
- `services/www_ameide_platform` → `services/io.ameide.platform`
- Add `services/com.future` (new public website app)
- Add `services/com.future.run` (new run UI app)

### B) Package identity

Match `package.json` `name` to the folder identity:

- `www-ameide` → `io.ameide`
- `www-ameide-platform` → `io.ameide.platform`
- New: `com.future`
- New: `com.future.run`

### C) Hostnames / product mapping

The *product* mapping is:

- `ameide.io` → `io.ameide` (public entry + product narrative)
- `platform.ameide.io` (or equivalent) → `io.ameide.platform` (authenticated transformation product UI)
- `com.future` (domain TBD) → `com.future` (public website for the complete platform)
- `com.future.run` (domain TBD) → `com.future.run` (authenticated run UI), temporarily served under `run.ameide.io` until the final domain is chosen.

Exact DNS choices are owned by GitOps, but the repo structure must assume these are distinct UISurfaces.

### D) Top navigation (UX contract)

`io.ameide.platform` (Transformation product UI) top menu:

- Overview
- Repository
- Transformations
- Settings

`com.future.run` (Run the business UI) top menu:

- Overview
- Functions
  - Sales
  - Customer Management (CRM)
  - Procurement
  - Inventory & Warehousing
  - Manufacturing
  - Projects
  - Service Management
  - HR
  - Finance
  - Reporting & Analytics
  - Transformation
  - (extend as domains ship)
- Settings

## Constraints (non-negotiable)

1. **Same backend, different UIs.** No forked Transformation backend; only UI/UX differs.
2. **No local registries.** All images are published to GHCR and GitOps deploys digest-pinned refs (see `backlog/602-image-pull-policy.md`).
3. **Kubernetes resource names cannot contain dots.** Folder/package names may contain dots, but K8s object names must be DNS-1123 labels. We must define a deterministic slug:
   - `io.ameide` → `io-ameide`
   - `io.ameide.platform` → `io-ameide-platform`
   - `com.future` → `com-future`
   - `com.future.run` → `com-future-run`
4. **CLI + GitOps contracts remain stable.** Where external interfaces exist (e.g. `ameide dev`, `ameide test cluster`, environment ConfigMaps/Secrets), changes must be coordinated and documented (see `backlog/691-cli-surface-test-dev-e2e-smoke-dev.md`).

## Goals

- Make the **product split** explicit without duplicating backends or fragmenting domain ownership.
- Make the repo structure align with how we communicate the platform:
  - “Transformation is a domain” (but can be sold standalone at `ameide.io`).
  - “Full business platform” at `com.future` / `com.future.run` (domains TBD).
- Eliminate ambiguous naming (`www_*`) in favor of a consistent **`services/` UISurface naming scheme** that maps to:
  - package names,
  - image names,
  - GitOps app identifiers,
  - runtime configuration objects (via K8s-safe slugs).

## Non-goals

- Re-architecting the backend/domain model (this is a frontend + naming/backplane wiring change).
- Creating a second Transformation Domain service or schema.
- Keeping backwards-compatibility for old frontend folder names indefinitely.

## Work plan (implementation outline)

### 1) Repo refactor

- Rename directories under `services/` and fix all monorepo references:
  - `pnpm-workspace.yaml` paths
  - import paths and test paths
  - Docker build contexts and any scripts that assume `services/www_*`
- Update internal documentation references that point to old paths.

### 2) Build + publish identity

- Rename GHCR image repos to align with the new identities (folder/package-aligned), while respecting:
  - digest pinning in GitOps,
  - multi-arch `:main` channel tags for automation (per `backlog/611-trunk-based-main-and-gitops-environment-promotion.md`).

### 3) GitOps + runtime wiring (owned in `ameide-gitops`)

Coordinate a PR in `ameide-gitops` to:

- Rename charts/values/releases/config objects from `www-ameide*` to the new K8s-safe slugs.
- Ensure DNS + Gateway listeners cover the new hostnames (including dev wildcard needs for `ameide dev`, if applicable).
- Keep secret hygiene and existing secret authority models intact.

### 4) New `com.future` app

- Public website for the complete business application platform (unauthenticated).

### 5) New `com.future.run` app

- Scaffold a Next.js app that consumes the same Ameide SDKs and auth patterns.
- Establish product UX differences:
  - `io.ameide.platform`: transformation-first UX (initiatives, BPMN, artifacts, governance).
  - `com.future.run`: process-first “run the business” UX (domains/processes as primary navigation), with Transformation surfaced as a Function.

## Acceptance criteria

- `./ameide test` passes after the rename + new app introduction.
- GitOps deploys digest-pinned images for all UISurfaces with stable K8s-safe names.
- Public routing and authenticated routing work for:
  - `ameide.io` → `io.ameide`
  - `platform.ameide.io` (or final chosen hostname) → `io.ameide.platform`
  - `com.future` (domain TBD) → `com.future`
  - `com.future.run` (domain TBD) → `com.future.run`
- No remaining `services/www_ameide*` references in repo code paths (excluding historical docs or external references explicitly kept as compatibility notes).

## Cross references

- Platform vision and “Transformation is a domain”: `backlog/470-ameide-vision.md`
- Domains portfolio where Transformation is one domain among many: `backlog/475-ameide-domains.md`
- GitOps digest pinning and no-local-registry policy: `backlog/602-image-pull-policy.md`
- Trunk-based main + artifact promotion: `backlog/611-trunk-based-main-and-gitops-environment-promotion.md`
- CLI surface contracts that reference platform config objects: `backlog/691-cli-surface-test-dev-e2e-smoke-dev.md`
