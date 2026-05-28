# Implementation Spec: Dev → Prod on Azure Databricks (DAB + GitHub Actions)

**Project:** Azure Databricks Real-World Healthcare End-to-End  
**Goal:** Two isolated Azure Databricks workspaces (dev, prod), two Git branches (`dev`, `main`), automated deploy via Databricks Asset Bundles (DAB) and GitHub Actions.  
**Audience:** You (fork owner) implementing after the YouTube course repo.

---

## 1. Target architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ GitHub (your fork)                                                          │
│   dev branch  ──push──►  deploy-dev.yml   ──►  Dev Databricks workspace     │
│   main branch ──push──►  deploy-prod.yml  ──►  Prod Databricks workspace  │
└─────────────────────────────────────────────────────────────────────────────┘
         │                              │
         │                              └── databricks bundle deploy -t {dev|prod}
         │
         └── Repo: notebooks + databricks.yml + resources/job.yml

Per environment (dev and prod), each with its own:
  • Resource group
  • Storage account (ADLS Gen2) + container + folder layout
  • Access Connector for Azure Databricks (managed identity)
  • Databricks workspace (Premium recommended for Unity Catalog)
  • Unity Catalog metastore (or shared metastore with separate catalogs)
  • Service principal (or PAT) for CI/CD only
```

### Branch → environment mapping

| Git branch | DAB target | Databricks workspace | Deploy trigger |
|------------|------------|----------------------|----------------|
| `dev`      | `dev`      | Existing dev workspace | Push to `dev` |
| `main`     | `prod`     | New prod workspace     | Push/merge to `main` |

### Promotion flow (recommended)

1. Develop notebooks in **dev workspace** (Git folder) → push to `dev` branch on GitHub.
2. Complete **Phase E** (DAB variables + job parameters) — see §8; use **E7 checklist** before prod.
3. Auto-deploy `dev` branch → dev workspace → run job, validate data.
4. Open PR: `dev` → `main` (review + optional manual approval in GitHub Environment).
5. Merge to `main` → auto-deploy to prod workspace (prod catalog/storage via DAB variables only).

---

## 2. Current repo state (what you inherited)

Your fork already has useful scaffolding but is **not yet multi-environment ready**:

| Item | Current state | Action needed |
|------|---------------|---------------|
| `databricks.yml` | `dev` and `prod` both point to **same** workspace host | Set distinct `host` per target |
| GitHub secrets | Single `DATABRICKS_HOST` + `DATABRICKS_TOKEN` | Use **GitHub Environments** with per-env secrets |
| Notebooks | Hardcoded `anirvandecodes` catalog and `anirvandecodesstorage` paths | Parameterize via bundle variables |
| `main` branch | Likely only branch today | Create `dev` branch; use `main` for prod only |
| Storage / UC | Author’s Azure resources | Provision **your** dev + prod stacks |

---

## 3. Naming conventions (pick once, use everywhere)

Use a consistent prefix, e.g. `hm` or `healthcare`:

| Resource | Dev example | Prod example |
|----------|-------------|--------------|
| Resource group | `rg-dbx-healthcare-dev` | `rg-dbx-healthcare-prod` |
| Storage account | `sthealthcaredev` (globally unique, 3–24 chars) | `sthealthcareprod` |
| Container | `data` | `data` |
| Databricks workspace | `dbw-healthcare-dev` | `dbw-healthcare-prod` |
| Access Connector | `ac-dbx-healthcare-dev` | `ac-dbx-healthcare-prod` |
| Unity Catalog catalog | `healthcare_dev` | `healthcare_prod` |
| UC schemas | `bronze`, `silver`, `gold` | same |
| DAB bundle name | `healthcare-end-to-end-project` (keep or rename) | same |

**Storage folder layout** (mirror the course; separate accounts per env):

```
data/                          # container
├── staging/
│   ├── diagnosis/
│   ├── hospitals/
│   ├── patients/
│   └── visits/                # upload raw_data/*.csv here
├── bronze/
│   ├── diagnosis_raw/  {checkpoint,schema}/
│   ├── hospital_raw/
│   ├── patient_raw/
│   └── visit_raw/
├── silver/
│   ├── dim_diagnosis/checkpoint/
│   ├── dim_hospital/checkpoint/
│   ├── dim_patient/checkpoint/
│   └── fact_visit/checkpoint/
└── (optional) gold/ checkpoints if needed later
```

---

## 4. Phase A — Azure foundation (Dev workspace)

You said dev workspace already exists. **Validate or complete** the following.

### A1. Resource group and storage

1. Azure Portal → Resource groups → create `rg-dbx-healthcare-dev` (region e.g. `eastus`).
2. Create storage account `sthealthcaredev`:
   - Performance: Standard
   - Redundancy: LRS (dev) / GRS or ZRS (prod, later)
   - **Hierarchical namespace: Enabled** (required for `abfss://`)
3. Create container `data`.
4. Create folder structure under `data` (Blob storage → containers → `data` → add virtual directories), or create folders on first upload.

### A2. Upload sample raw files

Copy from repo `raw_data/` to ADLS:

| Local file | ADLS path |
|------------|-----------|
| `diagnosis_raw.csv` | `staging/diagnosis/` |
| `hospital_raw.csv` | `staging/hospitals/` |
| `patient_raw.csv` | `staging/patients/` |
| `visit_raw.csv` | `staging/visits/` |

Use Azure Storage Explorer or `az storage blob upload-batch`.

### A3. Access Connector for Azure Databricks

1. In the **same** resource group, create **Access Connector for Azure Databricks** (`ac-dbx-healthcare-dev`).
2. Note the **managed identity** (Object ID) on the connector resource.
3. On storage account → **Access control (IAM)** → Add role assignment:
   - Role: **Storage Blob Data Contributor**
   - Assign to: managed identity of the Access Connector

This replaces legacy “mount” patterns for serverless / UC external locations.

### A4. Databricks workspace (dev)

If not already done:

1. Create workspace `dbw-healthcare-dev` in `rg-dbx-healthcare-dev`.
2. Pricing tier: **Premium** (Unity Catalog).
3. Deploy No Public IP / VNet only if your org requires it (optional for learning).

Record:

- Workspace URL: `https://adb-XXXXXXXX.XX.azuredatabricks.net`
- Workspace resource ID (for UC storage credentials)

### A5. Unity Catalog (dev)

1. Workspace → **Catalog** → enable Unity Catalog if prompted.
2. Create catalog `healthcare_dev`.
3. Create schemas: `bronze`, `silver`, `gold`.
4. **External location** (recommended):
   - Name: `healthcare_dev_adls`
   - URL: `abfss://data@sthealthcaredev.dfs.core.windows.net/`
   - Storage credential: Azure managed identity via Access Connector
5. Grant your user (and later SP) `USE CATALOG`, `USE SCHEMA`, `CREATE TABLE` on dev catalog.

### A6. Link workspace to storage (credential chain)

In Databricks (SQL or UI):

- Register storage credential pointing to Access Connector.
- Create external location on `abfss://data@sthealthcaredev.dfs.core.windows.net/`
- Ensure catalog default or notebooks use this location for checkpoints under `bronze/`, `silver/`.

---

## 5. Phase B — Azure foundation (Prod workspace)

Repeat Phase A in a **separate** resource group and workspace. Do **not** share storage between dev and prod.

| Step | Dev | Prod |
|------|-----|------|
| Resource group | `rg-dbx-healthcare-dev` | `rg-dbx-healthcare-prod` |
| Storage | `sthealthcaredev` | `sthealthcareprod` |
| Access Connector | `ac-dbx-healthcare-dev` | `ac-dbx-healthcare-prod` |
| Databricks | `dbw-healthcare-dev` | `dbw-healthcare-prod` |
| UC catalog | `healthcare_dev` | `healthcare_prod` |
| Data | Test/sample CSVs | Prod-like or subset; **never** point prod job at dev storage |

Optional hardening for prod:

- Disable public network access on storage / workspace
- Private endpoints
- Separate Azure subscription
- GitHub Environment protection rules (required reviewers before prod deploy)

---

## 6. Phase C — CI/CD identity (both environments)

**Prefer service principal (SP)** over personal PAT for GitHub Actions.

### Per workspace: create SP

**Option 1 — Databricks account/console (if you have account admin):**  
Create service principal, assign to workspace, generate OAuth secret.

**Option 2 — Azure AD app registration:**

1. App registration → `sp-dbx-healthcare-github-dev` (and `-prod`).
2. Enterprise application → assign to Databricks workspace (if using Azure AD SSO).
3. In Databricks: **Admin** → **Service principals** → add app, grant:
   - `Can Use` workspace
   - Unity Catalog: `ALL PRIVILEGES` on catalog (or minimal: USE CATALOG, CREATE TABLE, MODIFY on schemas for deploy user)

4. Generate **Databricks OAuth** or **PAT** for the SP (document which your org allows).

Store in GitHub (see Phase F):

- `DATABRICKS_HOST` (different per environment)
- `DATABRICKS_CLIENT_ID` + `DATABRICKS_CLIENT_SECRET` (OAuth) **or** `DATABRICKS_TOKEN` (PAT)

---

## 7. Phase D — Git branching strategy

### D1. Create `dev` branch from current `main`

```bash
git checkout main
git pull origin main
git checkout -b dev
git push -u origin dev
```

### D2. Branch protection (GitHub → Settings → Branches)

| Branch | Rules |
|--------|--------|
| `main` | Require PR from `dev`, 1 reviewer, no direct push (optional) |
| `dev` | Allow direct push for daily work |

### D3. Default branch

- Set **default branch** to `dev` if you want new clones to start in dev (optional).
- Keep **prod deploy** tied only to `main`.

---

## 8. Phase E — DAB variables & notebook parameterization (review at end)

> **When to do this:** After you finish editing and testing notebooks in the **dev** workspace (Git folder), push to GitHub, pull in Cursor, then complete this phase before enabling CI/CD to prod.  
> **Goal:** One notebook codebase; dev vs prod differ only in `databricks.yml` targets and GitHub secrets—not duplicate notebooks.

### Why variables (not hardcoded prod catalog in notebooks)

| Approach | Verdict |
|----------|--------|
| Same notebooks + DAB variables per target | **Recommended** |
| Separate `bronze_dev` / `bronze_prod` notebook copies | Avoid (drift) |
| Manually change catalog in prod workspace only | Avoid (breaks single source of truth) |

### Three-layer pattern

| Layer | Responsibility |
|-------|----------------|
| **1. `databricks.yml` targets** | Dev vs prod values for `catalog_name`, `storage_account`, `container_name`, optional path prefix |
| **2. `resources/job.yml`** | Pass `${var.*}` into each task as `base_parameters` |
| **3. Notebooks** | Read parameters at top; build `abfss://` URLs and `{catalog}.bronze.*` table names |

Prod never hardcodes `haredecodes_prod` in notebook bodies—only the **prod** target in `databricks.yml` sets it.

---

### E0 — Your environment values (fill in when implementing)

| Variable | Dev (`-t dev`) | Prod (`-t prod`) |
|----------|----------------|------------------|
| `catalog_name` | `haredecodes` | `haredecodes_prod` |
| `storage_account` | `haredecodesnew` | `haredecodesnewprod` |
| `container_name` | `data` | `data` |
| `raw_path_prefix` | `landing` (or `staging` if you align folders) | same as dev |
| Workspace host | `https://adb-7405614871562851.11.azuredatabricks.net` | prod workspace URL from Azure Overview |
| Resource group | `haredecodes` | `haredecodes-prod` |

**Keep the same in both envs (do not variable-ize):** schema names `bronze`, `silver`, `gold`; notebook filenames; job task keys.

**Do not deploy to Databricks:** `docs/**` (this spec), `DEPLOYMENT.md` — excluded via `sync.exclude` in `databricks.yml`.

---

### E1 — Edit notebooks in dev first (current step)

1. In **dev** workspace Git folder, replace author defaults (`anirvandecodes`, `anirvandecodesstorage`) with **dev** values from the table above.
2. Use **`abfss://`** paths only (never `https://...blob.core.windows.net/...`).
3. Test each notebook or the full job in dev.
4. **Commit & push** from Databricks Git UI to GitHub branch **`dev`**.
5. **`git pull`** in Cursor so local repo matches GitHub.

---

### E2 — Add bundle variables in `databricks.yml`

Define shared variables and **per-target overrides**:

```yaml
variables:
  catalog_name:
    description: Unity Catalog name
  storage_account:
    description: ADLS storage account name (no protocol)
  container_name:
    default: data
  raw_path_prefix:
    default: landing
    description: Folder under container for raw CSVs (landing or staging)

targets:
  dev:
    mode: development
    workspace:
      host: https://adb-7405614871562851.11.azuredatabricks.net
    variables:
      catalog_name: haredecodes
      storage_account: haredecodesnew
      raw_path_prefix: landing

  prod:
    mode: production
    workspace:
      host: https://adb-<PROD_WORKSPACE_ID>.azuredatabricks.net
      root_path: /Workspace/.bundle/${bundle.name}/${bundle.target}
    variables:
      catalog_name: haredecodes_prod
      storage_account: haredecodesnewprod
      raw_path_prefix: landing
```

Ensure `sync.exclude` includes `docs/**` so this spec is not uploaded on deploy.

---

### E3 — Pass variables into every job task (`resources/job.yml`)

On **each** notebook task, add `base_parameters` (names must match notebook widgets/params):

```yaml
base_parameters:
  catalog_name: ${var.catalog_name}
  storage_account: ${var.storage_account}
  container_name: ${var.container_name}
  raw_path_prefix: ${var.raw_path_prefix}
```

When the job runs, DAB injects dev or prod values based on `-t dev` / `-t prod`.

---

### E4 — Notebook pattern (all 9 notebooks)

At the top of the first Python cell in each notebook:

```python
dbutils.widgets.text("catalog_name", "haredecodes")
dbutils.widgets.text("storage_account", "haredecodesnew")
dbutils.widgets.text("container_name", "data")
dbutils.widgets.text("raw_path_prefix", "landing")

catalog = dbutils.widgets.get("catalog_name")
storage = dbutils.widgets.get("storage_account")
container = dbutils.widgets.get("container_name")
raw_prefix = dbutils.widgets.get("raw_path_prefix")

base = f"abfss://{container}@{storage}.dfs.core.windows.net"
source_path = f"{base}/{raw_prefix}/diagnosis/"   # adjust per notebook entity
checkpoint_path = f"{base}/bronze/diagnosis_raw/checkpoint/"
schema_location = f"{base}/bronze/diagnosis_raw/schema/"

spark.sql(f"CREATE SCHEMA IF NOT EXISTS {catalog}.bronze")
# ...
# .toTable(f"{catalog}.bronze.diagnosis_raw")
```

- **Widget defaults** = dev values so interactive runs in dev Git folder still work without the job.
- **Job / bundle deploy** overrides widgets via `base_parameters` for prod.

**Files to update:** all under `bronze/` (4), `silver/` (4), `gold/resubmission_analysis.ipynb` (1).

---

### E5 — SQL cells

Replace hardcoded three-level names with dynamic SQL in Python:

```python
spark.sql(f"SELECT * FROM {catalog}.bronze.diagnosis_raw")
```

---

### E6 — Validate locally before CI

```bash
databricks bundle validate -t dev
databricks bundle deploy -t dev
databricks bundle run end_to_end_healthcare_project_job -t dev

# After prod Azure + secrets are ready:
databricks bundle validate -t prod
databricks bundle deploy -t prod
```

Confirm dev job uses `haredecodesnew` and prod deploy uses `haredecodesnewprod` (query tables or paths in run output).

---

### E7 — Phase E review checklist (run before merging to `main`)

- [ ] No remaining `anirvandecodes` or `anirvandecodesstorage` in repo (`grep` in Cursor)
- [ ] All notebooks use widgets + `catalog` / `base` variables
- [ ] `databricks.yml` dev and prod hosts are **different** workspace URLs
- [ ] `databricks.yml` variable table matches E0
- [ ] `resources/job.yml` every task has `base_parameters` for all four variables
- [ ] `sync.exclude` has `docs/**` (spec stays on GitHub only)
- [ ] Dev job run succeeds against `haredecodes` + `haredecodesnew`
- [ ] Prod storage has same folder layout + sample CSVs as dev
- [ ] Prod catalog `haredecodes_prod` + external location on `haredecodesnewprod` exist
- [ ] Ready for GitHub Environment secrets (`DATABRICKS_HOST` per env)

---

### E8 — What we are not doing

| Item | Note |
|------|------|
| Git folder in **prod** workspace | Prod uses **GitHub Actions + bundle deploy** only |
| Variable for UC external location **name** | Build paths from `storage_account`; UC locations are separate admin setup |
| Duplicate notebooks per environment | Single repo on `dev` branch → promote to `main` |

---

## 9. Phase F — DAB configuration (final `databricks.yml`)

### F1. Targets (required)

| Setting | `dev` target | `prod` target |
|---------|--------------|---------------|
| `mode` | `development` | `production` |
| `workspace.host` | Dev URL | Prod URL |
| `workspace.root_path` | Default (user prefix in dev mode) | `/Workspace/.bundle/.../prod` |
| `variables` | dev catalog + storage | prod catalog + storage |

`development` mode prefixes deployed resources with `[dev your_user]` and allows faster iteration. `production` mode enforces stable paths and is appropriate for `main`.

### F2. Job resource

Keep `resources/job.yml` with relative notebook paths (`../bronze/...`) and `source: WORKSPACE` — DAB syncs notebooks on deploy.

Optional additions:

```yaml
# resources/job.yml (prod-oriented)
resources:
  jobs:
    end_to_end_healthcare_project_job:
      name: healthcare-e2e-${bundle.target}
      email_notifications:
        on_failure:
          - your-email@domain.com
      # schedule only in prod:
      # schedule:
      #   quartz_cron_expression: '0 0 6 * * ?'
      #   timezone_id: America/New_York
```

Use [bundle overrides](https://docs.databricks.com/dev-tools/bundles/settings.html) or separate `resources/job-dev.yml` / `job-prod.yml` if schedules differ.

### F3. Local validation

```bash
# Install CLI: https://docs.databricks.com/dev-tools/cli/databricks-cli.html
databricks auth login --host https://adb-<DEV>.azuredatabricks.net

databricks bundle validate -t dev
databricks bundle deploy -t dev
databricks bundle run end_to_end_healthcare_project_job -t dev
```

Repeat with `-t prod` after prod workspace exists.

---

## 10. Phase G — GitHub Actions

### G1. Use GitHub Environments (not repo-level secrets only)

Create environments: **dev**, **production**

| Environment | Secrets |
|-------------|---------|
| `dev` | `DATABRICKS_HOST`, `DATABRICKS_TOKEN` (or CLIENT_ID/SECRET) |
| `production` | Same names, **prod workspace values** |

This allows identical workflow YAML with different secret values per environment.

### G2. `deploy-dev.yml` (target file: `.github/workflows/deploy-dev.yml`)

```yaml
name: Deploy to Dev

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

      - name: Validate bundle
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: databricks bundle validate -t dev

      - name: Deploy bundle
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: databricks bundle deploy -t dev
```

### G3. `deploy-prod.yml`

```yaml
name: Deploy to Prod

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production   # add required reviewers in GitHub UI
    steps:
      - uses: actions/checkout@v4

      - uses: databricks/setup-cli@main

      - name: Validate bundle
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: databricks bundle validate -t prod

      - name: Deploy bundle
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: databricks bundle deploy -t prod
```

### G4. Optional: run job after deploy

```yaml
      - name: Run healthcare job
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: databricks bundle run end_to_end_healthcare_project_job -t dev
```

Run on dev only first; prod may be deploy-only until data is validated.

### G5. OAuth variant (recommended long-term)

If using SP OAuth, replace token env with:

```yaml
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_CLIENT_ID: ${{ secrets.DATABRICKS_CLIENT_ID }}
          DATABRICKS_CLIENT_SECRET: ${{ secrets.DATABRICKS_CLIENT_SECRET }}
```

And configure CLI auth per [Databricks CI/CD docs](https://docs.databricks.com/dev-tools/bundles/ci-cd.html).

---

## 11. Phase H — End-to-end verification checklist

### Dev (`dev` branch)

- [ ] Push to `dev` triggers GitHub Action successfully
- [ ] Bundle deploy creates/updates job in dev workspace
- [ ] Job tasks reference correct notebook paths under `.bundle` or workspace sync path
- [ ] Auto Loader reads from `abfss://data@haredecodesnew.../landing/...` (or `staging/...`)
- [ ] Tables exist: `haredecodes.bronze.*`, `silver.*`, `gold.hospital_disease_kpi`
- [ ] Phase E checklist (§8 E7) completed
- [ ] No references to `anirvandecodes` or `anirvandecodesstorage` remain

### Prod (`main` branch)

- [ ] Merge `dev` → `main` triggers prod workflow
- [ ] Prod job runs against `haredecodesnewprod` and `haredecodes_prod` catalog only
- [ ] Prod GitHub Environment requires approval (if configured)
- [ ] Dev and prod jobs can run concurrently without shared storage

---

## 12. Implementation order (suggested timeline)

| Order | Task | Est. effort |
|-------|------|-------------|
| 1 | Complete dev Azure storage + connector + folders + upload CSVs | 2–4 h |
| 2 | Unity Catalog + external location on `haredecodesnew` (dev) | 1–2 h |
| 3 | Edit/test notebooks in dev Git folder; push to GitHub `dev`; pull in Cursor | 2–4 h |
| 4 | **Phase E (§8):** DAB variables, job `base_parameters`, E7 checklist | 2–4 h |
| 5 | Local `bundle validate/deploy/run` on dev | 1 h |
| 6 | GitHub Environment `dev`, secrets, workflow | 1 h |
| 7 | Provision prod Azure + Databricks + UC (Phase B) | 3–5 h |
| 8 | SP for prod, GitHub Environment `production`, prod host in `databricks.yml` | 1–2 h |
| 9 | PR `dev` → `main`, validate prod deploy (Phase H) | 1 h |
| 10 | Update `DEPLOYMENT.md` / README with your hosts and naming | 30 min |

---

## 13. Files to create or modify (summary)

| File | Action |
|------|--------|
| `databricks.yml` | Distinct hosts; `variables` + targets; `sync.exclude` for `docs/**` |
| `resources/job.yml` | Add `base_parameters` on every task (Phase E3) |
| `bronze/*.ipynb` (4) | Parameterize paths + catalog |
| `silver/*.ipynb` (4) | Parameterize |
| `gold/resubmission_analysis.ipynb` | Parameterize |
| `.github/workflows/deploy-dev.yml` | Add `environment: dev`, `bundle validate` |
| `.github/workflows/deploy-prod.yml` | Add `environment: production`, `bundle validate` |
| `DEPLOYMENT.md` | Document your secrets, environments, SP setup |
| `docs/IMPLEMENTATION_SPEC.md` | This document |

**Do not commit:** tokens, client secrets, `.databricks/` profile with secrets. Ensure `.gitignore` covers them.

---

## 14. Risks and mitigations

| Risk | Mitigation |
|------|------------|
| Same secret used for dev and prod | GitHub Environments with separate secret values |
| Prod job reads dev storage | Separate storage accounts; variables only set in `databricks.yml` targets |
| `development` mode renames resources | Expected in dev; use `production` mode on `main` |
| Hardcoded author paths in notebooks | Grep for `anirvandecodes` before every release |
| PAT expiry | Use SP + OAuth; rotate secrets on calendar |
| Unity Catalog not enabled | Premium workspace + metastore assignment |

---

## 15. Reference links

- [Databricks Asset Bundles](https://docs.databricks.com/dev-tools/bundles/index.html)
- [Bundle configuration (targets, variables)](https://docs.databricks.com/dev-tools/bundles/settings.html)
- [CI/CD for bundles](https://docs.databricks.com/dev-tools/bundles/ci-cd.html)
- [Access Connector for Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/connect/storage/azure-access-connectors)
- [Unity Catalog external locations](https://docs.databricks.com/data-governance/unity-catalog/manage-external-locations-and-credentials.html)

---

## 16. Quick reference: abfss URL template

```
abfss://<container>@<storage_account>.dfs.core.windows.net/<path>/

Dev:  abfss://data@haredecodesnew.dfs.core.windows.net/landing/diagnosis/
Prod: abfss://data@haredecodesnewprod.dfs.core.windows.net/landing/diagnosis/
```

---

## 17. Phase E at a glance (bookmark)

| Step | Action |
|------|--------|
| E0 | Confirm dev/prod values table (§8) |
| E1 | Notebooks in dev → push → pull Cursor |
| E2 | `databricks.yml` variables + targets |
| E3 | `job.yml` `base_parameters` |
| E4–E5 | Notebook widgets + dynamic SQL |
| E6 | `bundle validate/deploy/run` |
| E7 | **Review checklist before `main`** |

---

*Spec version: 1.1 — added Phase E variable workflow (haredecodes naming), docs exclude, E7 review checklist.*
