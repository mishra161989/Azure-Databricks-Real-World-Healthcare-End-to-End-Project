# Databricks E2E Project Blueprint (Azure + Unity Catalog + DAB + CI/CD)

**Purpose:** Reusable checklist and reference for implementing a medallion pipeline on Azure Databricks—from workspace creation through prod deployment and job execution.

**Based on:** Healthcare E2E project (`haredecodes` dev / `haredecodes-prod` prod).

**Related doc:** [`IMPLEMENTATION_SPEC.md`](IMPLEMENTATION_SPEC.md) has deeper phase-by-phase detail; this file is the **operational blueprint** you run for each new project.

---

## Table of contents

1. [Architecture at a glance](#1-architecture-at-a-glance)
2. [Phase 0 — Naming and planning](#2-phase-0--naming-and-planning)
3. [Phase 1 — Azure foundation (per environment)](#3-phase-1--azure-foundation-per-environment)
4. [Phase 2 — Unity Catalog](#4-phase-2--unity-catalog)
5. [Phase 3 — Storage access (Access Connector + UC)](#5-phase-3--storage-access-access-connector--uc)
6. [Phase 4 — Repository and notebooks](#6-phase-4--repository-and-notebooks)
7. [Phase 5 — Databricks Asset Bundle (DAB)](#7-phase-5--databricks-asset-bundle-dab)
8. [Phase 6 — GitHub Actions CI/CD](#8-phase-6--github-actions-cicd)
9. [Phase 7 — Dev validation](#9-phase-7--dev-validation)
10. [Phase 8 — Promote to prod](#10-phase-8--promote-to-prod)
11. [Notebook standards (medallion)](#11-notebook-standards-medallion)
12. [Job design checklist](#12-job-design-checklist)
13. [Troubleshooting playbook](#13-troubleshooting-playbook)
14. [Dev vs prod — why behavior differs](#14-dev-vs-prod--why-behavior-differes)
15. [Quick reference commands](#15-quick-reference-commands)

---

## 1. Architecture at a glance

```
GitHub
  dev branch  ──push──►  deploy-dev.yml   ──►  bundle deploy -t dev   ──►  Dev workspace
  main branch ──push──►  deploy-prod.yml  ──►  bundle deploy -t prod  ──►  Prod workspace

Per environment (separate Azure stack):
  Resource group
  ADLS Gen2 storage account + container (data/)
  Access Connector (managed identity) ──RBAC──► Storage
  Databricks workspace (Premium / UC-capable)
  Unity Catalog catalog (e.g. haredecodes / haredecodes_prod)
  Schemas: bronze, silver, gold

Shared (common in one subscription/region):
  Unity Catalog metastore (optional but typical for dev+prod)
```

**Code promotion model (recommended):**

- **Do not** rely on job-level `git_source` for prod if you use DAB deploy.
- **Do:** `databricks bundle deploy` syncs notebooks to workspace; job uses `source: WORKSPACE` + bundle-relative paths.
- **Branches:** `dev` → dev workspace; `main` → prod workspace.

---

## 2. Phase 0 — Naming and planning

Pick names once; use everywhere (Azure, UC, `databricks.yml`, docs).

| Layer | Dev example (this project) | Prod example |
|-------|---------------------------|--------------|
| Resource group | `haredecodes` | `haredecodes-prod` |
| Databricks workspace | `haredecodes` | `haredecodes-prod` |
| Storage account | `haredecodesnew` | `haredecodesnewprod` |
| Container | `data` | `data` |
| Access connector | `haredecodes-access-connector` | `haredecodes-prod-access-connector` |
| UC catalog | `haredecodes` | `haredecodes_prod` |
| UC schemas | `bronze`, `silver`, `gold` | same |
| Raw CSV prefix | `staging/` (or `landing/`) | same |

**Storage layout:**

```text
abfss://data@{storage_account}.dfs.core.windows.net/
  staging/          # or landing/ — raw CSVs per entity
    diagnosis/
    hospitals/
    patients/
    visits/
  bronze/           # Auto Loader checkpoints + schema (not UC tables)
    diagnosis_raw/checkpoint/
    diagnosis_raw/schema/
    ...
  silver/           # Structured streaming checkpoints
    dim_diagnosis/checkpoint/
    dim_hospital/checkpoint/
    ...
```

**Checklist**

- [ ] Dev and prod use **separate** resource groups, workspaces, storage accounts, catalogs
- [ ] Raw path prefix documented (`staging` vs `landing`)
- [ ] Git branches: `dev` (daily work), `main` (prod releases only)

---

## 3. Phase 1 — Azure foundation (per environment)

Repeat for **dev**, then **prod**.

### 3.1 Resource group

- Create RG (e.g. `haredecodes-prod`)
- Region aligned with Databricks (e.g. East US)

### 3.2 Storage account (ADLS Gen2)

- **Standard** performance, **LRS** (or your standard)
- **Enable hierarchical namespace** (required for `abfss://`)
- Container: `data`
- Upload sample CSVs under `{raw_prefix}/diagnosis/`, etc.

### 3.3 Access Connector for Azure Databricks

- Create in the same RG
- Note: **No workspace name** is configured on the Azure resource—it only hosts a **managed identity**
- Copy the connector **resource ID** / identity for IAM

### 3.4 IAM on storage

- Access connector managed identity → **Storage Blob Data Contributor** on the storage account (or scoped container)

### 3.5 Databricks workspace

- **Premium** (Unity Catalog capable)
- Same region as storage
- Note workspace URL: `https://adb-xxxxxxxx.azuredatabricks.net`

**Checklist**

- [ ] ADLS Gen2 enabled
- [ ] Sample data in `data/{staging|landing}/...`
- [ ] Access connector MI has blob access on storage
- [ ] Workspace URL saved for GitHub secrets and `databricks.yml`

---

## 4. Phase 2 — Unity Catalog

In each workspace (or via account console if metastore is account-level):

### 4.1 Metastore

- Dev and prod workspaces often attach to **one regional metastore** (objects visible in both workspaces; isolate by **catalog** + **storage**)

### 4.2 Catalog (per environment)

```sql
CREATE CATALOG IF NOT EXISTS haredecodes_prod;
-- Grant USE CATALOG, CREATE SCHEMA to your user/group
```

### 4.3 Schemas

Create explicitly **or** create in notebooks (recommended: notebooks create what they need):

```sql
CREATE SCHEMA IF NOT EXISTS haredecodes_prod.bronze;
CREATE SCHEMA IF NOT EXISTS haredecodes_prod.silver;
CREATE SCHEMA IF NOT EXISTS haredecodes_prod.gold;
```

**Blueprint rule:** Every layer that writes tables must ensure schema exists:

| Layer | Notebook pattern |
|-------|------------------|
| Bronze | `CREATE SCHEMA IF NOT EXISTS {catalog}.bronze` |
| Silver | `CREATE SCHEMA IF NOT EXISTS {catalog}.silver` |
| Gold | `CREATE SCHEMA IF NOT EXISTS {catalog}.gold` |

**Checklist**

- [ ] Catalog exists per environment
- [ ] Bronze/silver/gold schemas exist or are created in code
- [ ] Your user can create tables in the catalog

---

## 5. Phase 3 — Storage access (Access Connector + UC)

### 5.1 How it connects (no workspace in Azure connector)

| Step | Where | What |
|------|--------|------|
| 1 | Azure | Access Connector = managed identity |
| 2 | Azure IAM | MI → Storage Blob Data Contributor on ADLS |
| 3 | Databricks UC | **Storage credential** points to that access connector |
| 4 | Databricks UC | **External location** = `abfss://data@{account}.dfs.core.windows.net/` + credential |
| 5 | Notebooks | Paths use `abfss://...`; UC enforces access via external location |

The **workspace** uses the metastore; Databricks control plane uses the registered identity—no workspace ID on the Azure connector.

### 5.2 External location URL

```text
abfss://data@haredecodesnewprod.dfs.core.windows.net/
```

(Trailing slash on folder-scoped paths; container root is fine for whole-container access.)

**Checklist**

- [ ] Storage credential created in UC
- [ ] External location created and accessible
- [ ] Test: `dbutils.fs.ls("abfss://data@...")` or read a sample file from a notebook

---

## 6. Phase 4 — Repository and notebooks

### 6.1 Repo layout (medallion + DAB)

```text
bronze/           # Auto Loader → Delta bronze tables
silver/           # Streaming bronze → silver (dims + facts)
gold/             # Analytics / KPI tables
resources/
  job.yml         # Lakeflow job definition
databricks.yml    # Bundle + dev/prod targets + variables
.github/workflows/
  deploy-dev.yml
  deploy-prod.yml
docs/             # Excluded from bundle sync (reference only)
```

### 6.2 Parameterize every notebook (widgets + job params)

**Cell 1 pattern:**

```python
dbutils.widgets.text("catalog_name", "haredecodes")
dbutils.widgets.text("storage_account", "haredecodesnew")
dbutils.widgets.text("container_name", "data")
dbutils.widgets.text("raw_path_prefix", "staging")

catalog = dbutils.widgets.get("catalog_name")
storage = dbutils.widgets.get("storage_account")
container = dbutils.widgets.get("container_name")
raw_prefix = dbutils.widgets.get("raw_path_prefix")

base = f"abfss://{container}@{storage}.dfs.core.windows.net"
```

**Never hardcode** catalog or storage in prod-bound code.

### 6.3 SQL in notebooks

Prefer parameterized SQL from Python:

```python
display(spark.sql(f"SELECT * FROM {catalog}.bronze.diagnosis_raw"))
```

**Checklist**

- [ ] All notebooks use widgets + `catalog` / `base`
- [ ] No author-specific catalog/storage left in repo
- [ ] Bronze/silver/gold schema creation where tables are written

---

## 7. Phase 5 — Databricks Asset Bundle (DAB)

### 7.1 `databricks.yml` essentials

```yaml
bundle:
  name: healthcare-end-to-end-project

sync:
  exclude:
    - docs/**

variables:
  catalog_name:
  storage_account:
  container_name:
    default: data
  raw_path_prefix:
    default: staging

targets:
  dev:
    mode: development
    workspace:
      host: https://adb-....azuredatabricks.net
    variables:
      catalog_name: haredecodes
      storage_account: haredecodesnew
      ...

  prod:
    mode: production
    workspace:
      host: https://adb-....azuredatabricks.net
      root_path: /Workspace/.bundle/${bundle.name}/${bundle.target}
    run_as:
      user_name: your-email@domain.com
    variables:
      catalog_name: haredecodes_prod
      storage_account: haredecodesnewprod
      ...
```

**Notes**

- **Dev `development` mode:** deploys under user path (`~/...`); do not force `/Workspace/.bundle/...` for dev.
- **Prod `production` mode:** use `root_path` under `/Workspace/.bundle/.../prod/`.
- Avoid `permissions` blocks on dev if CI fails with “Cannot apply local deployment permissions”.

### 7.2 `resources/job.yml` essentials

- `notebook_path: ../bronze/....ipynb` (relative to bundle)
- `source: WORKSPACE` (notebooks deployed by bundle, not runtime Git pull)
- **No `git_source`** on job if you use bundle deploy for prod safety
- `base_parameters` on **every** task:

```yaml
base_parameters:
  catalog_name: ${var.catalog_name}
  storage_account: ${var.storage_account}
  container_name: ${var.container_name}
  raw_path_prefix: ${var.raw_path_prefix}
```

**Checklist**

- [ ] `databricks bundle validate -t dev` passes locally
- [ ] `databricks bundle validate -t prod` passes
- [ ] Job task DAG matches dependency order (bronze → silver → gold)

---

## 8. Phase 6 — GitHub Actions CI/CD

This project uses **two workflows** in `.github/workflows/`:

| Workflow file | Trigger branch | GitHub Environment | DAB target |
|---------------|----------------|--------------------|------------|
| `deploy-dev.yml` | push to `dev` | **`dev`** | `-t dev` |
| `deploy-prod.yml` | push to `main` | **`prod`** | `-t prod` |

Both run: `databricks bundle validate` → `databricks bundle deploy`.

---

### 8.1 Enable GitHub Actions (fork)

1. GitHub repo → **Settings** → **Actions** → **General**
2. Allow actions (e.g. “Allow all actions and reusable workflows”)
3. Ensure workflow files exist on **`main`** (required for some UI features like manual run on default branch)

---

### 8.2 Create GitHub Environments (not repo-level secrets only)

Use **Environments** so the same workflow YAML works with different credentials per env.

1. GitHub repo → **Settings** → **Environments**
2. Create environment: **`dev`**
3. Create environment: **`prod`** (name must match `environment: prod` in `deploy-prod.yml`)

**Optional (recommended for prod):**

- On **`prod`** environment → **Required reviewers** (manual approval before prod deploy)

---

### 8.3 Create Databricks PAT (one per workspace)

Create a **separate** token in **each** workspace (dev PAT ≠ prod PAT).

**In Databricks (dev workspace):**

1. User icon → **Settings** → **Developer** → **Access tokens** (or **Manage** → **Personal access tokens**)
2. **Generate new token**
3. Comment: e.g. `github-actions-dev`
4. Lifetime: per org policy (90 days, etc.)
5. Copy token **once** — you cannot view it again

**Repeat in prod workspace** with a different token (`github-actions-prod`).

**Minimum PAT scopes (API scopes):**

| Scope | Why |
|-------|-----|
| **`bundle`** | `bundle validate` / `bundle deploy` |
| **`workspace`** | Sync notebooks to workspace |
| **`jobs`** | Create/update job from `resources/job.yml` |

Add **`access-management`** only if your `databricks.yml` includes `permissions` blocks that CI must apply.

Do **not** use “All APIs” unless your org requires it.

---

### 8.4 Store secrets in GitHub Environments

Add secrets to **each environment** separately. Use **Secrets**, not **Variables** (tokens must be secret).

**Environment `dev`:**

| Secret name | Value |
|-------------|--------|
| `DATABRICKS_HOST` | Dev workspace URL, e.g. `https://adb-7405614871562851.11.azuredatabricks.net` |
| `DATABRICKS_TOKEN` | Dev PAT from step 8.3 |

**Environment `prod`:**

| Secret name | Value |
|-------------|--------|
| `DATABRICKS_HOST` | Prod workspace URL, e.g. `https://adb-7405616194048266.6.azuredatabricks.net` |
| `DATABRICKS_TOKEN` | Prod PAT from step 8.3 |

**How to add:**

1. **Settings** → **Environments** → click **`dev`**
2. **Environment secrets** → **Add secret**
3. Add `DATABRICKS_HOST`, then `DATABRICKS_TOKEN`
4. Repeat for **`prod`** with prod workspace URL and prod PAT

**Important:**

- Secret names are **exact** — workflows use `${{ secrets.DATABRICKS_HOST }}` and `${{ secrets.DATABRICKS_TOKEN }}`
- Do **not** put catalog name or storage account in GitHub secrets — those live in `databricks.yml` `variables` per target
- Host URL has **no** trailing slash

---

### 8.5 What CI deploys (and what it does not)

| Deployed by CI | Not deployed by CI |
|----------------|-------------------|
| Notebooks (`bronze/`, `silver/`, `gold/`) | Azure storage, access connector, RG |
| `resources/job.yml` (Lakeflow job) | Unity Catalog catalog/schemas (created in Azure/Databricks UI or notebooks) |
| Bundle metadata under `/Workspace/.bundle/...` (prod) or `~/...` (dev) | Raw CSV files on ADLS (upload separately) |

`docs/**` is excluded from bundle sync — blueprint stays in GitHub only.

---

### 8.6 Workflow pattern (reference)

**Dev** (`.github/workflows/deploy-dev.yml`):

```yaml
on:
  push:
    branches: [dev]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: dev

    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main
      - run: databricks bundle validate -t dev
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
      - run: databricks bundle deploy -t dev
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
```

**Prod** (`.github/workflows/deploy-prod.yml`): same pattern with `environment: prod`, `-t prod`, trigger on `main`.

---

### 8.7 CI/CD promotion flow

```text
1. Push to dev branch
      → deploy-dev.yml runs
      → bundle deploy -t dev
      → test job in dev workspace

2. PR dev → main → merge
      → deploy-prod.yml runs
      → bundle deploy -t prod
      → run job in prod workspace
```

---

### 8.8 CI troubleshooting

| Error | Fix |
|-------|-----|
| `Invalid access token` / 403 | Regenerate PAT; check scopes (`bundle`, `workspace`, `jobs`) |
| Wrong workspace deployed | `DATABRICKS_HOST` in wrong environment — dev secrets must be dev URL |
| `Cannot apply local deployment permissions` | Remove `permissions` block from dev target in `databricks.yml` |
| Dev deploy path error | Do not set `root_path` on `development` mode target |
| Workflow not visible / no “Run workflow” | Workflow must exist on default branch (`main`); or push to trigger |
| Environment not found | GitHub environment name must match YAML exactly (`dev`, `prod`) |

---

### 8.9 CI/CD checklist

- [ ] Actions enabled on GitHub repo
- [ ] GitHub Environment **`dev`** with `DATABRICKS_HOST` + `DATABRICKS_TOKEN`
- [ ] GitHub Environment **`prod`** with prod host + prod PAT
- [ ] PAT created **in dev workspace** for dev; **in prod workspace** for prod
- [ ] PAT scopes: `bundle`, `workspace`, `jobs`
- [ ] `deploy-dev.yml` triggers on `dev` branch
- [ ] `deploy-prod.yml` triggers on `main` branch
- [ ] First green deploy-dev on push to `dev`
- [ ] First green deploy-prod after merge to `main`
- [ ] Optional: required reviewers on **`prod`** environment

---

## 9. Phase 7 — Dev validation

### 9.1 Deploy and run

1. Push to `dev` → CI deploys bundle
2. Open job `azure-e2e-project-anirban` (or your job name) in **dev** workspace
3. Confirm task parameters: `haredecodes`, `haredecodesnew`, `staging`
4. **Run now** (full job)

### 9.2 Validate data

```sql
SELECT COUNT(*) FROM haredecodes.bronze.diagnosis_raw;
SELECT COUNT(*) FROM haredecodes.silver.dim_diagnosis;
SELECT COUNT(*) FROM haredecodes.gold.hospital_disease_kpi;
```

### 9.3 Greenfield test in dev (recommended before trusting dev)

Dev can look “green” with **warm state** (tables/schemas already exist). For a honest test:

```sql
DROP TABLE IF EXISTS haredecodes.silver.dim_diagnosis;
```

Delete `abfss://data@haredecodesnew.../silver/dim_diagnosis/checkpoint/` then re-run silver task.

**Checklist**

- [ ] Full job succeeds in dev
- [ ] Row counts sensible in bronze/silver/gold
- [ ] Optional: greenfield silver test passed

---

## 10. Phase 8 — Promote to prod

### 10.1 Promotion flow

```text
dev branch (tested) → PR dev → main → merge → deploy-prod → run prod job
```

1. Open PR `dev` → `main`
2. Review; merge
3. Confirm **Deploy to Prod** workflow succeeded
4. Prod job parameters must show `haredecodes_prod`, `haredecodesnewprod`

### 10.2 Prod first run

- Ensure CSVs exist on **prod** storage under `staging/...`
- Run full job (bronze will load; silver/gold depend on upstream)
- Do **not** use dev Git folder in prod; bundle path is under `/Workspace/.bundle/.../prod/`

### 10.3 After failed prod runs

| Symptom | Action |
|---------|--------|
| Silver empty, checkpoint exists | Drop silver table + delete checkpoint folder; redeploy; re-run |
| Wrong catalog in logs | Fix job `base_parameters` / `databricks.yml` prod variables |
| Gold `SCHEMA_NOT_FOUND` | Add `CREATE SCHEMA IF NOT EXISTS {catalog}.gold` in gold notebook |

**Checklist**

- [ ] Prod deploy green
- [ ] All 9 tasks succeed (or your task count)
- [ ] SQL counts on `haredecodes_prod.*` expected

---

## 11. Notebook standards (medallion)

### 11.1 Bronze (Auto Loader)

```python
spark.sql(f"CREATE SCHEMA IF NOT EXISTS {catalog}.bronze")

source_path = f"{base}/{raw_prefix}/diagnosis/"
checkpoint_path = f"{base}/bronze/diagnosis_raw/checkpoint/"
schema_location = f"{base}/bronze/diagnosis_raw/schema/"

df = spark.readStream.format("cloudFiles") \
    .option("cloudFiles.format", "csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .option("cloudFiles.schemaLocation", schema_location) \
    .load(source_path)

df.writeStream.format("delta") \
    .option("checkpointLocation", checkpoint_path) \
    .outputMode("append") \
    .trigger(availableNow=True) \
    .toTable(f"{catalog}.bronze.diagnosis_raw")
```

Bronze `toTable` often completes without explicit `awaitTermination` in jobs, but silver **foreachBatch** requires it (see below).

### 11.2 Silver (streaming + foreachBatch merge)

**Required for job reliability:**

```python
(
    df.writeStream
        .foreachBatch(merge_fn)
        .outputMode("update")
        .trigger(availableNow=True)
        .option("checkpointLocation", checkpoint_path)
        .start()
        .awaitTermination()   # REQUIRED in jobs
)
```

Without `awaitTermination()`, the job task can exit while the stream is still starting—silver tables never created in prod (proven via A/B test).

**Merge pattern:**

```python
def merge_fn(batch_df, batch_id):
    if not spark.catalog.tableExists(silver_table):
        batch_df.write.format("delta").mode("overwrite").saveAsTable(silver_table)
        return
    DeltaTable.forName(spark, silver_table).alias("t") \
        .merge(batch_df.alias("s"), "t.key = s.key") \
        .whenMatchedUpdateAll() \
        .whenNotMatchedInsertAll() \
        .execute()
```

### 11.3 Gold

```python
spark.sql(f"CREATE SCHEMA IF NOT EXISTS {catalog}.gold")

# ... build gold_df ...

gold_df.write.mode("overwrite").saveAsTable(f"{catalog}.gold.hospital_disease_kpi")
```

---

## 12. Job design checklist

- [ ] One job; tasks for bronze → silver → gold with `depends_on`
- [ ] All tasks: same four `base_parameters`
- [ ] `environment_key: Default` (serverless) if used in dev
- [ ] Notebook paths bundle-relative: `../silver/dim_diagnosis_ingestion.ipynb`
- [ ] `source: WORKSPACE` only (no prod `git_source` on `dev` branch)
- [ ] Job name documented for dev vs prod (prod may lack `[dev user]` prefix)

**Task order (healthcare project):**

```text
diagnosis_raw → diagnosis_silver
hospital_raw → hospital_silver
patients_raw → patient_silver
visits_raw → fact_visit (depends on dims)
resubmission (gold, depends on fact_visit)
```

---

## 13. Troubleshooting playbook

| Issue | Likely cause | Fix |
|-------|----------------|-----|
| Prod silver missing; job “succeeded” early | No `awaitTermination()` | Add `.awaitTermination()` on foreachBatch streams |
| `Rows read = 0` | Checkpoint caught up; no new bronze | Delete checkpoint or add bronze data |
| `foreachBatch` prints not in logs | Serverless / Spark Connect | Use SQL row counts; checkpoints prove run |
| `SCHEMA_NOT_FOUND` gold | No `gold` schema in prod | `CREATE SCHEMA IF NOT EXISTS {catalog}.gold` |
| `TABLE_OR_VIEW_NOT_FOUND` gold | Silver failed upstream | Fix silver; re-run from silver |
| Dev works, prod fails, same code | Cold prod vs warm dev | Greenfield test; drop table + checkpoint |
| CI “Cannot apply local deployment permissions” | `permissions` in dev target | Remove dev permissions block |
| CI dev path error | `root_path` in development mode | Use default `~/` for dev; `root_path` prod only |
| Duplicate bronze rows | Manual load + Auto Loader | One source path; or dedupe in silver |
| Storage access denied | MI RBAC | Blob Data Contributor on storage for access connector |

---

## 14. Dev vs prod — why behavior differs

| Topic | Dev | Prod |
|-------|-----|------|
| Catalog | `haredecodes` | `haredecodes_prod` |
| Storage | `haredecodesnew` | `haredecodesnewprod` |
| Schemas/tables | Often created during exploration | Created only by pipelines |
| Checkpoints | Many reruns | Fresh or fewer runs |
| Job wait | Notebook may feel “fine” without `awaitTermination` | Job exits early without wait |
| UC metastore | May be shared | Same metastore; different catalog |
| Git folder in workspace | Optional for interactive dev | Bundle deploy path only |

**Rule:** Validate **greenfield** in dev (drop table + checkpoint) before assuming dev success proves prod.

---

## 15. Quick reference commands

### Local bundle

```bash
databricks bundle validate -t dev
databricks bundle validate -t prod
databricks bundle deploy -t dev
databricks bundle deploy -t prod
```

### Git

```bash
git checkout dev
git add ... && git commit -m "..." && git push origin dev

# After PR merge to main, prod deploy runs automatically
```

### Prod validation SQL

```sql
SELECT COUNT(*) FROM haredecodes_prod.bronze.diagnosis_raw;
SELECT COUNT(*) FROM haredecodes_prod.silver.dim_diagnosis;
SELECT COUNT(*) FROM haredecodes_prod.gold.hospital_disease_kpi;
```

### Reset silver for rerun

```sql
DROP TABLE IF EXISTS haredecodes_prod.silver.dim_diagnosis;
```

Delete ADLS folder: `silver/dim_diagnosis/checkpoint/` (and `checkpoint/_test` if used).

---

## Appendix — Files in this project

| File | Role |
|------|------|
| `databricks.yml` | Bundle name, dev/prod hosts, variables |
| `resources/job.yml` | 9-task DAG, base_parameters |
| `.github/workflows/deploy-dev.yml` | Push `dev` → deploy dev |
| `.github/workflows/deploy-prod.yml` | Push `main` → deploy prod |
| `bronze/*.ipynb` | Auto Loader ingestion |
| `silver/*.ipynb` | Streaming merges + `awaitTermination` |
| `gold/resubmission_analysis.ipynb` | Gold KPI + `CREATE SCHEMA gold` |

---

## New project starter checklist (one page)

- [ ] Azure: RG, ADLS, access connector, IAM, Databricks workspace (×2 for dev/prod)
- [ ] UC: catalog + bronze/silver/gold schemas
- [ ] UC: storage credential + external location per storage account
- [ ] Repo: widgets, parameterized paths, schema creation per layer
- [ ] DAB: `databricks.yml` variables + dev/prod targets
- [ ] Job: WORKSPACE paths, base_parameters, DAG, **silver `awaitTermination`**
- [ ] GitHub: `dev` + `prod` environments, PATs, workflows
- [ ] Dev: push, deploy, full job, SQL counts
- [ ] Prod: merge `main`, deploy, full job, SQL counts
- [ ] Document environment-specific names in `databricks.yml` only (not in notebooks)

---

*Last updated from healthcare E2E implementation (Azure Databricks, Unity Catalog, DAB, GitHub Actions).*
