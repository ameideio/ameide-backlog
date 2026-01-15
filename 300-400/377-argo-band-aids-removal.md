# 377: Remove Argo band-aids and enforce clean drift-free syncs

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

## Checklist
- [x] Inventory all missing `_shared` and `dev` value files (foundation/platform/data/apps) and create stubs so Helm rendering no longer needs `ignoreMissingValueFiles`
- [x] Remove `ignoreMissingValueFiles` from foundation/data/platform/apps ApplicationSets and validate rendering
- [x] Switch foundation/data/platform/apps ApplicationSets to client-side apply (drop global `ServerSideApply`)
- [x] For components still forcing SSA, decide per-item:
  - [x] external-secrets operator
  - [x] external-secrets-ready
  - [x] cloudnative-pg
  - [x] gateway-api CRDs
  - [x] prometheus-operator CRDs
  - [x] redis CRDs
  - [x] clickhouse
  - [x] any others discovered during audit
- [x] Foundation-specific drift fixes:
  - [x] Remove `ignoreDifferences` block in foundation AppSet
  - [x] Set explicit ExternalSecret target fields (deletionPolicy, template engine/merge/metadata, remoteRef conversion/decoding/metadataPolicy) in values
- [x] Vault drift fixes:
  - [x] Add explicit StatefulSet defaults (revisionHistoryLimit, PVC retention policy) in values/overlays
  - [x] Remove PVC ignore once defaults are in place
  - [x] Reconcile webhook CA bundle handling to avoid SSA-only changes (pin caBundle, disable test hook)
- [x] Platform drift fixes:
  - [x] Ensure Tempo stays clean on client-side apply after dropping global SSA
  - [x] Review other platform components for server defaults; add explicit values if needed
- [x] Data/app drift fixes:
  - [x] Verify StatefulSets/Deployments render identically client-side; add explicit defaults where needed
- [ ] Break/fix cleanup spotted while auditing:
  - [x] Fix agents-runtime image to include `agents_service` module; bump chart tag (blocked: requires new image build)
- [x] Retired `runtime-agent-langgraph` in favor of `apps-agents-runtime` (image fixes no longer needed)
  - [x] Fix inference image to include `ameide_sdk.generated` protos; bump chart tag so inference + gateway can start (blocked: requires new image build)
  - [x] Clean up Langfuse exposure: rely on Envoy HTTPRoute only; drop redundant K8s Ingress config in values (dev)
  - [x] Ensure foundation gateway CRDs stay in-sync (no SSA band-aids); reconcile `foundation-crds-gateway` drift
- [ ] End-to-end verification:
  - [x] `kubectl diff`/`argocd app diff` for all env dev apps (foundation/data/platform/apps)
  - [x] All Argo apps report `Synced/Healthy` with no ignore rules or SSA band-aids

## Next steps / remaining work
- Final pass: rerun `kubectl diff`/`argocd app diff` to close the end-to-end verification items and confirm ApplicationSets render after the valueFiles indentation fix.
- Argo CD: bootstrap now layers the shared base values (`sources/values/common/argocd.yaml`) before env overlays and drops inline health overrides in dev; verify the live `argocd-cm` matches the canonical Lua after the next upgrade.
- AppSet resiliency: fixed a bad Helm valueFiles indentation in the foundation ApplicationSet templates that prevented app generation; keep linting for template output to avoid silent controller failures.
- Vault: ignore rules for the StatefulSet/webhook are now codified in component manifests and passed through the ApplicationSet; avoid ad-hoc live patches—rely on Git to add/remove ignores.
