# Bronze Layer (Raw Ingestion) — Documentation

## Overview
The Bronze layer represents the **raw landing zone** of the Lakehouse.  
Its purpose is to ingest source files into Delta tables with **minimal transformation**, while ensuring:

- raw data is preserved for replay and traceability  
- ingestion is scalable across multiple datasets  
- reruns do not create duplicates (idempotent ingestion)  
- consistent audit + lineage metadata is captured for governance  

The Bronze layer ingests multiple file formats to demonstrate real-world ingestion patterns:
- CSV via **COPY INTO**
- JSON via **Auto Loader**
- XML via **Spark XML parsing**
- incremental transactions via **DLT streaming**

---

## Bronze Layer Standards

### Raw Columns
- All business columns are ingested as **STRING** in Bronze.
- No casting or cleansing is performed in Bronze.
- Datatype enforcement is handled in the Silver layer.

### Audit & Lineage Columns
All Bronze tables include the following metadata columns:

| Column Name   | Description |
|-------------|-------------|
| `loaded_at`   | Timestamp when the row was ingested into Bronze |
| `updated_at`  | Timestamp for last update in Bronze (initially same as loaded_at) |
| `load_dt`     | Ingestion date (useful for filtering and reconciliation) |
| `source_file` | File-level lineage (exact file path/name from `_metadata`) |
| `source`      | *(Optional)* folder-level lineage tag (dataset-level source identifier) |

---

## Metadata-Driven COPY INTO Framework (Generic Notebook)

### Why a Generic Notebook?
Most datasets in this project are ingested from CSV.  
To avoid repeating ingestion logic for each dataset, a **single generic COPY INTO notebook** was created and reused for all CSV-based Bronze tables.

This design ensures:
- minimal code duplication  
- faster onboarding of new datasets  
- consistent audit and lineage standards across all tables  
- parameter-driven execution inside Databricks Jobs / DABs  

---

### How It Works (Job + Task Parameters)

The generic ingestion notebook accepts parameters using Databricks widgets:

#### Job Parameters (Reusable Across All Tasks)
- `catalog` → Unity Catalog catalog name (e.g., `coffee`)
- `bronze_schema` → schema name (e.g., `bronze`)
- `raw_volume` → base UC volume path where raw files are stored

#### Task Parameter (Varies Per Table)
- `table_name` → the dataset/table to ingest (e.g., `users`, `vouchers`, `payment_methods`)

Each COPY INTO task in the job passes a different `table_name`, while reusing the same notebook logic.

---

### Configuration-Driven Ingestion (`bronze_config.yml`)
A YAML configuration file is used to define ingestion metadata per table.  
Each table entry includes:

- `source_subfolder` → raw folder name inside the UC volume  
- `columns` → raw schema (column list)  
- `file_format` → input file format (CSV default)  
- `format_options` → parsing options (e.g., header = true)  
- `copy_options` → COPY INTO options (e.g., mergeSchema = false)  
- `add_source_column` → whether to include folder-level lineage (`source`)  
- `source_value` → value stored in `source` when enabled  

This enables ingestion to be fully driven by metadata rather than hardcoded logic.

---

### Global Utilities (`common/Globals`)
A shared utilities notebook was created to centralize reusable logic across Bronze ingestion notebooks.

It contains:
- YAML config loader (`load_config`)
- fully qualified table name builder (`build_table_fqn`)
- UC volume source path builder (`build_source_path`)
- Bronze `CREATE TABLE` DDL generator (`build_create_table_sql`)
- standardized audit column DDL snippet (reused across all tables)

This ensures consistency and reduces maintenance overhead.

---

## JSON Ingestion — `menu_items` (Auto Loader)

### Why Auto Loader?
The `menu_items` dataset was converted to JSON to demonstrate semi-structured ingestion.  
JSON ingestion was implemented using **Databricks Auto Loader** to support:

- incremental file discovery  
- schema tracking via schemaLocation  
- checkpoint-based idempotent ingestion  

### Design Notes
- A fixed schema is provided to prevent type inference issues.
- All columns are kept as STRING in Bronze.
- Audit and lineage columns are appended before writing to Delta.

---

## XML Ingestion — `stores` (Spark XML Reader)

### Why Spark XML Parsing?
The `stores` dataset was converted to XML to demonstrate ingestion of non-tabular formats.

Spark’s XML reader was used with:
- explicit schema definition  
- `rowTag = "store"` to map each `<store>` element to one row  

Audit and lineage columns are added before writing to Bronze Delta.

---

## Transactions Ingestion — Batch + Incremental Pattern

### Motivation
The `transactions` dataset was intentionally split to demonstrate real-world ingestion patterns:

- **Historical batch data** (16 months) → ingested using `COPY INTO`
- **New / incremental transactions** → ingested using a **DLT streaming table**

This simulates a common production setup where historical backfills are batch loaded, while new files are streamed incrementally.

---

### DLT Streaming Table — `transactions_incremental`
A DLT streaming table ingests new transaction CSV files using:

- `STREAM read_files(...)`
- DLT-managed file tracking + checkpointing
- audit + lineage enrichment during ingestion

This ensures rerun safety and incremental processing.

---

## Final Bronze Consolidation — Merge into `coffee.bronze.transactions`

After both ingestion paths complete, a final Bronze table is created and populated:

1. **Create target table** (schema aligned with batch source)
2. **MERGE batch transactions** into the final Bronze table
3. **MERGE incremental transactions** into the same table

### Why MERGE?
This ensures:
- no duplicate `transaction_id` values  
- incremental updates overwrite existing transaction rows when required  
- a single unified Bronze table is produced for downstream Silver processing  

---

## Job Orchestration (Dependency-Based Execution)
The Bronze job is designed with task dependencies:

- batch ingestion tasks run first  
- incremental DLT ingestion runs independently  
- final merge task runs only after both batch + incremental ingestion tasks complete  

This guarantees correct ordering and produces a consistent final Bronze dataset.

---

## Outcome
The Bronze layer provides a reliable raw foundation for the Silver layer by ensuring:

- raw data preservation  
- scalable ingestion patterns (COPY INTO, Auto Loader, Spark XML, DLT streaming)  
- consistent audit + lineage metadata  
- rerun-safe and production-aligned ingestion design  
