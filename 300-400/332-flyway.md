> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 332: Flyway Dual-Database Management _(Superseded – see backlog/363-migration-architecture.md)_

**Status:** Completed
**Date:** 2025-10-30
**Related Issues:** Agents service database connectivity, schema drift prevention

> **Update (2025-12-22):** Flyway is now treated as **schema-only** in GitOps; baseline/demo data (tenant/org/roles/repositories/personas) is owned by GitOps seed Jobs (e.g. `platform-dev-data`), not Flyway `V__`/`R__` scripts.

---

## 2025-11-05 Follow-up

- Added `infra/kubernetes/scripts/ensure-migrations-image.sh` and a Helm `prepare` hook so the local `db-migrations` job automatically builds/imports the `docker.io/ameide/ameide-migrations:dev` image before Flyway runs. This removes the manual “docker build + k3d import” step that used to break Step 42 on fresh machines.
- Removed runtime schema bootstrapping from the Platform, Transformation, and Workflows services, consolidating all DDL ownership inside Flyway migrations. Integration tests now apply the same SQL by reusing the checked-in Flyway scripts.
- Made the workflows V1 migration idempotent by guarding the `fk_workflows_service_definitions_latest_version` constraint with an `information_schema` check, so reruns after manual bootstraps no longer fail.

---

## Problem Statement

The platform uses two PostgreSQL databases with different purposes:
- **platform database**: Core application data (organizations, users, transformations, repositories, etc.)
- **agents database**: Agent definitions, routing rules, and execution state

Previously, only the platform database was managed by Flyway migrations. The agents database existed with a manually-created schema that was not version-controlled, leading to:
- Schema drift risk between environments
- No audit trail of schema changes
- Difficult deployment coordination
- Manual schema setup for new environments

## Solution

Implemented dual-database Flyway management where a single migration job processes both databases sequentially, with proper baselining for the existing agents schema.

### Architecture

```
db-migrations Helm Job
├── Migrate platform database (V1, V2, V3, ...)
│   ├── V1: initial schema
│   ├── V2: seed baseline data
│   └── V3: create agents schema (for reference only)
└── Migrate agents database (baseline at V3, then V4+)
    ├── V3: BASELINE (skip - schema exists)
    └── V4+: future migrations apply here
```

### Key Design Decisions

1. **Baseline at V3**: The agents database already had a complete schema, so we baseline at version 3 to skip V1, V2, V3 migrations (which are platform-specific)

2. **Separate DATABASE_URL**: Each database gets its own connection URL via Azure KeyVault/ExternalSecrets

3. **JDBC Format Conversion**: The migration job strips credentials from PostgreSQL URLs and converts them to JDBC format with separate user/password parameters

4. **Empty Schema Detection**: The job checks for "Empty Schema" state to determine if baselining is needed

---

## Implementation Details

### Files Changed

#### 1. New Migration File
**File:** `db/flyway/sql/platform/V3__create_agents_schema.sql`

Created a Flyway migration documenting the agents schema:
```sql
CREATE SCHEMA IF NOT EXISTS agents;
CREATE TABLE IF NOT EXISTS agents.nodes (
    node_id TEXT PRIMARY KEY,
    payload JSONB NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- ... additional tables: node_versions, instances, routing_rules, agent_tools, tool_grants
```

**Purpose:** Provides schema documentation and serves as the baseline reference for the agents database.

#### 2. Azure KeyVault Configuration
**File:** `infra/kubernetes/values/platform/azure-keyvault-secrets.yaml`

**Lines 517, 1095-1096** - Added AGENTS_DATABASE_URL to ExternalSecret templates:
```yaml
# For db-migrations secret (line 517):
AGENTS_DATABASE_URL: "postgresql://{{ `{{ .dbUser }}` }}:{{ `{{ .dbPassword }}` }}@postgres-ameide-rw.ameide.svc.cluster.local:5432/agents"

# For agents-db-secret (lines 1095-1096):
DATABASE_URL: "postgresql://{{ `{{ .dbUser }}` }}:{{ `{{ .dbPassword }}` }}@postgres-ameide-rw.ameide.svc.cluster.local:5432/agents"
```

#### 3. Migration Job Template
**File:** `infra/kubernetes/charts/platform/db-migrations/templates/job.yaml`

**Lines 110-165** - Added agents database migration logic:

```bash
# Migrate agents database if AGENTS_DATABASE_URL is set
if [ -n "${AGENTS_DATABASE_URL:-}" ]; then
  echo "============================================"
  echo "Migrating agents database..."
  echo "============================================"

  # Convert to JDBC URL format - strip credentials from URL
  AGENTS_JDBC_URL="${AGENTS_DATABASE_URL}"

  # Remove credentials if present (postgresql://user:pass@host -> postgresql://host)
  if [ "${AGENTS_JDBC_URL#*://}" != "${AGENTS_JDBC_URL}" ]; then
    AGENTS_PROTOCOL="${AGENTS_JDBC_URL%%://*}"
    AGENTS_REST="${AGENTS_JDBC_URL#*://}"
    if [ "${AGENTS_REST#*@}" != "${AGENTS_REST}" ]; then
      AGENTS_HOST_PATH="${AGENTS_REST#*@}"
      AGENTS_JDBC_URL="${AGENTS_PROTOCOL}://${AGENTS_HOST_PATH}"
    fi
  fi

  # Convert protocol to JDBC format
  if [ "${AGENTS_JDBC_URL#jdbc:}" = "${AGENTS_JDBC_URL}" ]; then
    case "${AGENTS_JDBC_URL}" in
      postgresql://*)
        AGENTS_JDBC_URL="jdbc:${AGENTS_JDBC_URL}"
        ;;
    esac
  fi

  # Check if baselining is needed
  echo "Running flyway info for agents database..."
  AGENTS_INFO_OUTPUT=$(flyway info -url="${AGENTS_JDBC_URL}" \
    -user="${FLYWAY_USER}" -password="${FLYWAY_PASSWORD}" 2>&1 || true)
  echo "${AGENTS_INFO_OUTPUT}"

  # Baseline at version 3 if schema history doesn't exist
  echo "Checking if agents database needs baselining..."
  if echo "${AGENTS_INFO_OUTPUT}" | grep -q "Empty Schema"; then
    echo "Agents database is new - creating baseline at version 3..."
    flyway baseline -baselineVersion=3 \
      -baselineDescription="Baseline existing agents schema" \
      -url="${AGENTS_JDBC_URL}" \
      -user="${FLYWAY_USER}" -password="${FLYWAY_PASSWORD}"
    echo "Baseline created. Flyway will skip V1, V2, V3 migrations."
  fi

  # Run migrations and validation
  echo "Running flyway migrate for agents database..."
  flyway migrate -url="${AGENTS_JDBC_URL}" \
    -user="${FLYWAY_USER}" -password="${FLYWAY_PASSWORD}"

  echo "Running flyway validate for agents database..."
  flyway validate -url="${AGENTS_JDBC_URL}" \
    -user="${FLYWAY_USER}" -password="${FLYWAY_PASSWORD}"

  echo "Agents database migration completed successfully"
fi
```

**Key Features:**
- Credential stripping prevents JDBC from parsing `user:pass@host` as hostname
- Baseline detection using "Empty Schema" string match
- Sequential execution after platform database migrations
- Comprehensive logging for debugging

#### 4. Local Environment Configuration
**File:** `infra/kubernetes/environments/local/platform/db-migrations.yaml`

**Line 18** - Added AGENTS_DATABASE_URL:
```yaml
env:
  FLYWAY_URL: jdbc:postgresql://postgres-ameide-rw.ameide.svc.cluster.local:5432/platform
  FLYWAY_USER: dbuser
  FLYWAY_PASSWORD: dbpassword
  AGENTS_DATABASE_URL: postgresql://dbuser:dbpassword@postgres-ameide-rw.ameide.svc.cluster.local:5432/agents
```

---

## Troubleshooting Journey

### Issue 1: JDBC URL Format - Credentials in Hostname
**Error:**
```
java.net.UnknownHostException: dbuser:dbpassword@postgres-ameide-rw.ameide.svc.cluster.local
```

**Root Cause:** Simple `jdbc:${URL}` prefix included credentials in what JDBC parsed as hostname.

**Fix:** Added shell script logic to strip credentials from URL before JDBC conversion.

### Issue 2: V1 Migration Against Agents Database
**Error:**
```
ERROR: schema "platform" does not exist
```

**Root Cause:** Flyway tried to run all migrations (V1, V2, V3) but V1/V2 are platform-specific.

**Fix:** Implemented baseline detection to skip V1, V2, V3 when operating on agents database.

### Issue 3: Empty Schema History Table
**Error:**
```
ERROR: Unable to baseline schema history table "public"."flyway_schema_history" as it already exists, and is empty.
```

**Root Cause:** Previous failed migration attempts left an empty flyway_schema_history table.

**Fix:** Manually dropped the table before re-running migration:
```bash
kubectl exec -n ameide postgres-ameide-1 -- \
  psql -U postgres -d agents -c "DROP TABLE IF EXISTS public.flyway_schema_history;"
```

#### 4. Layer-Driven Secret Bootstrap
- `infra/kubernetes/scripts/run-helm-layer.sh` invokes `scripts/infra/bootstrap-db-migrations-secret.sh` while reconciling Helm layer 42, so Tilt’s `infra:42-db-migrations` run always materialises `ameide/db-migrations-local` with the expected `FLYWAY_URL` / `FLYWAY_USER` / `FLYWAY_PASSWORD`.
- The helper populates credentials from the `platform-db-credentials` secret or falls back to the repo’s Vault defaults, preventing the pre-upgrade hook from failing with `FLYWAY_URL not set` after container restarts without relying on DevContainer hooks.

---

## Verification

### Platform Database State
```sql
SELECT version, description, type FROM flyway_schema_history ORDER BY installed_rank;
```
```
 version |     description      | type
---------+----------------------+------
 1       | initial schema       | SQL
 2       | seed baseline data   | SQL
 3       | create agents schema | SQL
```

### Agents Database State
```sql
SELECT version, description, type FROM flyway_schema_history ORDER BY installed_rank;
```
```
 version |           description           |   type
---------+---------------------------------+----------
 3       | Baseline existing agents schema | BASELINE
```

### Service Health
```bash
kubectl logs -n ameide deployment/agents --tail=10
```
```json
{"level":"INFO","msg":"agents service listening (Connect + gRPC)","addr":":8080"}
```

---

## Usage

### Running Migrations
```bash
# Run migrations for both databases
pnpm migrate

# The script will:
# 1. Build migration image with latest SQL files
# 2. Import image into k3d cluster
# 3. Upgrade db-migrations Helm release
# 4. Execute pre-upgrade hook job
# 5. Show migration output and results
```

### Adding New Migrations

#### Platform-Only Migration
Create `db/flyway/sql/platform/V4__your_migration.sql` with platform-specific changes:
```sql
-- This will run against the platform database only
ALTER TABLE platform.users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;
```

#### Agents-Only Migration
Create `db/flyway/sql/agents/V4__your_migration.sql` with agents-specific changes:
```sql
-- This will run against the agents database (baseline skips V1-V3)
ALTER TABLE agents.nodes ADD COLUMN metadata JSONB;
```

#### Both Databases
Create `db/flyway/sql/platform/V4__your_migration.sql` with conditional logic:
```sql
-- Check which database we're running against
DO $$
BEGIN
  IF EXISTS (SELECT 1 FROM pg_namespace WHERE nspname = 'platform') THEN
    -- Platform database changes
    ALTER TABLE platform.users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;
  ELSIF EXISTS (SELECT 1 FROM pg_namespace WHERE nspname = 'agents') THEN
    -- Agents database changes
    ALTER TABLE agents.nodes ADD COLUMN metadata JSONB;
  END IF;
END $$;
```

**Best Practice:** Keep migrations database-specific unless truly cross-cutting.

### Checking Migration Status
```bash
# View Helm release history
helm history db-migrations -n ameide

# Check job logs
kubectl logs -n ameide job/db-migrations

# Query Flyway history directly
kubectl exec -n ameide postgres-ameide-1 -- \
  psql -U postgres -d platform -c \
  "SELECT version, description, installed_on FROM flyway_schema_history;"

kubectl exec -n ameide postgres-ameide-1 -- \
  psql -U postgres -d agents -c \
  "SELECT version, description, installed_on FROM flyway_schema_history;"
```

---

## Production Deployment

### Azure KeyVault Setup
Ensure the following secrets exist in Azure KeyVault for each environment:

**Staging:**
```bash
az keyvault secret set --vault-name ameide-stg-kv \
  --name db-migrations-agents-database-url \
  --value "postgresql://dbuser:dbpassword@postgres-host:5432/agents"
```

**Production:**
```bash
az keyvault secret set --vault-name ameide-production-kv \
  --name db-migrations-agents-database-url \
  --value "postgresql://dbuser:dbpassword@postgres-host:5432/agents"
```

### Deployment Flow
1. **Pre-Deploy:** Review migration SQL files in PR
2. **Deploy:** Helm upgrade triggers pre-upgrade hook
3. **Migrate:** Job runs both database migrations sequentially
4. **Verify:** Check job logs and Flyway history tables
5. **Rollback:** If needed, use Helm rollback and manual SQL scripts

### First-Time Environment Setup
For new environments where agents database doesn't exist:
1. Create empty agents database
2. Run migration - baseline will be created automatically
3. Verify agents service connects successfully

For environments where agents database exists with schema:
1. Drop empty flyway_schema_history table if present:
   ```sql
   DROP TABLE IF EXISTS public.flyway_schema_history;
   ```
2. Run migration - baseline at V3 will be created
3. Verify no "schema does not exist" errors

---

## Benefits

✅ **Version Control:** All database schemas tracked in Git
✅ **Audit Trail:** Flyway history shows when/what changed
✅ **Consistency:** Same schema across all environments
✅ **Automation:** Migrations run automatically during deployments
✅ **Safety:** Validation prevents schema drift
✅ **Documentation:** Migration files serve as schema docs

---

## Future Considerations

### Multi-Database Pattern
This dual-database pattern can be extended to additional databases:
```yaml
# Add more database URLs to db-migrations secret
    WORKFLOWS_DATABASE_URL: postgresql://...
ANALYTICS_DATABASE_URL: postgresql://...
```

Then extend the job template with similar migration blocks.

### Database-Specific Migration Directories
Migrations are now split per database:
```
db/flyway/
├── sql/
│   ├── platform/
│   │   ├── V1__initial_schema.sql
│   │   ├── V2__seed_baseline_data.sql
│   │   └── …
│   ├── agents/
│   │   └── …
│   ├── workflows/
│   │   └── …
│   └── langgraph/
│       └── …
└── repeatable/
    └── …
```
Future platform migrations should live under `db/flyway/sql/platform`, agents migrations under `db/flyway/sql/agents`, workflows migrations under `db/flyway/sql/workflows`, and LangGraph checkpoint migrations under `db/flyway/sql/langgraph`.

### Baseline Automation
Currently, the first-time setup requires manually dropping empty schema history tables. Consider adding a Flyway callback or init container to handle this automatically.

---

## References

- [Flyway Baseline Documentation](https://documentation.red-gate.com/fd/baseline-184127464.html)
- [JDBC URL Format](https://jdbc.postgresql.org/documentation/use/)
- [Helm Pre-Upgrade Hooks](https://helm.sh/docs/topics/charts_hooks/)
- Platform database schema: [V1__initial_schema.sql](../db/flyway/sql/platform/V1__initial_schema.sql)
- Agents database schema: [V3__create_agents_schema.sql](../db/flyway/sql/platform/V3__create_agents_schema.sql)
- Workflow database schema: [V1__workflows_service_schema.sql](../db/flyway/sql/workflows/V1__workflows_service_schema.sql)
- LangGraph checkpoint schema: [V1__create_checkpoint_tables.sql](../db/flyway/sql/langgraph/V1__create_checkpoint_tables.sql)

---

## Related Work

- **Original agents schema creation:** Archived in `db/flyway/archive/V29__create_agents_schema.sql`
- **Agents service configuration:** Fixed DATABASE_URL to point to agents database in [azure-keyvault-secrets.yaml:1095](../infra/kubernetes/values/platform/azure-keyvault-secrets.yaml#L1095)
- **Workflows runtime circular import fix:** Made Python SDK generated module lazy-loaded
- **Tilt LiveUpdate permissions:** Excluded Go generated directories from sync
