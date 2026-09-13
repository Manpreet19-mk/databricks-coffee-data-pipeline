# vstone-databricks-pipeline

##  VStone Coffee Analytics – Databricks Lakehouse Pipeline

## Overview
This project implements an end-to-end **Databricks Lakehouse pipeline** using **Medallion Architecture**:

- **Bronze** → raw ingestion into Delta tables  
- **Silver** → cleaned, standardized, validated, rerun-safe incremental tables  
- **Gold** → dimensional model (facts + dimensions) with SCD support and data quality expectations  

The dataset contains coffee shop business data including:  
`transactions`, `transaction_items`, `users`, `stores`, `vouchers`, `menu_items`, and `payment_methods`.

---

## Key Design Thought Process
The pipeline was designed with a focus on production-aligned Lakehouse practices:

- Preserve raw data in Bronze for traceability and replay
- Standardize schema and enforce types in Silver to support reliable downstream modeling
- Implement rerun-safe incremental logic to prevent duplicates
- Handle known data issues (duplicates, missing keys, invalid rows) with controlled transformations
- Build a reporting-ready Gold layer using a dimensional model

---

## Project Flow

### 1) Data Profiling (Discovery Phase)
Before building transformations, all datasets were profiled to understand:

- null patterns
- uniqueness of business keys
- duplicate patterns (especially in `transaction_items`)
- datatype issues and schema inconsistencies

This profiling guided key decisions in Silver and Gold design.

---

### 2) Multi-Format Data Ingestion (Chunking + Conversion)
To demonstrate handling of different ingestion patterns:

- `transactions` was split into multiple files (chunking)
- `stores` was converted into **XML**
- `menu_items` was converted into **JSON**
- remaining datasets were kept as **CSV**

Ingestion methods used:

- **COPY INTO** (CSV)
- **Auto Loader** (JSON)
- **PySpark parsing** (XML)

---

### 3) Bronze Layer (Raw → Delta)
Bronze tables store raw ingested data with minimal transformation and include audit metadata:

- `loaded_at`
- `updated_at`
- `load_dt`
- `source`
- `source_file`

---

### 4) Silver Layer (Cleaning + Standardization + Incremental)
Silver tables include:

- standardized column names using a reusable UDF
- datatype casting into correct formats
- incremental ingestion using `loaded_at` watermark logic
- rerun-safe processing (MERGE / incremental filtering)
- quarantine handling for invalid records (where applicable)
- silver audit columns (`silver_loaded_at`, `silver_updated_at`)

---

### 5) Special Handling – transaction_items Rollup + Surrogate Key
`transaction_items` contains repeated identical rows and does not contain a reliable natural primary key.

To preserve correct revenue and quantity:

- duplicates were rolled up (aggregated)
- `quantity` and `subtotal` were summed
- deterministic surrogate key generated using SHA2 hashing

---

### 6) Gold Layer (Dimensional Model + Expectations)
Gold tables are built using a dimensional model approach:

**Fact tables**
- `fact_transactions`
- `fact_transaction_items`

**Dimension tables**
- `dim_users`
- `dim_stores`
- `dim_menu_items`
- `dim_payment_methods`
- `dim_vouchers`

Gold also includes:
- SCD support where applicable
- expectations/constraints for key business validations
- audit + lineage columns for traceability

---

## Testing & Validation
Testing was performed after each layer using SQL-based validation:

- record count reconciliation across layers
- EXCEPT checks (difference validation)
- aggregate reconciliation (amounts, subtotal, quantities)
- uniqueness checks for keys
- validation of transaction_items rollup logic

Additionally, reusable **unit-test notebooks** were created to validate core transformation logic.

---

## CI/CD (Databricks Asset Bundles + GitHub Actions)
The project uses **Databricks Asset Bundles (DABs)** and **GitHub Actions** to automate:

- bundle validation
- deployment to DEV and PROD targets
- execution of unit test job after deployment

---

## Repository Structure
- `src/` → notebooks and pipeline logic  
- `resources/` → Databricks Jobs + Pipeline YAML definitions  
- `tests/` → reconciliation + unit test notebooks  
- `docs/` → project documentation (requirements, CI/CD)  
- `.github/workflows/` → GitHub Actions CI/CD pipeline  
- `databricks.yml` → DAB configuration  

---

## Future Enhancements 
- Monitoring dashboards for pipeline health and job performance  
- Cost optimization reporting using system billing tables  
