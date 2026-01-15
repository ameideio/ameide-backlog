# 378: ArgoCD Vault injector TLS via cert-manager (vendor-aligned)

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

> **Scope:** Applies to every environment where Vault injector relies on cert-manager. Cloud clusters consume Azure-issued wildcard/CA chains, while Terraform-managed local clusters reuse the same chart with a SelfSigned → CA Issuer pattern (see [444-terraform.md](../444-terraform.md#local-target-safeguards)). The operational guidance below covers both; local simply points the Issuer at the in-cluster CA secrets.

## Why
- Vault injector webhook drifted because the live `caBundle` gained multiple PEMs; Argo marked `MutatingWebhookConfiguration` OutOfSync.
- We want vendor-aligned TLS for the injector with cert-manager managing issuance, while keeping Argo health/sync deterministic.

## Vendor guidance (HashiCorp)
- **Manual TLS (no cert-manager):** create your own CA + webhook TLS secret, pin the PEM chain in `injector.certs.caBundle`. (HashiCorp “Vault Agent Injector TLS configuration”)  
  https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/injector-tls
- **With cert-manager:** issue CA + serving cert via cert-manager, annotate the webhook with `cert-manager.io/inject-ca-from`, leave `caBundle` to cainjector. (HashiCorp “Vault Agent Injector TLS with cert-manager”)  
  https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/injector-tls-cert-manager
- Chart knobs: `injector.certs.secretName` (TLS secret) and `injector.certs.caBundle` (PEM) are the official inputs.  
  https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/configuration#injector-certs-cabundle
- cert-manager CA injection docs: https://cert-manager.io/docs/concepts/ca-injector/

## Chosen approach (dev)
- Use cert-manager to issue a self-signed CA + serving cert for the injector (`vault-webhook-certs` chart/component). Local/k3d clusters reuse the same component with Issuers defined inside `ameide-local` so no Azure/DNS-01 dependency exists.
- Pin `injector.certs.secretName` to the cert-manager secret and set `cert-manager.io/inject-ca-from` on the webhook.
- Pin `injector.certs.caBundle` to the CA PEM (single chain from cert-manager) so Argo has a deterministic manifest, with an ignore-diff on `webhooks[0].clientConfig.caBundle` acknowledging cainjector ownership.

## Rationale
- Aligns with HashiCorp cert-manager pattern for issuance and CA injection.
- Keeps Argo sync stable by pinning the CA to the cert-manager secret contents (single PEM), avoiding multi-PEM drift.
- Minimal ignores: only the injected `caBundle` field is excluded from drift checks; all other webhook fields remain under Git control.

## Current state (dev)
- Cert-manager `vault-webhook-certs` component issues the CA + serving cert; secrets are healthy.
- Vault chart pins `injector.certs.secretName`/`caBundle` to the current cert-manager CA PEM; webhook annotated with `cert-manager.io/inject-ca-from` and Argo ignore on `webhooks[0].clientConfig.caBundle`.
- Argo ignore rules for the Vault StatefulSet/webhook are defined in component YAML and now passed through by the foundation ApplicationSet (post valueFiles indentation fix).
- Sync is clean after deleting/recreating the app; no band-aids beyond the intentional caBundle ignore.

## Follow-ups
- When ready to trust cainjector rotation, drop the pinned `caBundle` and keep only the injection annotation (retain the ignore on the injected field).
- For production hardening, consider a ClusterIssuer/KMS-backed CA instead of self-signed in-cluster CA; update the pinned PEM accordingly.
- Keep a lightweight script/check to refresh the pinned PEM from the cert-manager CA secret before expiry.
