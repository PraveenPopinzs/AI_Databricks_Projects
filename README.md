# AI_Databricks_Projects

A **Medallion Architecture** data engineering project on Databricks that ingests raw CRM and ERP CSV data, transforms it through Bronze → Silver → Gold layers, and delivers analytics-ready dimension and fact tables in Unity Catalog.

> **Note:** The full project pipeline lives on the `feature/medallion-architecture` branch. The `Main` branch contains the README and the Gold Dim Date notebook.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Repository Structure](#repository-structure)
3. [Data Sources](#data-sources)
4. [Bronze Layer — Ingestion](#bronze-layer--ingestion)
5. [Silver Layer — Transformation](#silver-layer--transformation)
   - 5.1 [CRM Transformations](#crm-transformations)
   - 5.2 [ERP Transformations](#erp-transformations)
   - 5.3 [Silver Pipeline Orchestration](#silver-pipeline-orchestration)
6. [Gold Layer — Dimensional Modelling](#gold-layer--dimensional-modelling)
   - 6.1 [dim_customer](#dim_customer)
   - 6.2 [dim_product](#dim_product)
   - 6.3 [dim_date](#dim_date)
   - 6.4 [fact_sales](#fact_sales)
   - 6.5 [Gold Pipeline Orchestration](#gold-pipeline-orchestration)
7. [Data Flow Summary](#data-flow-summary)

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

## Data Sources

Six CSV files from two source systems are ingested into the Bronze layer via UC Volumes.

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

## Silver Layer — Transformation

Cleans, standardises, and enriches Bronze data into readable Silver tables in `workspace.silver`.

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

## Gold Layer — Dimensional Modelling

Joins and enriches Silver tables into star-schema dimensions and a fact table in `workspace.gold`.

### dim_customer

**Notebook:** `Scripts/gold/Gold Dim Customer.ipynb`

Three-way LEFT JOIN of CRM `customer_info` + ERP `customer_info1` + ERP `customer_info2` into a unified customer dimension.

- Combines CRM demographics (name, marital status, gender) with ERP data (birth date, country)
- All customer IDs normalised to `AW000xxxxx` format for direct equality joins

### dim_product

**Notebook:** `Scripts/gold/Gold Dim Product.ipynb`

LEFT JOIN of CRM `product_info` with ERP `product_info2` into a unified product dimension.

- Strips category/subcategory prefix from `product_key` using `REGEXP_EXTRACT` for clean join keys
- Enriches products with category and subcategory information

### dim_date

**Notebook:** `Scripts/gold/Gold Dim Date.ipynb` *(Main branch)*

Creates `workspace.gold.dim_date` with auto-generated integer `date_key` (YYYYMMDD format) covering 2010-01-01 to 2014-12-31.

- Columns: `date_key`, `full_date`, `year`, `quarter`, `month`, `day`, `day_of_week`, `day_name`, `month_name`, `is_weekend`, `quarter_label`
- Adds `order_date_key`, `ship_date_key`, `due_date_key` FK columns to `fact_sales`

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

## Data Flow Summary

```
CSV Files (UC Volumes)
    │
    ▼  Bronze Layer.ipynb
Bronze Delta Tables (workspace.bronze)
    │
    ▼  Silver notebooks (6 notebooks + orchestration)
Silver Delta Tables (workspace.silver)
    │
    ▼  Gold notebooks (dim_customer, dim_product, dim_date, fact_sales)
Gold Star Schema (workspace.gold)
    ├── dim_customer
    ├── dim_product
    ├── dim_date
    └── fact_sales  ◄── joins to all three dimensions
```
