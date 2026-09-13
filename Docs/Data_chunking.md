## Data Preparation – Chunking, Format Conversion & Validation

### Purpose
Before implementing the Bronze layer ingestion, a small data preparation phase was performed to simulate real-world ingestion scenarios such as:

- multi-format source data (CSV, JSON, XML)
- historical batch loads vs incremental loads
- validation of transformations to ensure no data loss

This preparation step was intentionally added to demonstrate production-aligned ingestion patterns.

---

## 1) Format Conversion

### a) `menu_items` (CSV → JSON)
The `menu_items` dataset was converted from CSV to JSON in order to demonstrate ingestion using **Databricks Auto Loader** for semi-structured data.

- **Input:** `menu_items/` (CSV)
- **Output:** `menu_items_json/` (JSON)

---

### b) `stores` (CSV → XML)
The `stores` dataset was converted from CSV to XML in order to demonstrate ingestion using **Spark XML parsing**, which is commonly required for legacy enterprise datasets.

- **Input:** `stores/` (CSV)
- **Output:** `stores_xml/` (XML)

> Note: Databricks UI does not support previewing XML files stored inside Volumes, but Spark can read and process them correctly.

---

## 2) Transaction Chunking (Batch + Incremental Split)

### Purpose
To simulate a production ingestion pattern, the `transactions` dataset was split into:

- **Batch (historical)** data loaded using `COPY INTO`
- **Incremental (new)** data loaded using DLT streaming ingestion

### Logic
The split was performed using a cutoff date on `created_at`:

- records **<= cutoff date** → written to `chunked_transactions/batch/`
- records **> cutoff date** → written to `chunked_transactions/incremental/`

This enabled the project to demonstrate both:
- one-time historical backfill ingestion
- continuous incremental ingestion

---

## 3) Unit Testing – Format Conversion Validation

### Objective
A unit-test notebook was created to validate the correctness of format conversions by checking:

- record count reconciliation (CSV vs JSON / CSV vs XML)
- column presence validation (missing / extra columns)

### Outcome
The tests confirmed that conversions did not introduce unintended data loss.  
In the case of `menu_items`, two columns (`available_from`, `available_to`) were missing in the JSON output because they contained only null values in the source and were dropped during conversion. This was validated and treated as expected behavior.

---

### Result
This preparation phase ensured that all converted and chunked datasets were correct and ready for downstream Bronze ingestion, while also showcasing multi-format and incremental ingestion patterns.
