# ShopBR: AWS Data Pipeline

> Serverless data pipeline for a fictitious e-commerce, following the **Medallion Architecture** (Bronze → Silver → Gold) on AWS.

---

## Overview

The ShopBR project builds a fully automated data pipeline on AWS for ingesting, cleaning, transforming, and serving order data for business analytics with no manual intervention.

Data arrives as raw CSV files and is progressively refined until it is ready for analytical SQL queries via Amazon Athena.

---

## Presentations

| Language | File |
|---|---|
| 🇧🇷 Portuguese | [📄 Download PT](https://raw.githubusercontent.com/joycemsm/etl-ecommerce/main/ShopBR_AWS_PT.pdf) |
| 🇺🇸 English | [📄 Download EN](https://raw.githubusercontent.com/joycemsm/etl-ecommerce/main/ShopBR_AWS_EN.pdf) |

---

## Architecture

```
CSV (upload) → S3 Bronze → Lambda (trigger) → Glue Job Silver → S3 Silver → Glue Job Gold → S3 Gold → Athena
                                   ↓
                             CloudWatch (logs)
```

**Services used:**

| Service | Role |
|---|---|
| Amazon S3 | Data lake with three layers (bronze, silver, gold) |
| AWS Lambda | Automatic trigger when a new file is detected in S3 |
| AWS Glue | PySpark jobs for transformation and Crawlers for cataloging |
| Glue Data Catalog | Central catalog of schemas and tables |
| Amazon Athena | Serverless SQL queries directly on S3 data |
| Amazon CloudWatch | Pipeline logs and monitoring |

**Region:** `us-east-1`  
**Main bucket:** `shopbr-datalake-joyce`

---

## Bronze Layer — Ingestion

Stores **raw data** exactly as it arrived. No transformations are applied.

**S3 path:** `s3://shopbr-datalake-joyce/bronze/pedidos/`  
**Format:** CSV  
**Trigger:** `s3:ObjectCreated` event automatically invokes the Lambda function `lambda-trigger-glue-shopbr`

**Bronze schema (`table_pedidos`):**

| Column | Type |
|---|---|
| pedido_id | string |
| cliente_id | string |
| produto | string |
| categoria | string |
| valor | double |
| status | string |
| estado | string |
| data_pedido | string |

---

## Silver Layer — Processing

**Cleaned, typed, and partitioned** data. The Silver Glue Job applies transformations via PySpark.

**S3 path:** `s3://shopbr-datalake-joyce/silver/pedidos_limpos/`  
**Format:** Parquet, partitioned by `estado`  
**Glue Job:** `job` (Glue ETL Script, version 5.0)

**Transformations applied:**

- Drops rows where `pedido_id` or `valor` is null
- Filters out orders with `status = 'cancelado'`
- Casts `valor` → `double`
- Casts `data_pedido` → `DATE` (format `yyyy-MM-dd`)
- Converts `estado` → uppercase

**Partitions created:** AM, BA, GO, MG, PR, RJ, RS, SC, SP

> **Why Parquet?** Athena charges per data scanned. The columnar format + Parquet compression reduces query cost by up to **90%** compared to the original CSV.

---

## Gold Layer — Analytics

**Aggregated data**, ready for consumption by the business team.

**S3 path:** `s3://shopbr-datalake-joyce/gold/vendas_mensais/`  
**Format:** Parquet  
**Glue Job:** `jobGold` (Glue ETL Script, version 5.0)

**What the Gold layer produces:**

- Reads clean data from the Silver layer
- Creates the `mes` column (format `yyyy-MM`) from `data_pedido`
- Groups by `estado` and `mes`
- Calculates `total_vendas` (sum of valor) and `qtd_pedidos` (count)
- Writes with `.mode('overwrite')` to ensure idempotency

**Gold schema (`vendas_mensais`):**

| Column | Type |
|---|---|
| estado | string |
| mes | string |
| total_vendas | double |
| qtd_pedidos | bigint |

---

## Cataloging — Glue Crawlers

Crawlers scan S3 after each job, detect the schema, and register tables in the **Glue Data Catalog**.

| Crawler | Status | Database |
|---|---|---|
| crawler-shopbr-bronze | Ready | shopbr_catalog_bronze |
| crawler-shopbr-silver | Ready | shopbr_catalog_silver |
| crawler-shopbr-gold | Ready | shopbr_catalog_gold |

> **Important:** The Glue Job reads **directly from S3** and does not depend on the Crawler. The Crawler exists so Athena can locate and interpret the data through the Data Catalog. They are independent automations.

---

## Athena Queries

Athena runs SQL directly on S3 files, with no need to load data into a database. It uses the Glue Data Catalog to locate data and resolve schemas.

### Query 1 — Sales by Category
```sql
SELECT
    categoria,
    COUNT(*) AS total_pedidos,
    ROUND(SUM(valor), 2) AS receita_total,
    ROUND(AVG(valor), 2) AS ticket_medio
FROM shopbr_catalog_silver.pedidos_limpos
GROUP BY categoria
ORDER BY receita_total DESC;
```

**Result:** Eletrônicos leads with R$ 5,996.90 and an average ticket of R$ 1,499.23.

---

### Query 2 — Sales by State
```sql
SELECT
    estado,
    COUNT(*) AS total_pedidos,
    ROUND(SUM(valor), 2) AS receita_total
FROM shopbr_catalog_silver.pedidos_limpos
GROUP BY estado
ORDER BY receita_total DESC
LIMIT 10;
```

**Result:** RS leads with R$ 3,499.00, followed by SP (R$ 1,598.90).

---

### Query 3 — Orders by Status
```sql
SELECT
    status,
    COUNT(*) AS total,
    ROUND(SUM(valor), 2) AS valor_total
FROM shopbr_catalog_bronze.table_pedidos
GROUP BY status;
```

**Result:** 6 delivered (`entregue`), 1 cancelled (`cancelado`), 1 in transit (`em_transito`).

---

### Query 4 — Top 5 Products
```sql
SELECT
    produto,
    categoria,
    COUNT(*) AS vezes_vendido,
    ROUND(SUM(valor), 2) AS receita
FROM shopbr_catalog_silver.pedidos_limpos
GROUP BY produto, categoria
ORDER BY receita DESC
LIMIT 5;
```

**Result:** Notebook Pro (R$ 3,499.00), Smartphone X (R$ 1,299.90), Tablet 10 (R$ 899.00).

---

## Technical Decisions

| Decision | Rationale |
|---|---|
| **Manual polling in Lambda** | Glue's native waiter blocks execution. Lambda has a 15-min timeout and Glue jobs can run longer. Polling with a configurable interval is more resilient. |
| **Parquet instead of CSV** | Columnar format reads only the necessary columns. Reduces Athena query cost by up to 90%. |
| **Partitioning by `estado`** | A query filtering for SP reads only the `estado=SP` partition, not the entire dataset. Performance and cost scale well as volume grows. |
| **`.mode('overwrite')`** | Ensures idempotency — the pipeline can be re-executed without generating duplicates. |

---

## Next Steps

-  Migrate to region `sa-east-1` (São Paulo) for lower latency in Brazil
-  Set up VPC and granular IAM with least-privilege roles per service
-  Schedule Crawlers with CRON for automatic cataloging
-  Separate pipelines per domain (customers, products, inventory)
-  Create saved Views in Athena for the business team
-  Configure CloudWatch alerts for pipeline failures

Thanks!
