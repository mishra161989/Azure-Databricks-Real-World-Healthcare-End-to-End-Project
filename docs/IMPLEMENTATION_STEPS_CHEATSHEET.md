# Databricks E2E — Steps Cheatsheet

Start → end. Do **dev first**, then repeat Azure/UC/storage for **prod**.

---

## 1. Azure (per environment)

1. Create resource group  
2. Create ADLS Gen2 storage + container `data`  
3. Upload CSVs → `data/staging/{diagnosis,hospitals,patients,visits}/`  
4. Create Access Connector  
5. Grant connector MI → **Storage Blob Data Contributor** on storage  
6. Create Databricks workspace (Premium) → save workspace URL  

---

## 2. Unity Catalog (per environment)

1. Create catalog (e.g. `haredecodes` / `haredecodes_prod`)  
2. Create schemas `bronze`, `silver`, `gold` *(or let notebooks create them)*  
3. Create **storage credential** → link Access Connector  
4. Create **external location** → `abfss://data@{storage}.dfs.core.windows.net/`  
5. Test read from notebook: `dbutils.fs.ls("abfss://...")`  

---

## 3. Repo & branches

1. Fork/clone repo  
2. Create branch **`dev`** from `main`**  
3. Parameterize notebooks: widgets `catalog_name`, `storage_account`, `container_name`, `raw_path_prefix`  
4. Each layer notebook: `CREATE SCHEMA IF NOT EXISTS {catalog}.bronze|silver|gold`  
5. Silver streams: `.start().awaitTermination()` on foreachBatch writes  

---

## 4. Databricks Asset Bundle

1. `databricks.yml`: variables + **`dev`** target (`mode: development`, **no** `root_path`)  
2. `databricks.yml`: **`prod`** target (`mode: production`, `root_path`, `run_as`)  
3. `sync.exclude`: `docs/**` (docs in GitHub only)  
4. `resources/job.yml`: `source: WORKSPACE`, bundle paths `../bronze|silver|gold/*.ipynb`  
5. All tasks: `base_parameters` → `${var.*}`  
6. **No** `git_source` on job  
7. Local: `databricks bundle validate -t dev`  

---

## 5. GitHub CI/CD

1. Enable Actions on repo  
2. Create environments: **`dev`**, **`prod`**  
3. Generate PAT in **dev workspace** → scopes: `bundle`, `workspace`, `jobs`  
4. Generate PAT in **prod workspace** (separate token)  
5. Env **`dev`** secrets: `DATABRICKS_HOST`, `DATABRICKS_TOKEN`  
6. Env **`prod`** secrets: `DATABRICKS_HOST`, `DATABRICKS_TOKEN`  
7. Workflows: `deploy-dev.yml` (push `dev`), `deploy-prod.yml` (push `main`)  

---

## 6. Dev run

1. Push to **`dev`** → CI deploys bundle  
2. Open job in dev workspace → confirm params (`haredecodes`, `haredecodesnew`, `staging`)  
3. **Run now** → all tasks green  
4. Validate: `SELECT COUNT(*)` on bronze / silver / gold tables  

---

## 7. Prod promote

1. PR **`dev` → `main`** → merge  
2. CI deploys prod bundle  
3. Confirm prod job params (`haredecodes_prod`, `haredecodesnewprod`)  
4. **Run now** (or repair failed tasks)  
5. Validate SQL on `haredecodes_prod.*`  

---

## 8. If something fails (quick fixes)

| Issue | Fix |
|-------|-----|
| Silver empty in prod job | Add `awaitTermination()`; drop table + delete checkpoint; re-run |
| Gold schema error | `CREATE SCHEMA IF NOT EXISTS {catalog}.gold` in gold notebook |
| CI dev deploy error | Remove `permissions` block; no `root_path` on dev target |
| Rows read = 0 | Delete silver checkpoint or reload bronze |
| Reset silver greenfield | `DROP TABLE` + delete `.../silver/*/checkpoint/` |

---

## 9. Ongoing workflow

```text
edit on dev → push dev → deploy → test job → PR to main → deploy prod → run prod job
```

---

*See `DATABRICKS_PROJECT_BLUEPRINT.md` for details.*
