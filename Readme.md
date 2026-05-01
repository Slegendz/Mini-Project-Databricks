# Data Engineering Pipeline Documentation

## Medallion Architecture (Bronze → Silver → Gold)
**Technology Stack:** PySpark, Delta Lake, Azure Data Lake, Databricks

---

# 1. Project Overview

This project implements a **scalable data pipeline** using the Medallion Architecture to transform raw data into **business-ready insights**.

The pipeline is designed to ensure:

* Data reliability
* Scalability
* Auditability
* Clear separation of concerns

---

# 2. Architecture Design

The data pipeline follows a **Medallion Architecture**:

* **Bronze Layer** → Raw ingestion from source
* **Silver Layer** → Data cleaning & standardization
* **Gold Layer** → Business-ready data (DIM, FACT, KPIs)

```
Source (Azure Blob Storage)
        ↓
Bronze Layer (Raw Data)
        ↓
Silver Layer (Cleaned & Standardized Data)
        ↓
Gold Layer (Dimensional Model + KPIs + Business Logic)
```

---

# 🥉 3. Bronze Layer (Raw Ingestion)

## File

`bronze/ingest_bronze.ipynb`

## Objective

To ingest raw data from source systems **without any transformation**.

* Dynamically reads all tables from source catalog:

```python
tables = [t.name for t in spark.catalog.listTables("azure_blob_storage")]
```

* Adds ingestion timestamp:

```python
df = df.withColumn("ingestion_ts", current_timestamp())
```

* Stores as Delta tables:

```
workspace.bronze.bronze_<table_name>
```

## Assumptions

* Source data schema is consistent
* Data may contain errors, nulls, or duplicates
* No transformations are applied
* All tables are ingested **as-is without transformation**
* No filtering, no validation at this stage

## Data Quality Rules

* No validation performed
* Raw data preserved for traceability
* Schema drift allowed
* Ingestion timestamp ensures auditability

---

# 🥈 4. Silver Layer (Data Cleaning & Standardization)

## Objective

To clean, validate, and standardize data before business use.


## Transformations Applied

### 1. Date Standardization

* Convert string dates to proper format:

```python
to_date(col("order_date"), "dd-MM-yyyy")
```

### 2. Null Handling

* Remove or filter records with null values in:

  * `order_id`
  * `customer_id`
  * `product_id`
  * `order_status`
  * `customer_email`
  * `customer_name`

### 3. Deduplication

* Remove duplicate records using primary keys

### 4. Data Validity

* Is_valid column for checking valid records
* Is_invalid_status to check if status is valid and present

## Assumptions

* Data inconsistencies exist in raw layer
* All transformations must be handled in Silver layer
* Silver acts as a trusted data source
* Data may contain null vaues, incorrect formats, duplicates, orphan records

## Data Quality Rules

* No nulls in primary keys
* Dates must be correctly parsed
* No duplicate records
* Valid currency mappings
* Invalid records are filtered

---

# 🥇 5. Gold Layer (Business Layer)

## Objective

To provide **analytics-ready datasets** using dimensional modeling.


## Data Model

### Dimension Tables

* `dim_customer`
* `dim_product`
* `dim_date`
* `dim_country`

###  Fact Table

* `fact_orders`

---- 

## Key Business Transformations

### 1. Currency Standardization (USD)

All revenue is converted to USD:

```
revenue_usd = quantity × price × rate_to_usd
```

* Ensures global consistency
* Applied at fact table level

### 2. Status Handling

All order statuses are preserved:

* `Pending`
* `Completed`
* `Shipped`
* `Cancelled`

* No filtering in fact layer
* Enables flexible KPI calculations

### 3. Country Attribution

Revenue is attributed using:

```
orders.country
```

* Represents actual transaction location
* Used for regional performance analysis

## Assumptions

* Silver data is clean and reliable
* Exchange rates are accurate
* Fact table contains all transactional data
* Business logic is applied only here

## Data Quality Rules

* Revenue must be greater than 0
* Foreign keys must be valid
* No nulls in critical fields
* Valid joins across tables
* Exchange rates must exist

---

# 6. KPI Layer

## Objective

To derive business insights from the Gold layer.

## KPIs Implemented

### 1. Revenue Metrics

* Total Revenue (USD)
* Revenue by Country
* Revenue by Channel

✔ Only **completed orders** are considered


### 2. Customer Metrics

* Customer Acquisition (monthly)
* Active Customers

Based on first purchase date

### 3. Order Metrics

* Completed Order Rate
* Average Order Value (AOV)

### 4. Product Metrics

* Top Performing Products

### 5. Data Quality Score

Defined as:

```
Data Quality Score = Valid Records / Total Records
```

Valid records must:

* Have no null keys
* Contain valid revenue
* Pass all integrity checks


# 7. Key Design Decisions

## Medallion Architecture

Ensures scalability and modularity

## Separation of Concerns

* Bronze → Raw
* Silver → Clean
* Gold → Business

## Currency Conversion in Gold

* Avoids repeated calculations
* Ensures consistency across KPIs

## Fact Table Stores All Records

* No filtering
* Maintains auditability

## KPI Layer Controls Logic

* Flexible business rules
* Easy to modify

--- 

# 8. Conclusion

This pipeline provides:

* Scalable data processing
* Clean and reliable datasets
* Accurate business metrics
* Strong data governance

The design follows **industry best practices** and is suitable for real-world analytics and reporting systems.

---

# 9. Future Enhancements

* Incremental data processing
* Slowly Changing Dimensions (SCD Type 2)
* Data validation frameworks
* Real-time streaming pipelines
* Dashboard integration (Power BI / Tableau)
