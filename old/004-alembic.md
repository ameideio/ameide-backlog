Below is a **code‑first, migration‑driven workflows** that gives you the *"press‑**migrate**"* developer experience you know from Entity Framework, while staying 100 % open‑source on **PostgreSQL + Apache AGE™**.

---

## 0  Architecture snapshot

```
┌────────────────────┐    alembic upgrade head
│  Python ontology   │──────────────▶│  AGE graph  │
│  (ArchiMate/BPMN)  │   generate    │   schema    │
└────────────────────┘   migration   └─────────────┘
        ▲                                   ▲
        │ introspect (ag_label)             │
        └──────────────┬────────────────────┘
                       │
             version table in Postgres
```

*We will combine:*

* **ALEMBIC** – lightweight, BSD‑licensed migration runner (Flyway/Liquibase also work but Alembic keeps everything in Python).
* **apache‑age‑python** driver – SQL + OpenCypher from Python ([GitHub][1]).
* AGE's own DDL helpers: `create_graph`, `create_vlabel`, `create_elabel`, `drop_label`, `alter_graph` … ([Apache AGE][2], [DEV Community][3]).
* AGE catalog tables (`ag_graph`, `ag_label`) for live schema introspection ([Apache AGE][2]).

### ✅ IMPLEMENTATION UPDATE - Alembic Integration Complete

Successfully implemented Alembic-based migrations for Apache AGE:

- **Migration utilities** in `ameide_core_age.migrations`
- **3 migrations created**:
  - `444088816452_initial_age_graph_setup.py` - Creates graphs and cross-model linking
  - `13771c627d09_add_sample_order_model.py` - Imports sample BPMN/ArchiMate models
  - `9e51eea7b128_add_uuid_fields_to_existing_nodes.py` - Adds UUID v5 versioning
- **All migrations applied** and working with graph-db-age service
- **Cross-model linking** function implemented
- **UUID-based versioning** for all graph elements

---

## 1  Bootstrapping the migration repo

```bash
pip install alembic apache-age-python psycopg[binary]
alembic init migrations
```

`alembic.ini`

```ini
sqlalchemy.url = postgresql+psycopg://ameide:pass@localhost:5432/ameide
```

*(psycopg v3 can coexist with AGE; no ORM layer needed).*

### 🚀 Current Implementation

```bash
# Install dependencies
poetry install

# Run migrations
poetry run alembic upgrade head

# Create new migration
poetry run ameide migrate make "Add new labels"

# Import models
poetry run ameide import-model archimate model.archimate
poetry run ameide import-model bpmn process.bpmn
```

---

## 2  The *schema‑as‑code* diff algorithm

1. **Read the current graph definition**

```python
def fetch_labels(conn, graph='archimate_bpmn'):
    sql = "SELECT label_name, kind FROM ag_catalog.ag_label WHERE graphname=%s"
    cur = conn.cursor()
    cur.execute(sql, (graph,))
    return {(r[0], r[1]) for r in cur.fetchall()}   # {('BusinessProcess','v'), …}
```

2. **Read the intended model**
   Using your `ameide‑ontology‑archimate` & `ameide‑ontology‑bpmn` wheels:

```python
def planned_labels(archi_model, bpmn_model):
    for el in archi_model.eAllContents():
        yield el.eClass.name, 'v'                   # vertex
    for rel in archi_model.getAllRelationships():
        yield rel.eClass.name, 'e'                  # edge
    # repeat for BPMN constructs you expose as graph labels
```

3. **Diff and emit operations**

```python
to_create = planned - current
to_drop   = current - planned
```

4. **Generate a migration script**

```python
from alembic import op
def upgrade() -> None:
    op.execute("LOAD 'age'; SET search_path TO ag_catalog;")
    for lbl, kind in to_create:
        fn = 'create_vlabel' if kind=='v' else 'create_elabel'
        op.execute(f"SELECT {fn}('archimate_bpmn','{lbl}');")
    for lbl, kind in to_drop:
        op.execute(f"SELECT drop_label('archimate_bpmn','{lbl}');")
```

5. **`alembic revision -m "Sync graph to ontology"`**
   Alembic writes an entry into its own `alembic_version` table (exactly like EF's `__EFMigrationsHistory`).

---

## 3  Running & rolling back migrations

### Apply

```bash
alembic upgrade head
```

Under the hood Alembic executes every outstanding script **in one PostgreSQL transaction**; AGE functions run just like any other DDL because they are SQL functions on the server.

### Roll back

```bash
alembic downgrade -1          # walks 'downgrade()' block
```

In the `downgrade()` you reverse the calls – e.g., `drop_label` first, `create_vlabel` for previously‑dropped ones.

> **Note**  AGE cannot yet *rename* vertex/edge labels (issue #1254) ([Stack Overflow][4]). Treat a rename as *drop + create* and migrate the data manually with a Cypher `MATCH … CREATE … DELETE`.

---

## 4  Property‑level migrations (optional)

Entity Framework also tracks column/property changes.
For AGE you can mimic this by:

1. **Attach a JSON schema** to every vertex/edge label in a side‑car table:

```sql
CREATE TABLE IF NOT EXISTS graph_schema (
   label text primary key,
   json_schema jsonb,
   version  text
);
```

2. **In your Alembic migration** compare the new schema with what's stored and:

```python
op.execute("""
   UPDATE graph_schema
   SET json_schema = %s, version = %s
   WHERE label = %s
""", (json.dumps(new_schema), vtag, label))
```

3. **Optionally validate** existing data with `jsonschema` or `pyshacl` before committing the migration.

### ✅ Implemented

The `graph_schema` table is created in the initial migration and tracks:
- Graph name
- Label name and kind (vertex/edge)
- JSON schema for validation
- Version tracking
- Timestamps

---

## 5  Multi‑environment promotion

Because AGE lives inside ordinary PostgreSQL schemas you can piggy‑back on *every* Postgres‑friendly migration orchestrator:

| Tool                 | OSS?       | How to hook                                                                               |
| -------------------- | ---------- | ----------------------------------------------------------------------------------------- |
| **Flyway Community** | Apache 2.0 | Store AGE DDL in `V2025.07.20__add_customer_label.sql`; `flyway migrate` runs them on CI. |
| **Liquibase**        | Apache 2.0 | Use SQL formatted changelogs; Liquibase tracks in `databasechangelog` table.              |
| **Sqitch**           | Permissive | Stores reversible "deploy/verify/revert" scripts; excellent for peer‑reviewed graph DDL.  |

All three simply execute the same `SELECT create_vlabel…` etc., so pick whichever matches your organisation's pipelines.

---

## 6  Seeding & data migrations

Entity Framework lets you seed reference data.
With AGE, add a *post‑DDL* block in the same migration file:

```sql
-- :seed_business_roles
SELECT * FROM cypher('archimate_bpmn', $$
  CREATE (:BusinessRole {name:'Customer'})
  CREATE (:BusinessRole {name:'Supplier'})
$$) AS _ ;
```

Because Alembic/Flyway execute the file top‑to‑bottom inside the same transaction, your seed data and new labels arrive atomically.

---

## 7  Putting it in CI

```
.github/workflows/ci.yaml
└─ services:
   ├─ postgres-age:15.4
   └─ temporal:latest
steps:
  - run: alembic upgrade head           # fails build on unapplied migrations
  - run: pytest tests/                  # run workflows unit tests
  - run: alembic downgrade base         # prove reversibility
```

---

## 8  Key take‑aways

* **AGE == PostgreSQL**, so *any* SQL migration tool works; just embed AGE‑specific helpers (`create_vlabel`, `drop_label`, etc.).
* Alembic gives you EF‑style *revision files* and a history table with *no proprietary code*.
* Introspect `ag_catalog.ag_label` to auto‑generate diffs and keep migrations boring & deterministic.
* Treat label renames and property shape changes as first‑class migrations—either DDL or Cypher data patches—exactly as you would treat column renames or data backfills in EF.

With this setup you can issue:

```bash
ameide migrate make "Add Invoice Agent"
ameide migrate apply
```

…and AGE will move *forwards* (or *backwards*) just as smoothly as an EF‑backed SQL Server ever did—only now it's *graph‑native* and **100 % Apache‑licensed**.

### 🎯 Project Structure

```
ameide-core/
├── src/
│   ├── __init__.py
│   ├── cli.py                    # CLI interface
│   ├── migrations/
│   │   ├── __init__.py
│   │   ├── age_migrations.py     # AGE migration utilities
│   │   └── __tests__/
│   │       └── test_age_migrations.py
│   └── importers/
│       ├── __init__.py
│       ├── importers.py          # Model importers
│       └── __tests__/
│           └── test_importers.py
├── migrations/                   # Alembic migrations
│   ├── versions/
│   │   └── 444088816452_initial_age_graph_setup.py
│   ├── env.py
│   └── script.py.mako
├── alembic.ini                  # Alembic configuration
└── pyproject.toml               # Poetry configuration
```

### 🚀 Next Steps

1. Run the initial migration to create graph schemas
2. Import sample ArchiMate and BPMN models
3. Create additional migrations as the ontologies evolve
4. Set up CI/CD pipeline with migration checks

[1]: https://github.com/rhizome-ai/apache-age-python?utm_source=threadsgpt.com "rhizome-ai/apache-age-python: Python driver for Apache AGE"
[2]: https://age.apache.org/age-manual/master/intro/graphs.html?utm_source=threadsgpt.com "Graphs — Apache AGE master documentation"
[3]: https://dev.to/hallixon/useful-apache-age-functions-2mhg?utm_source=threadsgpt.com "Useful Apache AGE Functions - DEV Community"
[4]: https://stackoverflow.com/questions/75364952/apache-age-changing-labels-of-nodes-and-edges?utm_source=threadsgpt.com "Apache AGE - Changing Labels of Nodes and Edges - Stack Overflow"