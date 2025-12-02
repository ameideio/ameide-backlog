Here’s a north star doc you can paste into Notion/Confluence/ADR with minimal edits.

---

# North Star: CNPG-Owned Postgres Credentials (Vault as Mirror, Not Authority)

## 1. Context

We run Postgres on Kubernetes using CloudNativePG (CNPG), with applications consuming database credentials via Kubernetes Secrets. Historically, some credentials have been **pushed from Vault into Kubernetes** using ExternalSecrets (e.g. `postgres-ameide-auth`), and those credentials were sometimes also managed directly in Postgres.

This split-brain model (Vault as “source of truth”, Postgres + CNPG as enforcement) has caused issues:

* Secrets in Vault and passwords in Postgres drifting out of sync.
* Outages when rotations or manual changes updated one side but not the other (e.g. Langfuse).
* Extra coupling between DB availability and the Vault/ExternalSecrets path.

We want a clear, opinionated model for **who owns Postgres app credentials** and how apps consume them.

---

## 2. North Star Statement

> **CloudNativePG is the single source of truth for Postgres roles and passwords.
> Applications read credentials from CNPG-managed Kubernetes Secrets.
> Vault is optional and may mirror these secrets, but never drives them.**

In other words:

* CNPG + Kubernetes Secrets own **both** the credentials and the database roles.
* Vault integration is **downstream** (for visibility/audit), not upstream (for provisioning).

---

## 3. Design Principles

1. **CNPG as authority for credentials**

   * A single controller (CNPG) owns both:

     * The Postgres role (user/password in the DB).
     * The corresponding Kubernetes Secret (username/password, URIs).
   * The DB and Secret are always reconciled by the same operator.

2. **One Secret per app, managed in-cluster**

   * Each app (e.g. Langfuse, Temporal, Keycloak) has a dedicated DB user and Secret.
   * Apps only read from Kubernetes Secrets that CNPG knows about and reconciles.

3. **Superuser credentials are for operations only**

   * Superuser is kept in a separate Secret (e.g. `<cluster>-superuser`).
   * It is used only for bootstrap, maintenance, and emergency debugging — never for app connection strings.

4. **Rotation is driven via Secrets, not ad hoc SQL**

   * Password rotation happens by changing the Secret.
   * CNPG reconciles that change into the DB role.
   * Any rotation plan includes application restart / connection pool drain steps.

5. **Vault is a consumer, not a writer**

   * If Vault is required for audit/compliance:

     * It **reads** from CNPG-managed Secrets (via ESO / PushSecret) and stores them.
     * It does **not** push passwords back into CNPG-managed Secrets or directly into Postgres.

---

## 4. Target Architecture

### 4.1 Ownership model

* **Per-cluster:**

  * CNPG manages:

    * `<cluster>-superuser` Secret → `postgres` role.
    * `<cluster>-app` Secret → default low-privilege app user (optional, if we use it).
* **Per-application (preferred):**

  * For each app needing DB access (e.g. `langfuse`, `temporal`, `keycloak`):

    * A Postgres role is defined declaratively in `spec.managed.roles` (or equivalent mechanism).
    * That role is associated with a Kubernetes Secret, e.g. `<app>-db-auth`.
    * CNPG ensures that:

      * The role exists in the DB with the expected privileges.
      * The role’s password matches what is stored in the Secret.

**Result:** there is a tight, operator-controlled link between:

> “what’s in the Secret” ↔ “what the DB actually accepts”.

---

### 4.2 Secrets layout & naming

**Cluster-level:**

* `<cluster>-superuser`

  * `username: postgres`
  * `password: …`
  * For admin/maintenance only.
* `<cluster>-app` (optional default)

  * `username: <cluster>-app`
  * `password: …`
  * For simple use cases or legacy apps.

**App-level:**

* `<app>-db-auth`

  * `username: <app>`
  * `password: <strong random>`
  * Optionally:

    * `uri` / `jdbc-uri`
    * `pgpass`

Guideline:

* **New apps**: always use a dedicated `<app>-db-auth` Secret and role.
* **Existing shared “auth” secrets** (e.g. `postgres-ameide-auth`): migrate away toward per-app users.

---

### 4.3 Rotation model

* Rotation is initiated by updating the **Kubernetes Secret** (e.g. via External Secrets Operator’s `Password` generator, CI pipeline, or internal tool).
* CNPG:

  * Detects the Secret change.
  * Updates the corresponding Postgres role password.
* Platform team:

  * Coordinates app restart / connection pool drain where necessary.
  * Validates that new credentials are in use before retiring the old ones (if we use staged rotation).

Key points:

* No direct `ALTER ROLE ... PASSWORD` by hand without updating the Secret.
* No rotation that only changes Vault while leaving CNPG/DB stale.

---

### 4.4 Vault integration (optional)

If Vault is required:

* Use External Secrets **PushSecret** or equivalent to push CNPG Secrets into Vault.
* This provides:

  * Central visibility into DB credentials.
  * Vault-based audit on read access.
* What we explicitly **do not** do:

  * Vault **does not** write credentials into CNPG-managed Secrets.
  * Vault **does not** directly manage Postgres roles.

Conceptually:

> CNPG → Kubernetes Secret → (optional) Push → Vault

Not:

> Vault → ExternalSecret → Kubernetes Secret → CNPG / Postgres

---

### 4.5 Operational workflows

**New application:**

1. Define a managed role for the app in the CNPG Cluster spec (or follow the agreed template).
2. Create a Kubernetes Secret `<app>-db-auth` (or have ESO generate it).
3. Wire the app’s Deployment/Helm chart to read DB credentials from `<app>-db-auth`.
4. Grant the app’s role only the minimum required privileges.

**Password rotation:**

1. Trigger regeneration of the `<app>-db-auth` Secret (ESO / CI).
2. CNPG reconciles the new password into Postgres.
3. Restart the app or roll its pods to ensure they pick up the new credentials.
4. Confirm new connections succeed; optionally revoke any old/stale credentials.

**Incident handling:**

* If an app can’t connect due to auth failures:

  * First check the CNPG-managed Secret vs the role in Postgres.
  * Treat any divergent Vault values as **suspect**, not authoritative.

---

## 5. Scope & Non-Goals

**In scope:**

* Ownership and lifecycle of Postgres app users and passwords for CNPG-managed clusters.
* How applications should obtain and use database credentials.
* How Vault fits into this picture, if used.

**Out of scope (for now):**

* Non-Postgres secrets (e.g. API keys, other databases).
* Detailed RBAC design inside Vault.
* Cross-cluster secret replication patterns.
* Blue/green rotation schemes across multiple DB clusters.

These can be addressed separately, reusing the same general principle: the **service operator** that controls the resource (DB, cache, etc.) should own its credentials; Vault may mirror, not drive.

---

## 6. Migration Strategy (High-Level)

1. **Stop adding new Vault-authored DB credentials**

   * Freeze creation of new Vault-sourced Postgres users/secrets.
   * Mandate CNPG-owned credentials for all new services.

2. **Catalog existing patterns**

   * List clusters and apps currently using Vault-driven secrets (e.g. `postgres-ameide-auth`).
   * Identify which apps use shared credentials vs per-app credentials.

3. **Introduce CNPG-owned equivalents**

   * For each app:

     * Create a dedicated CNPG-managed role and Secret `<app>-db-auth`.
     * Grant appropriate privileges.

4. **Cut apps over**

   * Update app configs/Helm charts to read from the CNPG Secret.
   * Roll deployments and verify connectivity.
   * Monitor for errors.

5. **Optional: Mirror to Vault**

   * Configure PushSecret (or equivalent) to send CNPG Secrets to Vault for observability.
   * Document how teams can find DB credentials in Vault, with the caveat that Vault is **read-only** for these users.

6. **Retire legacy Vault-driven secrets**

   * Remove ExternalSecrets that were pushing credentials into CNPG.
   * Clean up unused Vault entries.

---

## 7. FAQ (for reviewers / stakeholders)

**Q: Why not keep Vault as the authority for DB passwords?**
A: Because we want a single controller to manage both the Postgres role and the Kubernetes Secret. With Vault as authority, we get two partially overlapping systems (Vault and CNPG) that can get out of sync, which is exactly what has caused outages.

**Q: Do we lose audit/compliance by not using Vault as the writer?**
A: No. Vault remains the audit surface for secret *access*. We simply change the direction of synchronisation: CNPG creates/owns credentials; Vault mirrors them for visibility and access logging.

**Q: Can we ever change passwords directly in Postgres?**
A: In emergencies, yes—but we must immediately update the corresponding Secret and reconcile back to the declarative state. The norm should be “change the Secret, let CNPG reconcile the DB”.

**Q: How does this affect developers?**
A: Developers point their app at “the CNPG secret name” for DB credentials. Behind the scenes, platform manages the lifecycle. From the app’s perspective, this is simpler and more consistent.

---

If you’d like, I can also turn this into a shorter ADR-style one-pager with an “Option A vs Option B” table and a final decision section.

---

## Current implementation status (dev)

- CNPG `platform-postgres-clusters` now declares managed roles and emits the app DB Secrets itself (platform/agents/agents_runtime/graph/transformation/threads/workflows/temporal/keycloak plus the base postgres auth). Secrets keep legacy names to avoid app changes.
- Vault-authored ExternalSecrets for those DB credentials were removed (foundation-vault-secrets-platform, foundation-vault-secrets-temporal, keycloak-db-credentials), eliminating the split-brain path; rollout-phase 250 now delivers both cluster and creds.
- Vault can be reintroduced later as a mirror (PushSecret) if needed, but is no longer authoritative for Postgres users/passwords.
