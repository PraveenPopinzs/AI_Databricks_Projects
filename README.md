# AI_Databricks_Projects

A **Medallion Architecture** data engineering project on Databricks that starts from raw CRM and ERP CSV files in a **Unity Catalog Volume**, ingests them into the **Bronze** layer, cleanses and standardises them through the **Silver** layer with **Spark SQL**, and builds a **Gold** star schema (dimensions + fact) — orchestrated end-to-end by a Databricks **Job**, and ready to connect to **Power BI** for visualization.

> **Note:** The full project pipeline lives on the `feature/medallion-architecture` branch. The `Main` branch contains the README and the Gold Dim Date notebook.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Repository Structure](#repository-structure)
3. [Source Data — Unity Catalog Volume](#source-data--unity-catalog-volume)
4. [Bronze Layer — Ingestion](#bronze-layer--ingestion)
5. [Silver Layer — Transformation (Spark SQL)](#silver-layer--transformation-spark-sql)
   - 5.1 [CRM Transformations](#crm-transformations)
   - 5.2 [ERP Transformations](#erp-transformations)
   - 5.3 [Silver Pipeline Orchestration](#silver-pipeline-orchestration)
6. [Gold Layer — Dimensional Modelling (Spark SQL)](#gold-layer--dimensional-modelling-spark-sql)
   - 6.1 [dim_customer](#dim_customer)
   - 6.2 [dim_product](#dim_product)
   - 6.3 [dim_date](#dim_date)
   - 6.4 [fact_sales](#fact_sales)
   - 6.5 [Gold Pipeline Orchestration](#gold-pipeline-orchestration)
7. [All Spark SQL — Combined (Silver + Gold)](#all-spark-sql--combined-silver--gold)
8. [Job Orchestration & Scheduling](#job-orchestration--scheduling)
9. [Data Flow Summary](#data-flow-summary)
10. [Connect to Power BI](#connect-to-power-bi)

---

## Project Overview

This project implements a **Medallion Architecture** (Bronze → Silver → Gold) on Databricks to build an analytics-ready sales data warehouse from two source systems: **CRM** and **ERP**.

| Aspect | Details |
|---|---|
| **Architecture** | Medallion (Bronze / Silver / Gold) |
| **Source Systems** | CRM (customer, product, sales) + ERP (customer demographics, locations, product categories) |
| **Target Catalog** | `workspace` (Unity Catalog) |
| **Schemas** | `workspace.bronze`, `workspace.silver`, `workspace.gold` |
| **Branches** | `Main` (README + dim_date) · `feature/medallion-architecture` (full pipeline) |
| **Transformation Language** | Spark SQL (`CREATE OR REPLACE TABLE`, `MERGE INTO`) |
| **Orchestration** | Databricks Job **Jobs for Sales Data** — bronze → silver → gold tasks |
| **Visualization** | Gold star schema ready to connect to Power BI |

---

## Repository Structure

```
AI_Databricks_Projects/
├── README.md
├── Data/
│   ├── source_crm/                # CRM source CSVs
│   │   ├── cust_info.csv
│   │   ├── prd_info.csv
│   │   └── sales_details.csv
│   └── source_erp/                # ERP source CSVs
│       ├── CUST_AZ12.csv
│       ├── LOC_A101.csv
│       └── PX_CAT_G1V2.csv
├── Scripts/
│   ├── bronze/
│   │   └── Bronze Layer.ipynb              # CSV → Delta ingestion
│   ├── silver/
│   │   ├── CRM/
│   │   │   ├── Silver CRM Customer Info.ipynb
│   │   │   ├── Silver CRM Product Info.ipynb
│   │   │   └── Silver CRM Sales Details.ipynb
│   │   ├── ERP/
│   │   │   ├── Silver ERP Customer AZ12.ipynb
│   │   │   ├── Silver ERP Customer Location.ipynb
│   │   │   └── Silver ERP Product Category.ipynb
│   │   └── Silver Pipeline Orchestration.ipynb
│   └── gold/
│       ├── Gold Dim Customer.ipynb
│       ├── Gold Dim Product.ipynb
│       ├── Gold Dim Date.ipynb
│       ├── Gold Fact Sales.ipynb
│       └── Gold Pipeline Orchestration.ipynb
```

---

## Source Data — Unity Catalog Volume

The pipeline starts in the **Unity Catalog Volume** `workspace.bronze.source_system` (`/Volumes/workspace/bronze/source_system/`), which holds the raw source files in two subdirectories:

- `source_crm/` — CRM CSV files: `cust_info.csv`, `prd_info.csv`, `sales_details.csv`
- `source_erp/` — ERP CSV files: `CUST_AZ12.csv`, `LOC_A101.csv`, `PX_CAT_G1V2.csv`

Six CSV files from these two source systems are ingested from the volume into the Bronze layer:

| # | Source System | File | Bronze Table | Rows |
|---|---|---|---|---|
| 1 | CRM | `cust_info.csv` | `workspace.bronze.cust_info` | 18,494 |
| 2 | CRM | `prd_info.csv` | `workspace.bronze.prd_info` | 397 |
| 3 | CRM | `sales_details.csv` | `workspace.bronze.sales_details` | 60,398 |
| 4 | ERP | `CUST_AZ12.csv` | `workspace.bronze.CUST_AZ12` | 18,484 |
| 5 | ERP | `LOC_A101.csv` | `workspace.bronze.LOC_A101` | 18,484 |
| 6 | ERP | `PX_CAT_G1V2.csv` | `workspace.bronze.PX_CAT_G1V2` | 37 |

---

## Bronze Layer — Ingestion

**Notebook:** `Scripts/bronze/Bronze Layer.ipynb`

Reads raw CSV files from UC Volumes (`/Volumes/workspace/bronze/source_system/`) and writes them as Delta tables in `workspace.bronze`.

- **CRM ingestion** — Reads `cust_info`, `prd_info`, and `sales_details` CSVs and writes as Delta tables with `overwrite` mode.
- **ERP ingestion** — Auto-discovers all `.csv` files in the `source_erp` volume directory and writes each as a Delta table.

---

## Silver Layer — Transformation (Spark SQL)

Cleans, standardises, and enriches Bronze data into readable Silver tables in `workspace.silver`. **Every transformation is written in Spark SQL** (`CREATE OR REPLACE TABLE ... AS SELECT` with `TRIM`, `CASE WHEN`, `COALESCE`, `REGEXP_REPLACE`, `TO_DATE`, `GREATEST`) — see [All Spark SQL — Combined](#all-spark-sql--combined-silver--gold).

### CRM Transformations

| Notebook | Bronze → Silver Table | Key Transformations |
|---|---|---|
| Silver CRM Customer Info | `cust_info` → `customer_info` | Column renaming, whitespace trimming, full name concatenation, marital status & gender mapping |
| Silver CRM Product Info | `prd_info` → `product_info` | Column renaming, trimming, product line mapping (R→Road, S→Sport, etc.), NULL cost handling |
| Silver CRM Sales Details | `sales_details` → `sales_details` | Column renaming, integer date → DATE type conversion, NULL/negative sales handling |

### ERP Transformations

| Notebook | Bronze → Silver Table | Key Transformations |
|---|---|---|
| Silver ERP Customer AZ12 | `CUST_AZ12` → `customer_info1` | Column renaming, customer ID cleaning (strip `NAS` prefix), gender standardisation |
| Silver ERP Customer Location | `LOC_A101` → `customer_info2` | Column renaming, customer ID cleaning (strip dashes), country standardisation (USA→United States, DE→Germany) |
| Silver ERP Product Category | `PX_CAT_G1V2` → `product_info2` | Column renaming, whitespace trimming |

### Silver Pipeline Orchestration

**Notebook:** `Scripts/silver/Silver Pipeline Orchestration.ipynb`

Runs all six Silver notebooks in sequence. Can be added as a pipeline task to build the entire Silver layer in one step.

---

## Gold Layer — Dimensional Modelling (Spark SQL)

Joins and enriches Silver tables into star-schema dimensions and a fact table in `workspace.gold`. **All joins and builds are written in Spark SQL** (`LEFT JOIN`, `ROW_NUMBER`, `QUALIFY`, `REGEXP_EXTRACT`, `MERGE INTO`) — see [All Spark SQL — Combined](#all-spark-sql--combined-silver--gold).

### dim_customer

**Notebook:** `Scripts/gold/Gold Dim Customer.ipynb`

Three-way LEFT JOIN of CRM `customer_info` + ERP `customer_info1` + ERP `customer_info2` into a unified customer dimension, implemented as **SCD Type 2** to preserve change history.

- Combines CRM demographics (name, marital status, gender) with ERP data (birth date, country)
- All customer IDs normalised to `AW000xxxxx` format for direct equality joins
- `customer_sk` surrogate PK (unique across all versions); `start_date` / `end_date` / `is_current` validity columns
- Deduplicates the silver source: one row per `customer_id` (latest `create_date`), NULL IDs excluded
- Incremental refresh: `MERGE` closes outdated versions, then `INSERT` adds new current versions
- Downstream joins should use `customer_sk` or filter `is_current = true`

### dim_product

**Notebook:** `Scripts/gold/Gold Dim Product.ipynb`

LEFT JOIN of CRM `product_info` with ERP `product_info2` into a unified product dimension.

- Strips category/subcategory prefix from `product_key` using `REGEXP_EXTRACT` for clean join keys
- Enriches products with category and subcategory information

### dim_date

**Notebook:** `Scripts/gold/Gold Dim Date.ipynb` *(Main branch)*

Creates `workspace.gold.dim_date` (1,826 rows) in pure SQL with an auto-generated integer `date_key` (YYYYMMDD format) covering 2010-01-01 to 2014-12-31 — the full range of dates in `fact_sales`.

- Columns: `date_key`, `full_date`, `year`, `quarter`, `month`, `month_name`, `day_of_week`, `day_of_month`, `day_of_year`, `day_name`, `week_of_year`, `year_month`, `is_weekday`, `is_weekend`
- Joins to `fact_sales` on `full_date` = `order_date` / `ship_date` / `due_date` (or on `date_key` = YYYYMMDD int of those dates)

### fact_sales

**Notebook:** `Scripts/gold/Gold Fact Sales.ipynb`

Builds `workspace.gold.fact_sales` from `workspace.silver.sales_details`, enriched with surrogate keys from `dim_customer` and `dim_product`.

| Source Table | Role | Join Key |
|---|---|---|
| `silver.sales_details` | Base fact grain (one row per order line) | — |
| `gold.dim_customer` | Adds `customer_id` surrogate key | `customer_id` |
| `gold.dim_product` | Adds `product_id` surrogate key | `product_key` |

### Gold Pipeline Orchestration

**Notebook:** `Scripts/gold/Gold Pipeline Orchestration.ipynb`

Runs all Gold notebooks in order: dimensions first (can run in parallel), then the fact table.

| Step | Notebook | Output Table | Depends On |
|---|---|---|---|
| 1 | Gold Dim Customer | `gold.dim_customer` | Silver customer tables |
| 2 | Gold Dim Product | `gold.dim_product` | Silver product tables |
| 3 | Gold Fact Sales | `gold.fact_sales` | dim_customer + dim_product |

---

## All Spark SQL — Combined (Silver + Gold)

Every Silver and Gold transformation is plain Spark SQL. The statements below are the complete, combined script — run them in order (e.g. in one SQL notebook) to rebuild the entire Silver and Gold layers from Bronze:

```sql
-- ══════════════════════════════════════
-- SILVER LAYER (6 tables)
-- ══════════════════════════════════════

-- 1. CRM: cust_info → silver.customer_info
CREATE OR REPLACE TABLE workspace.silver.customer_info AS
SELECT
    cst_id                              AS customer_id,
    cst_key                             AS customer_key,
    TRIM(cst_firstname)                 AS first_name,
    TRIM(cst_lastname)                  AS last_name,
    CONCAT(TRIM(cst_firstname), ' ', TRIM(cst_lastname)) AS full_name,
    CASE
        WHEN cst_marital_status = 'M' THEN 'Married'
        WHEN cst_marital_status = 'S' THEN 'Single'
        ELSE cst_marital_status
    END                                 AS marital_status,
    CASE
        WHEN cst_gndr = 'M' THEN 'Male'
        WHEN cst_gndr = 'F' THEN 'Female'
        ELSE cst_gndr
    END                                 AS gender,
    cst_create_date                     AS create_date
FROM workspace.bronze.cust_info;

-- 2. CRM: prd_info → silver.product_info
CREATE OR REPLACE TABLE workspace.silver.product_info AS
SELECT
    prd_id                              AS product_id,
    prd_key                             AS product_key,
    REGEXP_REPLACE(SUBSTRING(prd_key, 1, 5), '-', '_') AS Category_key,
    TRIM(prd_nm)                        AS product_name,
    COALESCE(prd_cost, 0)               AS cost,
    CASE TRIM(prd_line)
        WHEN 'R' THEN 'Road'
        WHEN 'S' THEN 'Sport'
        WHEN 'M' THEN 'Mountain'
        WHEN 'T' THEN 'Touring'
        ELSE 'Unknown'
    END                                 AS product_line,
    prd_start_dt                        AS start_date,
    prd_end_dt                          AS end_date,
    prd_end_dt IS NULL                  AS is_active
FROM workspace.bronze.prd_info;

-- 3. CRM: sales_details → silver.sales_details
CREATE OR REPLACE TABLE workspace.silver.sales_details AS
SELECT
    sls_ord_num                         AS order_number,
    sls_prd_key                         AS product_key,
    sls_cust_id                         AS customer_id,
    CASE
        WHEN LENGTH(CAST(sls_order_dt AS STRING)) = 8
        THEN TO_DATE(CAST(sls_order_dt AS STRING), 'yyyyMMdd')
    END                                 AS order_date,
    TO_DATE(CAST(sls_ship_dt AS STRING), 'yyyyMMdd') AS ship_date,
    TO_DATE(CAST(sls_due_dt AS STRING), 'yyyyMMdd')  AS due_date,
    GREATEST(COALESCE(sls_sales, 0), 0) AS sales_amount,
    sls_quantity                        AS quantity,
    GREATEST(COALESCE(sls_price, 0), 0) AS price
FROM workspace.bronze.sales_details;

-- 4. ERP: cust_az12 → silver.customer_info1
CREATE OR REPLACE TABLE workspace.silver.customer_info1 AS
SELECT
    REGEXP_REPLACE(TRIM(CID), '^NAS', '') AS customer_id,
    BDATE                               AS birth_date,
    CASE TRIM(GEN)
        WHEN 'Male'   THEN 'Male'
        WHEN 'M'      THEN 'Male'
        WHEN 'Female' THEN 'Female'
        WHEN 'F'      THEN 'Female'
        ELSE 'Unknown'
    END                                 AS gender
FROM workspace.bronze.cust_az12;

-- 5. ERP: loc_a101 → silver.customer_info2
CREATE OR REPLACE TABLE workspace.silver.customer_info2 AS
SELECT
    REGEXP_REPLACE(TRIM(CID), '-', '')  AS customer_id,
    CASE
        WHEN TRIM(CNTRY) IN ('USA', 'US') THEN 'United States'
        WHEN TRIM(CNTRY) = 'DE'          THEN 'Germany'
        WHEN TRIM(CNTRY) = ''            THEN 'Unknown'
        WHEN CNTRY IS NULL               THEN 'Unknown'
        ELSE TRIM(CNTRY)
    END                                 AS country
FROM workspace.bronze.loc_a101;

-- 6. ERP: px_cat_g1v2 → silver.product_info2
CREATE OR REPLACE TABLE workspace.silver.product_info2 AS
SELECT
    TRIM(ID)            AS category_id,
    TRIM(CAT)           AS category,
    TRIM(SUBCAT)        AS subcategory,
    TRIM(MAINTENANCE)   AS maintenance_required
FROM workspace.bronze.px_cat_g1v2;

-- ══════════════════════════════════════
-- GOLD LAYER (dimensions + fact)
-- ══════════════════════════════════════

-- 7. dim_customer — INITIAL SCD2 LOAD (run once; rebuilds with SCD2 columns)
CREATE OR REPLACE TABLE workspace.gold.dim_customer
COMMENT 'Gold-layer customer dimension (SCD Type 2). customer_sk is the surrogate PK; customer_id is the business key, unique among current rows.'
AS
SELECT
    CAST(ROW_NUMBER() OVER (ORDER BY customer_id) AS INT) AS customer_sk,
    customer_id, customer_key, first_name, last_name, full_name,
    birth_date, gender, marital_status, country, create_date,
    current_date()      AS start_date,
    CAST(NULL AS DATE)  AS end_date,
    true                AS is_current
FROM (
    SELECT
        ci.customer_id, ci.customer_key, ci.first_name, ci.last_name, ci.full_name,
        ci1.birth_date, ci.gender, ci.marital_status, ci2.country, ci.create_date
    FROM workspace.silver.customer_info ci
    LEFT JOIN workspace.silver.customer_info2 ci2 ON ci2.customer_id = ci.customer_key
    LEFT JOIN workspace.silver.customer_info1 ci1 ON ci1.customer_id = ci.customer_key
    WHERE ci.customer_id IS NOT NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY ci.customer_id ORDER BY ci.create_date DESC) = 1
);

-- 7a. dim_customer — INCREMENTAL STEP 1/2: close outdated current versions
MERGE INTO workspace.gold.dim_customer AS d
USING (
    SELECT
        ci.customer_id, ci.customer_key, ci.first_name, ci.last_name, ci.full_name,
        ci1.birth_date, ci.gender, ci.marital_status, ci2.country
    FROM workspace.silver.customer_info ci
    LEFT JOIN workspace.silver.customer_info2 ci2 ON ci2.customer_id = ci.customer_key
    LEFT JOIN workspace.silver.customer_info1 ci1 ON ci1.customer_id = ci.customer_key
    WHERE ci.customer_id IS NOT NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY ci.customer_id ORDER BY ci.create_date DESC) = 1
) AS s
    ON d.customer_id = s.customer_id
   AND d.is_current = true
WHEN MATCHED AND (
        NOT (d.customer_key    <=> s.customer_key)
     OR NOT (d.first_name     <=> s.first_name)
     OR NOT (d.last_name      <=> s.last_name)
     OR NOT (d.full_name      <=> s.full_name)
     OR NOT (d.birth_date     <=> s.birth_date)
     OR NOT (d.gender         <=> s.gender)
     OR NOT (d.marital_status <=> s.marital_status)
     OR NOT (d.country        <=> s.country)
) THEN UPDATE SET
    d.is_current = false,
    d.end_date   = current_date();

-- 7b. dim_customer — INCREMENTAL STEP 2/2: insert new current versions
INSERT INTO workspace.gold.dim_customer
SELECT
    (SELECT COALESCE(MAX(customer_sk), 0) FROM workspace.gold.dim_customer)
        + CAST(ROW_NUMBER() OVER (ORDER BY s.customer_id) AS INT) AS customer_sk,
    s.customer_id, s.customer_key, s.first_name, s.last_name, s.full_name,
    s.birth_date, s.gender, s.marital_status, s.country, s.create_date,
    current_date()      AS start_date,
    CAST(NULL AS DATE)  AS end_date,
    true                AS is_current
FROM (
    SELECT
        ci.customer_id, ci.customer_key, ci.first_name, ci.last_name, ci.full_name,
        ci1.birth_date, ci.gender, ci.marital_status, ci2.country, ci.create_date
    FROM workspace.silver.customer_info ci
    LEFT JOIN workspace.silver.customer_info2 ci2 ON ci2.customer_id = ci.customer_key
    LEFT JOIN workspace.silver.customer_info1 ci1 ON ci1.customer_id = ci.customer_key
    WHERE ci.customer_id IS NOT NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY ci.customer_id ORDER BY ci.create_date DESC) = 1
) s
LEFT JOIN workspace.gold.dim_customer d
    ON d.customer_id = s.customer_id
   AND d.is_current = true
WHERE d.customer_id IS NULL;

-- 8. dim_product — dedup to one row per product_key, enriched with ERP category
CREATE OR REPLACE TABLE workspace.gold.dim_product
COMMENT 'Gold-layer product dimension table joining CRM product_info and ERP product_info2 silver tables. Deduplicated to one row per product_key.'
AS
WITH product_enriched AS (
    SELECT
        pi.product_id,
        REGEXP_EXTRACT(pi.product_key, '^[^-]+-[^-]+-(.*)$', 1) AS product_key,
        pi.Category_key, pi.product_name, pi.cost, pi.product_line,
        pi.start_date, pi.end_date, pi.is_active,
        pi2.category, pi2.subcategory, pi2.maintenance_required,
        ROW_NUMBER() OVER (
            PARTITION BY REGEXP_EXTRACT(pi.product_key, '^[^-]+-[^-]+-(.*)$', 1)
            ORDER BY pi.is_active DESC, pi.start_date DESC
        ) AS rn
    FROM workspace.silver.product_info pi
    LEFT JOIN workspace.silver.product_info2 pi2
        ON pi.Category_key = pi2.category_id
)
SELECT
    product_id, product_key, Category_key, product_name, cost, product_line,
    start_date, end_date, is_active, category, subcategory, maintenance_required
FROM product_enriched
WHERE rn = 1;

-- 9. dim_date — 2010-01-01 to 2014-12-31, INT date_key in YYYYMMDD format
CREATE OR REPLACE TABLE workspace.gold.dim_date AS
WITH dates AS (
  SELECT explode(sequence(date '2010-01-01', date '2014-12-31', interval 1 day)) AS full_date
)
SELECT
  CAST(date_format(full_date, 'yyyyMMdd') AS INT) AS date_key,
  full_date,
  year(full_date)          AS year,
  quarter(full_date)      AS quarter,
  month(full_date)        AS month,
  monthname(full_date)    AS month_name,
  dayofweek(full_date)    AS day_of_week,   -- 1 = Sunday, 7 = Saturday
  day(full_date)          AS day_of_month,
  dayofyear(full_date)    AS day_of_year,
  dayname(full_date)      AS day_name,
  weekofyear(full_date)   AS week_of_year,
  date_format(full_date, 'yyyy-MM') AS year_month,
  CASE WHEN dayofweek(full_date) IN (1, 7) THEN false ELSE true END AS is_weekday,
  dayofweek(full_date) IN (1, 7) AS is_weekend
FROM dates;

-- 10. fact_sales — base grain from silver, enriched with surrogate keys
CREATE OR REPLACE TABLE workspace.gold.fact_sales
COMMENT 'Gold-layer sales fact table from silver.sales_details, enriched with surrogate keys from dim_customer and dim_product.'
AS
SELECT
    ROW_NUMBER() OVER (ORDER BY sd.order_number, sd.product_key) AS fact_sales_key,

    dc.customer_id       AS customer_id,    -- FK → dim_customer
    dp.product_id        AS product_id,    -- FK → dim_product (surrogate PK)
    sd.product_key       AS product_key,   -- original business key retained

    sd.order_number,
    sd.order_date,
    sd.ship_date,
    sd.due_date,

    sd.sales_amount,
    sd.quantity,
    sd.price

FROM workspace.silver.sales_details sd
LEFT JOIN workspace.gold.dim_customer dc
    ON sd.customer_id = dc.customer_id
LEFT JOIN workspace.gold.dim_product dp
    ON sd.product_key = dp.product_key;
```

---

## Job Orchestration & Scheduling

### Layer Orchestration Notebooks

| Notebook | What it runs |
|---|---|
| `Scripts/Silver/Silver Pipeline Orchestration.ipynb` | All 6 Silver notebooks in sequence (CRM first, then ERP) via `dbutils.notebook.run` |
| `Scripts/Gold/Gold Pipeline Orchestration.ipynb` | Gold notebooks in order — dimensions first, then the fact table |

### Databricks Job — `Jobs for Sales Data`

The whole pipeline is orchestrated as one multi-task Databricks Job:

| Task | Depends on | Notebook | Output |
|---|---|---|---|
| `bronze_Layer` | — | Bronze Layer | `workspace.bronze` tables |
| `Silver_Layer` | `bronze_Layer` | Silver Pipeline Orchestration | `workspace.silver` tables |
| `Gold_Layer` | `Silver_Layer` | Gold Pipeline Orchestration | `workspace.gold` star schema |

- One run executes the entire **Volume → Bronze → Silver → Gold** chain; each task starts only after the previous one succeeds (`ALL_SUCCESS`), so a failure stops everything downstream.
- `max_concurrent_runs = 1` with queueing enabled — runs never overlap.
- The job is currently triggered **manually**; attach a schedule (e.g. a daily cron) in the job's *Schedule* settings to run the full pipeline automatically.

---

## Data Flow Summary

```
UC Volume  /Volumes/workspace/bronze/source_system  (raw CRM + ERP CSVs)
    │
    ▼  Job task bronze_Layer   (Bronze Layer.ipynb)
Bronze Delta Tables (workspace.bronze)
    │
    ▼  Job task Silver_Layer   (Silver Pipeline Orchestration — 6 Spark SQL notebooks)
Silver Delta Tables (workspace.silver)
    │
    ▼  Job task Gold_Layer    (Gold Pipeline Orchestration + dim_date — Spark SQL)
Gold Star Schema (workspace.gold)
    ├── dim_customer  (SCD Type 2)
    ├── dim_product
    ├── dim_date
    └── fact_sales  ◄── joins to all three dimensions
    │
    ▼  Databricks SQL warehouse (connection details)
Power BI  ◄── ready to connect & visualize
```

---

## Connect to Power BI

The Gold star schema is **ready to connect to Power BI** for visualization:

1. **Get connection details** — open (or create) a Databricks **SQL warehouse** and copy its **Server Hostname** and **HTTP Path** from the *Connection Details* tab.
2. **Connect from Power BI Desktop** — `Get Data → Azure Databricks`, paste the Server Hostname and HTTP Path, and sign in (personal access token, OAuth, or Microsoft Entra ID).
3. **Load the Gold tables** — import or DirectQuery `workspace.gold.dim_customer`, `workspace.gold.dim_product`, `workspace.gold.dim_date`, and `workspace.gold.fact_sales`.
4. **Model the relationships** — `fact_sales.customer_id → dim_customer.customer_id`, `fact_sales.product_id → dim_product.product_id`, and `fact_sales.order_date → dim_date.full_date` (ship/due dates map the same way). Since `dim_customer` is SCD Type 2, filter it on `is_current = true` for one row per customer.
5. **Visualize** — build dashboards such as monthly sales trends (via `dim_date`), revenue by product category and product line, sales by country, and top customers by revenue.
