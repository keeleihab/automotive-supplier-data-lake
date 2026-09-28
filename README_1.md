# Cross-Plant Scrap Data Lake

An end-to-end data platform on **Azure** and **Databricks** that consolidates scrap data from two manufacturing plants, which record it differently, into one comparable KPI.

> **Note:** all data is **simulated**. The scenario models a multi-plant automotive supplier (cable protection conduits, fluid lines, thermal management tubing). It was built as a learning project to practice the Azure data engineering stack.

---

## The problem

Two plants make the same parts but record production and scrap differently, so headquarters cannot answer a simple question: **which plant wastes more, and on which parts?**

| | Plant MA | Plant DE |
|---|---|---|
| Source system | Azure SQL Database | CSV exports |
| Column names | English | German (`Materialnummer`, `Menge_Stk`, `Ausschuss_m`) |
| Part codes | `MA-xxxxx` | `FR-xxxx` |
| Scrap unit | **pieces** | **meters** |
| Format | Standard | `;` separator, `dd.MM.yyyy` dates, decimal comma |
| Data quality | Clean | Duplicate rows, unknown part codes |

## Key findings

Once both plants were harmonized onto one definition:

- **Plant MA runs 2.80% scrap vs 1.80% at Plant DE, about 56% higher.**
- The gap is concentrated in **fluid lines**: 3.55% vs 1.78%.
- The single biggest driver is the **heated SCR line (6 mm)**: 5.88% vs 3.07%, peaking at 8.35% in one week.
- Plant MA is also **less stable**: weekly scrap ranges from 2.3% to 3.2%, vs 1.7% to 2.1% at Plant DE.
- Plant MA does **better** on one part (thermal module connector tube: 2.37% vs 2.62%), a practice Plant DE could learn from.
- If Plant MA matched Plant DE's rate part by part, **about 37% of its scrap would be avoidable** (roughly 2,000 pieces over 6 weeks, about 17,600 per year).

**Recommendation:** start root cause analysis on the SCR and PA12 fuel lines at Plant MA (extrusion parameters, material lots, shifts), benchmarked against Plant DE.

![Scrap rate by product family](images/scrap_by_family.png)
![Scrap rate by part](images/scrap_by_part.png)

---

## Architecture

```
Plant MA (Azure SQL) ──┐                                 ┌─ bronze/plant_ma/<table>/yyyy/MM/dd/*.parquet
                       ├── Azure Data Factory ──► ADLS ──┤
Plant DE (CSV files) ──┘   daily trigger, 06:00          └─ bronze/plant_de/yyyy/MM/dd/*.csv
                                                                        │
                                                          Databricks (PySpark + Delta Lake)
                                                   silver_production · silver_exceptions_unmapped_parts
                                                                        │
                                                              gold_scrap_kpi_weekly
                                                                        │
                                                          Business insights notebook (charts)
```

| Component | Azure resource | Purpose |
|---|---|---|
| Data lake | ADLS Gen2 (`stplantlakeihab02`), hierarchical namespace | `landing`, `bronze`, `silver`, `gold` layers |
| Plant MA source | Azure SQL Database (`plant_ma_db`) | Simulated plant system with `ModifiedDate` on every table |
| Orchestration | Azure Data Factory (`adf-plantlake-ihab02`) | Ingestion pipelines, trigger, monitoring |
| Secrets | Azure Key Vault | SQL password; never stored in pipelines |
| Transformation | Databricks (PySpark, Spark SQL, Delta Lake) | Silver and gold layers, insights |
| Source control | GitHub | Feature branches, pull requests, ARM templates |

Region: **Germany West Central**.

---

## Ingestion: Azure Data Factory

### `pl_ingest_plant_ma`: metadata-driven incremental load
1. **Lookup** reads `etl.ControlTable`: which tables to load, and each one's last watermark.
2. **ForEach** table (sequential):
   - **Lookup** the new watermark: `MAX(ModifiedDate)`
   - **Copy** only rows where `old watermark < ModifiedDate <= new watermark` → bronze as **Parquet**, partitioned by table and load date
   - **Stored procedure** `etl.usp_UpdateWatermark` advances the watermark, **only after a successful copy**
3. Adding a table means adding a row to the control table, not building a new pipeline.

**Tested:** the first run loaded everything (108 orders). After adding a new week and correcting one old order, the second run copied **exactly 19 and 18 changed rows**, and **0** from the unchanged tables.

### `pl_ingest_plant_de`: file ingestion
Copies new CSV files from `landing/plant_de` to bronze **unchanged** (binary copy, for auditability), under a load-date folder, then deletes them from landing so **each file is processed exactly once**.

### `pl_master_daily`: orchestration
Runs both plant pipelines **in parallel** with *Execute Pipeline* (*Wait on completion* on, so a child failure fails the master). Scheduled by a **daily trigger at 06:00**. An **Azure Monitor alert** emails on any failed run.

### Security
- The SQL password lives only in **Key Vault**.
- Data Factory authenticates with its **system-assigned managed identity**:
  - **Storage Blob Data Contributor** on the lake
  - **Key Vault Secrets User** on the vault
- The SQL server firewall allows **selected networks only**: an admin IP plus the trusted Azure services exception.

---

## Transformation: Databricks

### Silver: harmonized, validated data
- **Plant MA:** keeps only the **latest version** of each record (window function on `ModifiedDate`), so corrections replace old values; aggregates scrap per order.
- **Plant DE:**
  - translates German columns
  - parses `dd.MM.yyyy` dates and decimal commas
  - removes **duplicate** export rows
  - maps part codes through the **crosswalk**
  - converts scrap from **meters to pieces** using each part's length
- Unknown part codes (`FR-9999`) go to **`silver_exceptions_unmapped_parts`** instead of being silently dropped.

### Gold: business-ready KPIs
`gold_scrap_kpi_weekly`: scrap rate and PPM per plant, part and week, on one definition.

### Corrections and audit trail
Plant DE's correction file is applied with a **Delta `MERGE`** (upsert: existing orders updated, new ones inserted). **`DESCRIBE HISTORY`** and **time travel** (`VERSION AS OF`) show exactly what changed and when.

---

## Data dictionary

### `silver_production`: one row per production order
| Column | Type | Description |
|---|---|---|
| plant | string | `MA` or `DE` |
| order_id | string | Production order number from the source plant |
| part_id | string | Harmonized part ID (Plant MA code system) |
| production_date | date | Production date |
| qty_produced | int | Total pieces produced, **including** scrap |
| scrap_pcs | double | Scrapped pieces. Plant DE is converted from meters: `scrap_m / LengthPerPiece_m` |
| line | string | Production line (Plant DE only) |
| source | string | `plant_ma_sql` or `plant_de_csv` |

### `silver_exceptions_unmapped_parts`
Plant DE rows whose material number is not in the crosswalk. Kept for the data owner to fix; excluded from KPIs.

### `gold_scrap_kpi_weekly`: one row per plant × week × part
| Column | Description |
|---|---|
| plant, week_start, product_family, part_id, part_name | Grouping keys (weeks start on Monday) |
| qty_produced, scrap_pcs | Weekly totals |
| scrap_rate_pct | `scrap_pcs / qty_produced × 100` |
| ppm | `scrap_pcs / qty_produced × 1,000,000` |

## Business glossary

| Term | Definition |
|---|---|
| **Scrap** | Pieces produced that fail quality and cannot be delivered, counted in **pieces** for all plants |
| **Scrap rate** | Scrap pieces ÷ total pieces produced, always calculated from totals, never by averaging percentages |
| **PPM** | Defective parts per million = scrap rate × 1,000,000 |
| **Crosswalk** | Master-data table mapping each plant's part code to one harmonized part ID |
| **Watermark** | The highest `ModifiedDate` already loaded for a table; the next run copies only newer rows |

## Data quality checks (on every silver load)
- No duplicate orders per plant
- All dates parsed
- Scrap never greater than production
- Unmapped part codes counted and routed to the exceptions table

---

## Runbook: a pipeline failed

1. **Alert email** → Data Factory **Monitor** → open the failed run.
2. **Drill down** from the master pipeline to the child pipeline to the failing activity, and read the error.
3. **Assess impact:** which plants or tables did not load, and which reports are stale.
4. **Fix the root cause:** source offline, credentials, schema change, or a wrong control-table entry.
5. **Rerun.** This is safe: watermarks only advance after a successful copy, and silver deduplicates by key.
6. **Record** the incident, and add a check to prevent it.

**Tested:** a non-existent table was added to the control table. The master pipeline failed and the root cause (*Invalid object name*) was traced. The other tables' watermarks were unaffected. It was fixed by deactivating the row (`IsActive = 0`), with **no pipeline change**.

---

## Problems solved during the build

| Problem | Cause | Fix |
|---|---|---|
| Deployment blocked | An **Azure Policy** restricted the allowed regions | Read the policy assignment; redeployed in Germany West Central |
| Database refused connections | Public network access disabled | Firewall set to selected networks, plus the trusted Azure services exception |
| Storage connection test failed | Managed identity missing its role | Assigned **Storage Blob Data Contributor** |
| **Watermarks advanced but no data was copied** | Activity dependencies set to *on skip* instead of *on success* | Fixed the dependencies in the pipeline JSON; reset the watermarks and **backfilled** |
| Files landed at the container root with random names | Stale dataset definition in the browser session | Corrected and saved the sink dataset path; reloaded the studio |
| Master pipeline reported success while a child failed | *Wait on completion* was off | Enabled it, so failures surface at the top |

**Main lesson:** a green "Succeeded" status is not proof. Verify row counts, and make the watermark depend strictly on a successful copy. A silent failure is worse than a loud one.

---

## Limitations and production design

| In this project | In production |
|---|---|
| Databricks **Free Edition**; bronze files moved manually | **Azure Databricks** reading ADLS through a Unity Catalog **external location** |
| Notebook run by hand | Data Factory **Databricks Notebook activity** runs it right after ingestion |
| Watermark on `ModifiedDate` (does not detect deletes) | **Change data capture (CDC)** for deletes |
| Public endpoints with firewall rules | **Private endpoints**, Entra ID authentication |
| Documentation in this README | Datasets registered in **Microsoft Purview** / Unity Catalog |
| Cloud source only | **Self-hosted integration runtime** for on-premises plant databases |

---

## Repository structure

```
adf/                      Data Factory pipelines, datasets, linked services (JSON, via Git integration)
sql/
  01_plant_ma_setup.sql   Plant MA source tables and data
  02_etl_control.sql      Control table and watermark stored procedure
  04_simulate_new_data.sql  New week plus a correction, for the incremental test
sample_data/              Plant DE CSV exports (German format)
databricks/
  03_databricks_silver_gold.py   Bronze → silver → gold, MERGE, time travel
  04_business_insights.py        Charts and findings
images/                   Screenshots
```

The `adf_publish` branch contains the **ARM templates** generated when publishing from `main`, used to deploy the factory to other environments.

## How to run
1. Run `sql/01_plant_ma_setup.sql`, then `sql/02_etl_control.sql`, in Azure SQL.
2. Upload `sample_data/*.csv` to `landing/plant_de/`.
3. Run `pl_master_daily` (or wait for the 06:00 trigger).
4. Load bronze into Databricks and run `databricks/03_databricks_silver_gold.py`, then `04_business_insights.py`.
