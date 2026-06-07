# Lakebase Postgres — Learning Notes

Concise notes from POC + discussion. Official hub: [Lakebase Postgres](https://docs.databricks.com/aws/en/oltp/projects/) · [Use cases](https://docs.databricks.com/aws/en/oltp/projects/use-cases)

---

## What it is

- **Fully managed Postgres** on Databricks (OLTP), integrated with **Unity Catalog**
- **Compute and storage are separate** — data persists when compute sleeps (scale-to-zero)
- **Not** live “federation” to Delta — for serving lakehouse data you **sync** UC tables into Postgres ([synced tables](https://docs.databricks.com/aws/en/oltp/projects/sync-tables))
- Runs in **workspace cloud region** (e.g. AWS `us-east-2`) on **Databricks-managed storage** — not your ADLS paths for Postgres data

---

## Autoscaling vs Provisioned

| | **Autoscaling** (use this) | **Provisioned** (legacy) |
|--|---------------------------|-------------------------|
| New projects | **Default** since Mar 2026 | Being upgraded/migrated |
| Sizing | Min/max **CU**, autoscale | Fixed CU |
| Scale-to-zero | Yes | No |
| Branches | Yes | No |
| DAB resource | `postgres_projects` | `database_instances` |

---

## Hierarchy (mental model)

```
Workspace (e.g. a360-dev)
  └── Project          e.g. postgres-test     ← one “Postgres system”
        └── Branch     e.g. production       ← environment (isolated data + compute)
              └── Database   e.g. databricks_postgres
                    └── Schemas / tables     e.g. public.todos
```

| Term | Meaning |
|------|--------|
| **Project** | Top-level container; has region, default CU, history retention |
| **Branch** | Isolated environment (UI/API only — **not** SQL `CREATE BRANCH`) — see [Branches & COW](#branches-cow-and-time-travel-core-architecture) |
| **Database** | Normal Postgres `DATABASE`; multiple per branch possible |
| **Primary compute** | Read-write engine (`.5 ↔ 2 CU` etc.) |
| **Endpoint** | Stable connection address (host) — unchanged when compute scales |
| **Data size (UI)** | Branch storage on managed layer (≥ `pg_database_size()` — includes WAL, metadata) |

**Default branch name `production`** = Lakebase default on a branch, **not** your company prod workspace.

---

## Architecture (storage + compute + query)

Reference: [Core concepts](https://docs.databricks.com/aws/en/oltp/projects/core-concepts)

### Overall (disaggregated Postgres)

```
┌─────────────────────────────────────────────────────────────┐
│  CLIENT (SQL Editor, App, psql, JDBC)                        │
└───────────────────────────┬─────────────────────────────────┘
                            │ TCP to ENDPOINT (stable hostname)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  COMPUTE LAYER (elastic, per branch)                       │
│  • primary = read-write (0.5–64+ CU autoscale)             │
│  • optional read replicas (same storage, more read compute)  │
│  • optional HA secondaries (failover, same endpoint)         │
│  • can SCALE TO ZERO when idle → no compute $               │
│  Runs: Postgres engine + RAM (buffer cache, query work)      │
└───────────────────────────┬─────────────────────────────────┘
                            │ read/write PAGES (not whole tables)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  STORAGE LAYER (durable, per branch)                         │
│  • Databricks-managed in project region (e.g. AWS S3-class)  │
│  • NOT your ADLS/S3 paths for UC Delta                       │
│  • heap files, index files, WAL, catalogs                  │
│  • copy-on-write across branches (child branch shares until │
│    changed)                                                  │
└─────────────────────────────────────────────────────────────┘
```

**Key idea:** Data **outlives** compute. Monitoring may show **ENDPOINT INACTIVE** while **Data size** on branch stays > 0.

---

### Storage — what lives where

| Stored on branch storage | Examples |
|--------------------------|----------|
| **Table heap** | Row data in ~8 KB **pages** |
| **Indexes** | B-tree files (PK, secondary indexes) |
| **System catalogs** | `pg_catalog` — table/column/index definitions |
| **WAL** | Durability for commits |
| **Stats** | Planner row counts (ANALYZE) |

| Not the same as Delta | |
|-----------------------|---|
| No Parquet files / `_delta_log` for Postgres tables | |
| UC **gold** stays in lakehouse storage until **synced table** copies into Lakebase | |

**UI “Data size”** ≈ whole branch footprint. **`pg_database_size()`** ≈ one database (~7 MB empty POC); gap = WAL + branch metadata + platform overhead.

**Empty `public` in Tables UI** can still be **~30 MB** branch — catalogs + internals, not user tables.

---

### Compute — what it does

| Piece | Role |
|-------|------|
| **Endpoint** | Connection URL; stays same through scale/failover |
| **Primary compute** | All writes + reads unless you use replica endpoint |
| **CU min ↔ max** | Autoscale **while awake** (you configure; not fixed 8–16) |
| **Scale-to-zero** | After idle timeout → 0 CU; next query **wakes** primary |
| **Read replicas** | Extra read-only compute; **same storage** (no full copy) |
| **Databricks App compute** | Separate process (Streamlit); talks to Lakebase over network |

Changing min/max CU may **briefly interrupt** connections.

---

### Query flow — `SELECT * FROM t WHERE id = 1`

**Filtering happens on compute (Postgres), not in S3/object storage.**

| Step | Where | What |
|------|--------|------|
| 1 | Client | Send SQL to **endpoint** |
| 2 | Lakebase | Wake **primary** if scale-to-zero slept |
| 3 | Compute | **Parse** + **planner** — see PK/index on `id` → often **Index Scan** |
| 4 | Compute | Walk **index B-tree** → find `id=1` → pointer **(heap page #, row slot)** |
| 5 | Compute | Need heap **page** → check **buffer cache (RAM)** |
| 6 | Storage | **Cache miss** → storage service returns that **page** only |
| 7 | Compute | Read row from page; apply any extra predicates; **return result** |
| 8 | Idle | Compute may scale down; **data remains** on storage |

**Without index** (`WHERE msg = '...'`): **Seq Scan** — read many/all heap pages, filter each row in compute (more I/O).

**Wrong mental model:** “SQL runs in S3” or “load full table into RAM every query.”  
**Right model:** “Postgres pulls **needed pages** into RAM; index shrinks pages read.”

### Metadata for finding rows (Postgres vs Delta)

| | **Lakebase (Postgres)** | **Delta (lakehouse)** |
|--|-------------------------|------------------------|
| Row location | Index → heap **page + slot** | File list + Parquet row groups + stats |
| Metadata | `pg_catalog`, index files on storage | `_delta_log`, manifest, column stats in Parquet |
| Filter execution | Postgres engine on compute | Spark/Photon + predicate pushdown on files |

---

### CU (compute units) — sizing

- **CU** ≈ Postgres compute (~2 GB RAM/CU on Autoscaling UI)
- **Min ↔ max** = autoscale range while awake; **editable** on primary compute
- **Project default compute** = preset for **new** branches only
- **App compute** (DBU) ≠ **Lakebase CU**

**Useful SQL (storage inspection):**

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));

SELECT n.nspname, c.relname, pg_size_pretty(pg_total_relation_size(c.oid))
FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r','i') ORDER BY pg_total_relation_size(c.oid) DESC LIMIT 20;
```

---

## Branches, COW, and time travel (core architecture)

### Branch ≠ SQL command

- Created via **Lakebase UI**, API, or DAB (`postgres_branches`) — **not** `CREATE BRANCH` in Postgres.
- Each branch: own **compute**, **endpoint**, **data view**; optional **auto-delete** (e.g. after 1 day for sandboxes).

### Create branch — UI options

| Field | Meaning |
|-------|--------|
| **Name** | Branch id (e.g. `dev-test`) |
| **Auto-delete** | Remove branch after TTL (good for throwaway dev) |
| **Parent branch** | Fork source (e.g. `production`) |
| **Branching point — current** | Snapshot = parent **right now** |
| **Branching point — past time** | Snapshot at timestamp within **history retention** (e.g. 7 days) |

Banner: *“Created instantly; only uses storage when you make changes”* = **copy-on-write** (below).

### `production` branch vs live data

- **`production` always shows current state** — latest commits only.
- Rows you **deleted** or **updated** are **not** visible on `production` as old values.
- “Last 2 days of todos” on `production` = rows **inserted** recently and **still present**, not a 2-day history view.

### History / time travel (similar goal to Delta, different mechanism)

| | **Delta** | **Lakebase** |
|--|-----------|--------------|
| Retention | Log / file retention per table | Project **history retention** (e.g. 7 days; up to ~30 on Autoscaling) |
| Read old data | `VERSION AS OF` / `TIMESTAMP AS OF` on **same table** | **New branch** from parent at **past timestamp** |
| Recover deleted row | Query older snapshot → re-insert | Branch **before delete** → `SELECT` → `INSERT` into `production` if needed |
| After retention | Old versions gone | Old branch points unavailable |

**Pick branching time before the delete/update** and within retention — e.g. branch “yesterday” recovers row deleted **today**; branch “3 days ago” only helps if that row existed then.

### Copy-on-write (Lakebase branches) — simplest terms

**Don’t copy the whole database at fork; share storage until a branch changes data; then copy only changed pages.**

Not Iceberg “rewrite Parquet files on UPDATE” — see comparison below.

#### Example: `todos` on `production`

**Start — `production`:**

| id | task | completed |
|----|------|-----------|
| 1 | learn lakebase | false |
| 2 | learn databricks apps | false |
| 3 | write docs | false |

**1) Create `dev-test` from `production` (current time)**

```
production  ──┐
              ├──►  shared storage pages P  (all 3 rows)
dev-test    ──┘
```

Instant; **little extra storage**. Both branches: `SELECT *` → 3 rows.

**2) Changes only on `dev-test`**

```sql
DELETE FROM todos WHERE id = 2;
UPDATE todos SET completed = true WHERE id = 1;
```

- **`dev-test`:** rows 1 (completed), 3 — row 2 gone.
- **`production`:** still rows 1, 2, 3 unchanged.

**COW on write:** only **changed heap/index pages** get new versions for `dev-test`; `production` still reads original pages.

```
production  ──►  P   (unchanged)

dev-test    ──►  P'  (new pages for updated/deleted rows only; row 3 may still share P)
```

**3) Delete on `production` later**

- `production` loses row 2 on **live** branch.
- **`recover`** branch from **timestamp before delete** → row 2 still visible there (within retention).

### Lakebase branch COW vs Iceberg table COW

Same phrase, **different layer**:

| | **Iceberg copy-on-write** | **Lakebase branch copy-on-write** |
|--|---------------------------|-----------------------------------|
| **When** | INSERT/UPDATE/DELETE on **table** | **Create child branch** or **write** on that branch |
| **Unit** | **Parquet files** / row groups | **Storage pages** (~8 KB Postgres blocks) |
| **Purpose** | How **table commits** are written (vs merge-on-read) | Cheap **fork** + isolated **environments** |
| **Merge delete files** | MoR merges on read; CoW rewrites files | N/A — Postgres heap/WAL on that branch |

| One line |
|----------|
| **Iceberg CoW:** rewrite **files** on row change; keep snapshots in log. |
| **Lakebase CoW:** fork **branch** shares disk; copy **pages** only when that branch changes data. |

---

## Four use cases ([doc](https://docs.databricks.com/aws/en/oltp/projects/use-cases))

| # | Pattern | Direction | Status (typical) |
|---|---------|-----------|------------------|
| 1 | **Serve lakehouse data** | UC Delta → Postgres (**synced tables**) | GA |
| 2 | **Store Postgres changes** | Postgres → Delta (**Lakebase CDF**) | **Public Preview** |
| 3 | **Application backend** | App ↔ Postgres (Apps, drivers, Data API) | GA |
| 4 | **AI agents & ML** | Memory / online features | Product-specific |

### Sync modes (use case #1)

| Mode | When |
|------|------|
| **Snapshot** | One-time / full refresh; views, Iceberg without CDF |
| **Triggered** | Scheduled or after gold job; incremental if source has **Delta CDF** |
| **Continuous** | Near real-time; highest cost |

**Triggered** = mode; **Schedule ongoing syncs** = set cadence (same thing, two doc steps).

---

## Workspace / project layout (healthcare E2E)

| Layer | Pattern |
|-------|---------|
| **DBX dev / stage / prod workspace** | Company env isolation (`haredecodes` → stage → `haredecodes_prod`) |
| **Lakebase project** | Often **one per app per workspace** (e.g. `healthcare-ops`) |
| **Lakebase branches** | Optional sandboxes **inside** one project (PR/CI testing) — **not** a substitute for dev/stage/prod workspaces |
| **Bundle deploy** | `bundle deploy -t dev` / `-t stage` / `-t prod` deploys `postgres_projects` ([manage with bundles](https://docs.databricks.com/aws/en/oltp/projects/manage-with-bundles)) — repo today has **no** Lakebase resources yet |
| **Synced tables** | Source catalog must match workspace (`haredecodes` vs `haredecodes_prod`) |

See [Dev → stage/prod deployment](#dev--stageprod-deployment-schema-only) for how to promote schema without copying dev data.

---

## Dev → stage/prod deployment (schema only)

### The key distinction first

You do **NOT** copy/move a dev Lakebase branch to stage or prod.

| What you deploy | How | Dev data copied? |
|-----------------|-----|------------------|
| Lakebase project / branch / endpoint | **Asset Bundles** (`postgres_*` resources) | No |
| Schema / tables / indexes | **SQL migrations** (Flyway, Alembic, etc.) | No |
| Dev rows / test data | Never auto-promoted | No |
| Seed / reference data | Optional script, dev/stage only | Controlled |
| App code | Databricks Apps deploy | N/A |

**Pattern:** deploy **infrastructure** per workspace via bundles; deploy **DDL** per endpoint via versioned migrations. Schema arrives; dev data stays in dev.

**Lakebase branch vs workspace:**

```
Lakebase branch   = sandbox inside ONE workspace/project (ci-pr-123, dev-test)
DBX workspace     = company environment (dev / stage / prod)
```

Branches are for PR isolation and schema testing **within** a workspace. Cross-environment promotion = separate workspace + same migration files.

---

### 1. `databricks.yml` — targets + Lakebase infra

```yaml
bundle:
  name: healthcare-end-to-end-project

targets:
  dev:
    workspace:
      host: https://adb-dev-<id>.azuredatabricks.net

  stage:
    workspace:
      host: https://adb-stage-<id>.azuredatabricks.net

  prod:
    workspace:
      host: https://adb-prod-<id>.azuredatabricks.net

resources:
  postgres_projects:
    healthcare_ops:
      project_id: healthcare-ops
      display_name: Healthcare Ops
      pg_version: 17
      enable_pg_native_login: false

  postgres_branches:
    main_branch:
      parent: ${resources.postgres_projects.healthcare_ops.id}
      branch_id: production
      no_expiry: true

  postgres_endpoints:
    primary:
      parent: ${resources.postgres_branches.main_branch.id}
      endpoint_id: primary
      endpoint_type: ENDPOINT_TYPE_READ_WRITE
      autoscaling_limit_min_cu: 0.5
      autoscaling_limit_max_cu: 2
```

Deploy infra to stage (empty project + branch + endpoint — no schema, no data yet):

```bash
databricks bundle deploy -t stage
```

Repeat with `-t dev` / `-t prod` for each workspace. Same bundle definition; each target creates an isolated Lakebase project in that workspace.

**Note:** `branch_id: production` is the Lakebase default branch name inside the project — not your company prod workspace.

---

### 2. `migrations/` — source of truth for schema (DDL)

```
repo/
├── databricks.yml
├── migrations/
│     ├── V001__create_schema.sql
│     ├── V002__create_tables.sql
│     └── V003__add_indexes.sql
└── .github/workflows/
      ├── deploy-dev.yml
      ├── deploy-stage.yml
      └── deploy-prod.yml
```

```sql
-- V001__create_schema.sql
CREATE SCHEMA IF NOT EXISTS app;

-- V002__create_tables.sql
CREATE TABLE IF NOT EXISTS app.current_fares (
  route_id    TEXT PRIMARY KEY,
  city        TEXT NOT NULL,
  fare        NUMERIC(10,2) NOT NULL,
  updated_by  TEXT,
  updated_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS app.user_entitlements (
  user_email  TEXT NOT NULL,
  region      TEXT,
  city        TEXT,
  can_read    BOOLEAN DEFAULT false,
  can_write   BOOLEAN DEFAULT false,
  granted_by  TEXT,
  granted_at  TIMESTAMPTZ DEFAULT now(),
  revoked_at  TIMESTAMPTZ
);

-- V003__add_indexes.sql
CREATE INDEX IF NOT EXISTS idx_entitlements_email
  ON app.user_entitlements(user_email);
```

No manual `ALTER TABLE` in any environment — every schema change = new versioned migration file.

---

### 3. Run migrations against each endpoint

After `bundle deploy -t <target>`, apply DDL to that workspace's endpoint:

```bash
flyway \
  -url="jdbc:postgresql://<stage-endpoint-host>/databricks_postgres" \
  -user=<token> \
  -password=<pat> \
  -locations="filesystem:./migrations" \
  migrate
```

Flyway creates `flyway_schema_history` in the target DB — tracks applied versions. Adding `V004__...` later runs only the new script.

**Auth tip:** prefer service principal OAuth for CI; use `WorkspaceClient().postgres.generate_database_credential(endpoint=...)` for app/runtime connections ([Apps tutorial](https://docs.databricks.com/aws/en/oltp/projects/tutorial-databricks-apps-autoscaling)).

---

### 4. CI/CD pipeline (recommended)

```
Developer adds migration file
        ↓
PR opens → CI creates temp Lakebase branch (ci-pr-123)
         → applies migrations → runs tests
        ↓
Merge to main
        ↓
deploy-dev:   bundle deploy -t dev   + migrate → dev endpoint
        ↓
Manual approval gate
        ↓
deploy-stage: bundle deploy -t stage + migrate → stage endpoint
        ↓
Manual approval gate
        ↓
deploy-prod:  bundle deploy -t prod  + migrate → prod endpoint
```

**PR branch pattern** (inside dev workspace): fork `ci-pr-<N>` from parent branch → apply migrations → post schema diff on PR → delete branch on merge. See [Evolutionary DB blog](https://www.databricks.com/blog/enabling-evolutionary-database-development-database-branching-lakebase-part-2).

---

### 5. Migration tool alternatives

| Tool | Language | Notes |
|------|----------|-------|
| **Flyway** | Java/CLI | Most common for Postgres |
| **Alembic** | Python | Pairs well with FastAPI / SQLAlchemy API |
| **sqitch** | Perl/CLI | Git-native migration tracking |
| **dbmate** | Go/CLI | Lightweight, easy in CI |

For a Databricks-heavy Python shop, **Alembic** pairs naturally with the API layer.

---

### 6. Environment map (healthcare E2E example)

| Environment | Workspace | Lakebase project | Branch | Data |
|-------------|-----------|------------------|--------|------|
| Dev | `haredecodes` | `healthcare-ops` | `production` (+ optional `dev-test` sandboxes) | test/dev rows |
| Stage | `haredecodes_stage` | `healthcare-ops` | `production` | stage-like / limited seed |
| Prod | `haredecodes_prod` | `healthcare-ops` | `production` | live operational data |

Same migration files run in all three; only endpoint host and workspace change.

---

### One-line summary

```text
bundle deploy -t stage  →  creates empty infra in stage workspace
flyway migrate          →  applies DDL to stage endpoint
Schema arrives. Dev data stays in dev.
```

---

## Databricks Apps + Lakebase (POC done)

- Template: **Lakebase Autoscaling app** (Streamlit todos) — code lands in **workspace**, not your GitHub repo
- **GitHub link** only needed to **install template** from Databricks Git catalog; runtime uses workspace + Lakebase env vars
- Resource: project + branch + `databricks_postgres` → service principal + `PGHOST`, `PGUSER`, `ENDPOINT_NAME`, etc.
- Connection in code: `WorkspaceClient().postgres.generate_database_credential(endpoint=...)` ([tutorial](https://docs.databricks.com/aws/en/oltp/projects/tutorial-databricks-apps-autoscaling))
- App schema: `{app-name}_schema_{id}.todos` (template may show `todos` under `public` depending on setup)

**POC outcome:** App writes ↔ same rows in Lakebase **Tables** — proves App → Postgres only (shallow Postgres learning).

**Cleanup:** Apps → Delete app; optional Lakebase → Delete project `postgres-test`.

---

## Features from docs (not covered in depth)

- [**Autoscaling**](https://docs.databricks.com/aws/en/oltp/projects/autoscaling) · [**Scale to zero**](https://docs.databricks.com/aws/en/oltp/projects/scale-to-zero)
- [**Branches**](https://docs.databricks.com/aws/en/oltp/projects/branches) · [**Instant restore**](https://docs.databricks.com/aws/en/oltp/projects/branches) (PITR branch from history)
- [**Read replicas**](https://docs.databricks.com/aws/en/oltp/projects/read-replicas) — shared storage, scale reads
- [**High availability**](https://docs.databricks.com/aws/en/oltp/projects/high-availability) — failover, same endpoint
- [**Connect**](https://docs.databricks.com/aws/en/oltp/projects/connect) — `psql`, drivers, SQL Editor
- [**Data API**](https://docs.databricks.com/aws/en/oltp/projects/data-api) — PostgREST-style HTTP (Autoscaling)
- [**Register in UC**](https://docs.databricks.com/aws/en/oltp/projects/register-uc) — query Lakebase from catalog
- [**Region availability**](https://docs.databricks.com/aws/en/oltp/instances/region-availability)
- [**Lakebase CDF**](https://docs.databricks.com/aws/en/oltp/projects/lakebase-cdf) — preview; needs `REPLICA IDENTITY FULL`, workspace preview flag

---

## Lakehouse vs Lakebase (one table)

| | **Delta / UC (lakehouse)** | **Lakebase (Postgres)** |
|--|---------------------------|-------------------------|
| Workloads | ETL, analytics, gold | Low-latency app reads/writes |
| Storage | Your cloud storage + UC | Databricks-managed per branch |
| Query engine | Spark / SQL warehouse | Postgres on Lakebase compute |
| Serve gold to app | Synced table or warehouse SQL | Synced table → Postgres |

---

## Deeper learning (recommended next)

1. SQL Editor: `CREATE TABLE`, indexes, `EXPLAIN ANALYZE`
2. **Child branch** → test DDL on branch, not `production`
3. **Synced table** from one **gold** table (Triggered after gold job)
4. Small **custom app** that only `SELECT`s synced data (write `app.py` once)
5. Add `postgres_*` resources to `databricks.yml` + `migrations/` folder → `bundle deploy -t dev` then migrate

---

## Quick links

- [Get started / create project](https://docs.databricks.com/aws/en/oltp/projects/)
- [Core concepts](https://docs.databricks.com/aws/en/oltp/projects/core-concepts)
- [Synced tables](https://docs.databricks.com/aws/en/oltp/projects/sync-tables)
- [Apps + Lakebase](https://docs.databricks.com/aws/en/oltp/projects/databricks-apps)
- [Manage with DAB](https://docs.databricks.com/aws/en/oltp/projects/manage-with-bundles)
- [Autoscaling by default / migration](https://docs.databricks.com/aws/en/oltp/upgrade-to-autoscaling)
