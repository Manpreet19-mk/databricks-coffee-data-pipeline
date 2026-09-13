# VStone Coffee Analytics – Requirement & Assumption Document (Batch Lakehouse Pipeline)

## 1. Project Overview

This project implements an end-to-end Databricks Lakehouse pipeline using **Medallion Architecture (Bronze → Silver → Gold)** for a coffee shop sales dataset. The pipeline ingests static batch datasets, applies cleaning and standardization, ensures rerun-safe incremental processing, and prepares analytics-ready dimensional and aggregate data for reporting and downstream consumption.

This is a **batch (static) pipeline**, not a streaming solution. However, incremental processing is implemented using Databricks capabilities such as Auto Loader, Delta MERGE, and Delta Live Tables (DLT) where applicable.

---

## 2. Business Objective

Build a reliable analytics foundation that supports:

- Clean and standardized tables for reporting
- Traceability and audit using metadata and operational logs
- Handling of invalid records using a quarantine strategy
- Dimensional model creation in the Gold layer (facts + dimensions)
- Production-aligned operational practices such as:
  - monitoring
  - logging
  - resource usage analysis
  - CI/CD automation

---

## 3. Data Sources / Tables

The following datasets are in scope:

- transactions
- transaction_items
- stores
- users
- vouchers
- payment_methods
- menu_items

---

## 4. Requirements

### 4.1 Bronze Layer Requirements

- Ingest raw datasets into Delta tables with minimal transformation.
- Preserve raw values and schema.
- Support reruns without creating duplicate data.
- Add audit metadata columns:

  - `loaded_at` (load timestamp)
  - `updated_at` (source / processing timestamp)
  - `load_dt` (load date)
  - `source` (dataset identifier / folder name)
  - `source_file` (raw file name)

---

### 4.2 Silver Layer Requirements

- Standardize column names using a reusable column-standardization UDF.
- Enforce correct datatypes (INT, DOUBLE, DATE, TIMESTAMP).
- Apply data quality checks:
  - Required columns must not be NULL.
  - Invalid records must be stored in quarantine tables.
- Ensure rerun safety using incremental loading:
  - Silver reads only new Bronze rows using watermark logic.
- Maintain auditability:
  - Preserve Bronze metadata columns in Silver.
  - Add Silver audit columns:
    - `silver_loaded_at`
    - `silver_updated_at`

---

### 4.3 Transaction Items Special Requirement (Validated)

The dataset contains repeated identical rows in `transaction_items`.  
To prevent revenue loss and maintain correctness:

- Exact duplicate line-items are **rolled up** by:
  - summing `quantity`
  - summing `subtotal`
- A deterministic surrogate key is generated:
  - `transaction_item_sk = sha2(transaction_id|item_id|unit_price|created_at)`
- This provides a stable primary-key-like identifier for analytics and joins.

---

### 4.4 Gold Layer Requirements

Gold is designed as the analytics-ready layer with a dimensional model and KPI aggregates.

#### 4.4.1 Gold Staging Tables
- Create staging tables (`stg_*`) for each major entity.
- Enrich data where needed (example: transaction_items enriched with store_id from transactions).
- Add lineage and audit metadata such as:
  - `source_table`
  - `gold_loaded_at`
  - `gold_updated_at`

#### 4.4.2 Gold Dimensional Model
- Create a star schema model:

  **Fact tables**
  - fact_transactions
  - fact_transaction_items

  **Dimension tables**
  - dim_users
  - dim_stores
  - dim_menu_items
  - dim_payment_methods
  - dim_vouchers

- Implement incremental updates for facts and dimensions using DLT `APPLY CHANGES INTO` (SCD Type 1 for this implementation).

#### 4.4.3 Aggregations / KPI Tables
Gold includes analytics-ready aggregates such as:
- top selling items
- store performance metrics
- revenue trends
- discount impact

These tables are designed for dashboards and BI reporting.

---

## 5. Databricks / Delta Features Used

This project uses the following Databricks features:

- Unity Catalog (catalog + schemas + managed tables)
- Delta Lake tables (ACID compliant)
- Auto Loader (incremental file ingestion)
- COPY INTO (batch file ingestion)
- MERGE INTO for incremental upserts
- Delta Live Tables (DLT) pipelines for Gold modeling and incremental ingestion
- Databricks Workflows / Jobs for orchestration
- Widgets (`dbutils.widgets`) to reduce hardcoding and improve reusability
- Delta Time Travel + History:
  - `DESCRIBE HISTORY`
  - `VERSION AS OF`
  - `RESTORE TABLE`

---

## 6. Orchestration Requirements (Jobs + Pipelines)

The pipeline is orchestrated using Databricks Jobs and DLT pipelines.

### 6.1 Databricks Jobs
Jobs represent medallion stages:

1. `Data_Chunking`
2. `bronze_ingestion`
3. `silver_ingestion`
4. `gold_ingestion`

### 6.2 Delta Live Tables Pipelines
DLT pipelines used:

- `transactions_incremental_dev` (Bronze incremental ingestion)
- `gold_dev` (Gold staging + dimensions + facts + aggregations)

---

## 7. Monitoring Requirements

Monitoring is implemented using built-in Databricks observability features:

### 7.1 Jobs Monitoring
Monitoring includes:
- run status
- duration and execution time
- task DAG
- task logs
- failure email notifications

### 7.2 DLT Monitoring
Monitoring includes:
- pipeline graph and lineage
- update history
- table update status
- event logs

Monitoring evidence is captured from the Databricks UI.

---

## 8. Logging & Audit Requirements

In addition to monitoring, the project includes operational logging to support traceability and audit.

### 8.1 Pipeline Logging Tables
Logging tables are created to store operational metadata such as:
- job name
- notebook name
- layer (bronze/silver/gold)
- start time, end time, duration
- status (success/failure)
- row counts (optional)
- error message (if failed)
- source metadata (folder + file)

These audit tables provide production-grade observability and can be used for troubleshooting and reporting.

---

## 9. Resource Usage Analysis Requirements

Resource usage analysis is performed to understand runtime and compute behaviour.

Analysis includes:
- job duration trends
- query execution time from query history
- DLT pipeline update duration
- compute type used (serverless)

This provides visibility into performance and operational cost drivers.

---

## 10. Testing Requirements

Testing is implemented at two levels:

### 10.1 Reconciliation / Data Validation Tests
Validation includes:
- Bronze vs Silver reconciliation:
  - record counts
  - subtotal / amount / quantity totals
- Uniqueness tests for primary keys (where applicable)
- EXCEPT-based difference checks
- Rollup correctness validation:
  - subtotal totals match before vs after rollup
  - quantity totals match before vs after rollup
- Delta Time Travel demo using update + restore for ACID verification

### 10.2 Unit Tests (Reusable Logic)
A separate unit testing folder was implemented for reusable logic.

Unit tests validate:
- SQL UDF column name standardization
- DataFrame column standardization function
- Deduplication logic (latest record retention)

Unit tests are executed through a dedicated Databricks job (`unit_test_runner`) and integrated into CI/CD.

---

## 11. Deployment & CI/CD Requirements

CI/CD automation is implemented using:

- Databricks Asset Bundles (DAB)
- GitHub Actions

The CI/CD pipeline ensures:
- bundle validation on PR and branch changes
- automated deployment to DEV on dev branch pushes
- automated deployment to PROD on main branch pushes
- automated unit test execution after deployment

This provides a production-aligned deployment process for Databricks jobs and pipelines.

---

## 12. Assumptions

### 12.1 Processing Assumptions
- The pipeline is batch-based and processes static datasets.
- Incremental processing is driven by load timestamps and ingestion metadata.

### 12.2 Primary Key Assumptions (Validated)
- `transactions`: `transaction_id` is treated as primary key.
- `users`: `user_id` is treated as primary key.
- `stores`: `store_id` is treated as primary key.
- `payment_methods`: `method_id` is treated as primary key.
- `menu_items`: `item_id` is treated as primary key.
- `vouchers`: `voucher_id` is treated as primary key; `voucher_code` may repeat.

### 12.3 Transaction Items Assumption (Key Design Decision)
- `transaction_items` does not contain a reliable natural primary key.
- Duplicate line-items exist and are valid.
- Therefore rollup + surrogate key generation is required for correctness.

### 12.4 Null Handling Assumptions
- Some fields may be NULL and are acceptable:
  - `voucher_id` and `user_id` in transactions
- Mandatory fields must not be NULL; invalid records are quarantined.

### 12.5 Serverless Compute Assumption
- Serverless compute is used for job execution and pipeline runs.
- Resource usage evidence is collected from job run details and query history.

---

## 13. Out of Scope

The following are explicitly out of scope for this submission:

- Streaming ingestion (Kafka / real-time ingestion)
- ML models (recommendation, forecasting, churn)
- Advanced cost optimization (cluster pools, spot instances, autoscaling tuning)

---

## 14. Future Enhancements / Scope

The following are future improvements planned beyond this submission:

### 14.1 Extended Audit Reporting
- Build reporting on audit/log tables for:
  - pipeline SLA
  - failure trends
  - row count drift
  - incremental load validation

### 14.2 Production Release Workflow
- Add approval-based deployment workflow (dev → staging → prod).
- Add automated rollback strategies using Delta restore and job versioning.
