# Diagrams: Analytics

*Draw these on paper or in draw.io as part of Week 10 study.*

---

## Diagram 1: S3 Data Lake → Analytics Pipeline

```
Data Sources                Ingestion               Storage (S3 Data Lake)
────────────               ──────────              ─────────────────────────
RDS, Aurora   ──DMS──────► Kinesis Firehose ──────► s3://data-lake/raw/
IoT Devices   ──Kinesis──►                          s3://data-lake/processed/
App logs      ──Firehose──►                          s3://data-lake/curated/

                            Glue Crawler ─────────► Glue Data Catalog
                            (schema discovery)       (tables/databases)

                                                            │
                                            ┌───────────────┼───────────────┐
                                            ▼               ▼               ▼
                                         Athena         Redshift        EMR Spark
                                        (ad-hoc SQL)   (Spectrum)    (batch ETL)
                                            │               │
                                            └───────┬───────┘
                                                    ▼
                                               QuickSight (BI)
```

---

## Diagram 2: Athena Partitioning

```
WITHOUT partitioning:
  Athena scans entire s3://logs/           ← expensive (TBs)

WITH partitioning:
  s3://logs/year=2024/month=01/day=01/*.gz
  s3://logs/year=2024/month=01/day=02/*.gz
  ...

  Query: WHERE year='2024' AND month='01'
  Athena ONLY scans Jan 2024 data          ← much cheaper
```

---

## Diagram 3: Glue ETL Job Flow

```
Source (S3 CSV)
      │
      ▼
Glue Crawler ──► Glue Data Catalog (table schema)
      │
      ▼
Glue ETL Job (PySpark):
  - Read from S3 (CSV)
  - Transform (filter, join, rename)
  - Write to S3 (Parquet, partitioned)
      │
      ▼
Destination (S3 Parquet)
      │
Glue Crawler ──► Catalog updated
      │
      ▼
Athena queries Parquet (faster, cheaper than CSV)
```

---

## Diagram 4: Lake Formation Permission Model

```
S3 Data Lake
      │
Glue Data Catalog (metadata)
      │
Lake Formation (permission layer)
      │
  ┌───┴──────────────────────────────────┐
  │  User: analyst@company.com           │
  │  Grant: SELECT on table.orders       │
  │         EXCEPT column: credit_card   │ ← column-level security
  │         WHERE region='US'            │ ← row-level security
  └──────────────────────────────────────┘
      │
  Athena / Redshift Spectrum / EMR
  (enforces Lake Formation permissions transparently)
```
