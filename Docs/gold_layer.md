# Gold Layer – Dimensional Model (DLT )

## Overview
The **Gold layer** represents the final analytics-ready model of the Lakehouse.  
It is implemented using **Delta Live Tables (DLT)to provide:

- managed incremental processing
- declarative dependencies across tables
- built-in data quality expectations
- SCD support for dimensional history
- end-user reporting readiness (facts + dimensions + KPI views)

Gold is built from the Silver layer and follows a **Star Schema** design.

---

## Gold Layer Flow
The Gold pipeline is organized into three logical steps:

1. **Gold Staging Tables (`stg_*`)**
2. **Gold Dimensions (`dim_*`)**
3. **Gold Facts (`fact_*`)**
4. **Gold Aggregations (materialized KPI views)**

---

## 1) Gold Staging Tables (`stg_*`)
Gold staging tables act as a clean and controlled input for the dimensional model.

### Purpose
- Select only business columns required for Gold
- Drop Bronze/Silver technical metadata not needed in reporting
- Add Gold audit timestamps
- Add lineage column (`source_table`) for traceability
- Apply DLT expectations for key validations
- Retain `silver_updated_at` to support SCD sequencing

### Staging Tables Created
- `coffee.gold.stg_users`
- `coffee.gold.stg_stores`
- `coffee.gold.stg_menu_items`
- `coffee.gold.stg_vouchers`
- `coffee.gold.stg_payment_methods`
- `coffee.gold.stg_transactions`
- `coffee.gold.stg_transaction_items`

### Audit + Lineage Columns Added in Staging
All staging tables include:
- `source_table` (example: `coffee.silver.users`)
- `gold_loaded_at`
- `gold_updated_at`

### Special Handling – `stg_transaction_items`
`stg_transaction_items` includes additional enrichment:
- joins `transaction_items` with `transactions` to add `store_id`

This was required to support downstream governance scenarios (e.g., row-level security), since nested row filters across fact tables are not supported.

It also adds:
- `transaction_item_key` (BIGINT) derived from `transaction_item_sk`  
  (useful for BI tools)

---

## 2) Gold Dimensions (`dim_*`) – SCD Type 2
All Gold dimensions are implemented using **DLT SCD Type 2** logic.

### Why SCD Type 2?
Dimensions contain descriptive attributes that may change over time (example: store metadata, menu categories, user attributes).  
SCD2 preserves full history and enables time-aware analytics.

### Implementation
Each dimension is created using:

- `APPLY CHANGES INTO`
- `KEYS (...)`
- `SEQUENCE BY silver_updated_at`
- `STORED AS SCD TYPE 2`

DLT automatically manages historical versions and adds SCD2 metadata.

### Dimensions Created
- `coffee.gold.dim_users` (key: `user_id`)
- `coffee.gold.dim_stores` (key: `store_id`)
- `coffee.gold.dim_menu_items` (key: `item_id`)
- `coffee.gold.dim_vouchers` (key: `voucher_id`)
- `coffee.gold.dim_payment_methods` (key: `method_id`)

### Automatically Managed SCD2 Columns
DLT adds the following metadata columns:
- `__START_AT`  → effective start timestamp  
- `__END_AT`    → effective end timestamp  

---

## 3) Gold Fact Tables (`fact_*`) – SCD Type 1
Fact tables represent business events and numeric measures.

### Why SCD Type 1 for facts?
Facts typically represent the final truth for a transaction/event.  
If updates arrive (late corrections), the fact should be overwritten rather than versioned.

### Fact Tables Created
- `coffee.gold.fact_transactions`
  - Grain: 1 row per transaction
  - Key: `transaction_id`

- `coffee.gold.fact_transaction_items`
  - Grain: 1 row per rolled-up transaction line item
  - Key: `transaction_item_sk` (deterministic SHA key from Silver)
  - Includes: `transaction_item_key` (BIGINT) for BI usability

### Implementation
Both facts use DLT-managed SCD Type 1:

- `APPLY CHANGES INTO`
- `KEYS (...)`
- `SEQUENCE BY silver_updated_at`
- `STORED AS SCD TYPE 1`

This guarantees:
- idempotent processing
- duplicate prevention
- only the latest version of each fact row

---

## 4) Gold Aggregations (Materialized Views)
To provide reporting-ready outputs, the Gold layer includes **business KPI materialized views** built on top of the Gold fact and dimension tables.

###  Materialized Views?
- fast dashboard performance
- reusable KPI logic
- simplified BI consumption

### Aggregation Outputs
- `agg_monthly_sales`
  - monthly sales, transactions, discount, AOV

- `agg_top_3_menu_items`
  - top 3 items by revenue + quantity sold  
  - joins current `dim_menu_items` (`__END_AT IS NULL`)

- `agg_voucher_usage`
  - voucher vs no-voucher performance summary

- `agg_payment_method_split`
  - transactions + revenue by payment method  
  - joins current `dim_payment_methods`

- `agg_customer_clv`
  - customer lifetime spend + orders + first/last purchase

- `agg_store_performance`
  - store-level KPIs by city/state  
  - joins current `dim_stores`

- `agg_top_10_customers_by_spend`
  - identifies highest-value customers

### Churn Metrics
- `agg_customer_churn_6m`
  - churned if no transaction in last 6 months

- `agg_new_customer_churn_6m`
  - one-time buyers whose only transaction occurred > 6 months ago

---

## Star Schema (Gold Model)
Gold follows a classic star schema:

### Fact Tables
- `fact_transactions`
- `fact_transaction_items`

### Dimension Tables
- `dim_users`
- `dim_stores`
- `dim_menu_items`
- `dim_payment_methods`
- `dim_vouchers`

`fact_transactions` acts as the transaction header table, and `fact_transaction_items` stores the line-item grain.

---

## Outcome
The Gold layer provides:

- governed, incremental, and expectation-driven tables via DLT
- SCD2 dimensions for historical analysis
- SCD1 facts for correct and duplicate-free event modeling
- KPI-ready materialized views for dashboards
- full audit and lineage metadata for traceability
