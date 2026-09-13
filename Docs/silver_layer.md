# Silver Layer (Clean + Standardized + Incremental) — Documentation

## Overview
The Silver layer transforms raw Bronze data into a **clean, standardized, and analytics-ready** format.  
This layer is responsible for enforcing schema consistency, applying data quality rules, and ensuring rerun-safe incremental processing.

The Silver design follows production-aligned Lakehouse principles:

- Bronze is preserved as raw and replayable
- Silver enforces consistency and correctness
- Silver outputs are stable inputs for Gold dimensional modeling

---

## Key Silver Design Principles

### 1) Centralized Column Standardization (UDF-Driven)
All Silver tables standardize column names using a centrally defined SQL UDF:

- **UDF Name:** `coffee.silver.standardize_column_name`

The UDF enforces consistent `snake_case` naming by:
- converting to lowercase
- replacing spaces/hyphens with underscores
- removing special characters
- collapsing repeated underscores
- trimming leading/trailing underscores

A reusable helper function (`standardize_columns(df)`) applies this UDF across all columns for every Silver table.

**Benefit:** prevents schema inconsistencies and simplifies downstream Gold modeling.

---

### 2) Metadata-Driven and Reusable Notebook Pattern
All Silver ingestion notebooks follow a consistent structure and are executed through a Databricks Job.

The job uses:

#### Job Parameters (common for all tasks)
- `catalog`
- `bronze_schema`
- `silver_schema`
- `default_watermark`

#### Task Parameter (varies per dataset)
- `source_table` / `table_name`

This allows the same Silver notebook logic to be reused across datasets with minimal changes, improving maintainability and scalability.

---

### 3) Task Dependencies (UDF as a Required Upstream Step)
All Silver tasks are configured to depend on the UDF task.

This ensures:
- the column standardization UDF is created/available before any Silver transformations execute
- consistent naming rules are applied across all Silver tables
- downstream logic does not break due to missing UDF dependencies

---

### 4) Incremental Processing Using Watermark Logic
Silver tables are processed incrementally using the Bronze ingestion timestamp (`loaded_at`) as a watermark.

Each Silver notebook filters Bronze data using:

- `loaded_at > MAX(loaded_at)` from the target Silver table  
- fallback to a `default_watermark` for first-run scenarios

This ensures:
- only new Bronze records are processed
- historical data is not reprocessed unnecessarily
- reruns remain safe and consistent

---

### 5) Data Quality Validation + Quarantine Handling
Silver applies business validation rules such as:

- required key columns must not be null  
- mandatory descriptive fields must be present  
- numeric fields must be valid  

Rows failing validation are redirected into a dedicated quarantine table:

- `<table>_quarantine`

Quarantine tables store:
- raw record values
- `quarantine_reason`
- `quarantined_at`

**Benefit:** prevents bad data from entering Silver while still preserving it for debugging and audit.

---

### 6) Deduplication + Latest Record Selection
Where duplicates exist in Bronze, Silver applies deduplication logic using window functions such as:

- `ROW_NUMBER() OVER (PARTITION BY <business_key> ORDER BY updated_at DESC)`

This ensures:
- one clean record per business key
- latest version of the record is retained

---

### 7) Rerun-Safe Upserts Using MERGE
All Silver tables are loaded using idempotent `MERGE` logic:

- `WHEN MATCHED` → update existing records  
- `WHEN NOT MATCHED` → insert new records  

This ensures Silver tables:
- support incremental updates
- avoid duplicate rows on reruns
- remain stable inputs for Gold fact/dimension modeling

---

## Special Silver Handling – `transaction_items`
The `transaction_items` dataset required special handling due to:

- repeated identical rows
- lack of a reliable natural primary key

Silver applies:
- duplicate rollup (`SUM(quantity)`, `SUM(subtotal)`)
- deterministic surrogate key generation using SHA2 hashing
- row_count metadata to track rollup behavior

This ensures correct revenue calculations and stable downstream joins.

---

## Outcome
The Silver layer produces curated Delta tables that are:

- standardized (naming + schema)
- typed and validated
- deduplicated and incrementally processed
- rerun-safe using MERGE
- traceable using Bronze + Silver audit metadata

These Silver outputs serve as the foundation for Gold dimensional modeling and reporting.

