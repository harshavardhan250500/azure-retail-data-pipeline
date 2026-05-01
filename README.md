# Azure End-to-End Retail Data Engineering Pipeline | Medallion Architecture

A production-grade cloud data engineering project built on Microsoft Azure implementing the full **Medallion Architecture (Bronze → Silver → Gold)** on a retail sales dataset, covering ingestion, transformation, warehousing, and interactive dashboards.

---

## Architecture Overview

```
JSON/SQL Source → Azure Data Factory → ADLS Gen2 (Bronze/Parquet) → Azure Databricks (Silver → Gold/Delta) → Power BI
```

| Layer | Tool | Purpose |
|---|---|---|
| Data Ingestion | Azure Data Factory (ADF) | Ingest JSON/SQL source files into Bronze layer as Parquet |
| Bronze Layer | Azure Data Lake Storage Gen2 | Raw data landing zone — no transformations |
| Silver Layer | Azure Databricks (PySpark + Delta Lake) | Cleaned, type-cast, deduplicated, joined data |
| Gold Layer | Azure Databricks (PySpark + Delta Lake) | Aggregated business-ready metrics |
| Visualization | Power BI | Interactive dashboards with DAX measures |

---

## Dataset

Source: 

Multi-source retail dataset comprising four tables:
- `customers.parquet` — customer demographics and registration data
- `products.parquet` — product catalog with pricing and category
- `stores.parquet` — store locations and names
- `transactions.parquet` — sales transaction records

---

## Project Walkthrough

### 1. Azure Resources Setup
- Created Azure Data Lake Storage Gen2 account: `retailproject`
- Created container: `retail` with three directories:
  - `bronze/` — raw ingested data
  - `silver/` — cleaned and joined data
  - `gold/` — aggregated analytical data
- Connected Databricks to ADLS using **Account Key authentication**:

```pyspark
spark.conf.set(
    "fs.azure.account.key.retailproject.dfs.core.windows.net",
    "<STORAGE_ACCOUNT_ACCESS_KEY>"
)
```

> **Note:** For production use, store the access key in **Azure Key Vault** and reference it via a Databricks secret scope instead of hardcoding.

---

### 2. Data Ingestion with Azure Data Factory
- Created ADF pipelines to ingest source files (JSON / SQL) from GitHub into the `bronze/` layer of ADLS, stored as **Parquet** format
- Verified ingestion using:

```pyspark
dbutils.fs.ls("abfss://retail@retailproject.dfs.core.windows.net/")
```

---

### 3. Bronze → Silver Transformation (Azure Databricks)

Read raw Parquet files from the Bronze layer:

```pyspark
base_path = "abfss://retail@retailproject.dfs.core.windows.net/bronze"

df_customers     = spark.read.parquet(f"{base_path}/customer/customers.parquet")
df_products      = spark.read.parquet(f"{base_path}/product/dbo.products.parquet")
df_stores        = spark.read.parquet(f"{base_path}/store/dbo.stores.parquet")
df_transactions  = spark.read.parquet(f"{base_path}/transaction/dbo.transactions.parquet")
```

Applied transformations — type casting, cleaning, deduplication:

```pyspark
from pyspark.sql.functions import col

df_transactions = df_transactions.select(
    col('transaction_id').cast('int'),
    col('customer_id').cast('int'),
    col('product_id').cast('int'),
    col('store_id').cast('int'),
    col('quantity').cast('int'),
    col('transaction_date').cast('date')
)

df_products = df_products.select(
    col('product_id').cast('int'),
    col('product_name'),
    col('category'),
    col('price').cast('double')
)

df_stores = df_stores.select(
    col('store_id').cast('int'),
    col('store_name'),
    col('location')
)

df_customers = df_customers.select(
    "customer_id", "first_name", "last_name", "city", "registration_date"
).dropDuplicates(['customer_id'])
```

Joined all four tables and computed `total_amount`, saved as **Delta table**:

```pyspark
df_silver = df_transactions \
    .join(df_customers, "customer_id") \
    .join(df_products, "product_id") \
    .join(df_stores, "store_id") \
    .withColumn("total_amount", col('quantity') * col('price'))

df_silver.write.mode("overwrite").format("delta").saveAsTable('cleaned_transactions')
```

---

### 4. Silver → Gold Aggregation (Azure Databricks)

Aggregated business-level metrics grouped by date, product, and store:

```pyspark
from pyspark.sql.functions import sum, countDistinct, avg

gold_df = df_silver.groupBy(
    "transaction_date",
    "product_id", "product_name", "category",
    "store_id", "store_name", "location"
).agg(
    sum("quantity").alias("total_quantity_sold"),
    sum("total_amount").alias("total_sales_amount"),
    countDistinct("transaction_id").alias("number_of_transactions"),
    avg("total_amount").alias("average_transaction_value")
)

gold_path = "abfss://retail@retailproject.dfs.core.windows.net/gold/"
gold_df.write.mode("overwrite").format("delta").save(gold_path)
gold_df.write.mode("overwrite").format("delta").saveAsTable("retail_gold_sales_summary")
```

---

### 5. Visualization with Power BI
- Connected Power BI Desktop to the Gold layer via Azure Databricks / Synapse connector
- Built interactive dashboards with DAX measures covering:
  - Total sales by category and store
  - Transaction trends over time
  - Average transaction value by product

---

## Azure Services Used

- Azure Data Factory (ADF)
- Azure Data Lake Storage Gen2 (ADLS)
- Azure Databricks (PySpark + Delta Lake)
- Microsoft Power BI Desktop (DAX)

---

## Key Concepts Demonstrated

- Medallion Architecture — Bronze, Silver, Gold layers
- Delta Lake format for ACID-compliant storage
- Multi-source data ingestion via ADF
- PySpark type casting, deduplication, and multi-table joins
- Derived column computation (`total_amount = quantity × price`)
- Business-level aggregations for analytical reporting
- Star schema design for Power BI data modeling
- DAX measures for calculated KPIs in dashboards

---

## Getting Started

> **Note:** This project requires an active Azure subscription with access to ADLS Gen2, Azure Databricks, ADF, and Power BI Desktop.

1. Clone this repository
2. Create ADLS Gen2 storage account with `bronze/`, `silver/`, `gold/` directories
3. Use ADF to ingest source files into the Bronze layer
4. Run the Databricks notebook in order: Bronze read → Silver transform → Gold aggregate
5. Connect Power BI to the Gold Delta table for dashboarding

> **Security reminder:** Never hardcode storage account keys in notebooks. Use Azure Key Vault with Databricks secret scopes in production.
