# Cheatsheet: Analytics

*Fill in as you complete Week 10. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| Athena pricing | $5 per TB scanned (use partitioning + columnar to save) |
| Athena query engine | Presto (serverless, no infrastructure) |
| Glue Crawler | Auto-discovers schema → populates Data Catalog |
| Glue ETL | Apache Spark under the hood (PySpark, Scala) |
| Glue Job Bookmarks | Track processed data to avoid re-processing |
| EMR primary node | Coordinates cluster (formerly "master") |
| EMR core nodes | Store HDFS data + run tasks (losing one = data loss) |
| EMR task nodes | Compute only, no HDFS (use Spot instances here) |
| Redshift distribution styles | AUTO, EVEN, KEY, ALL |
| Redshift Spectrum | Query S3 data directly (no loading into Redshift) |
| Lake Formation | Permission layer on top of Glue Data Catalog (column/row-level security) |
| QuickSight | Serverless BI/visualisation (SPICE in-memory engine) |

---

## Must-Know Distinctions

- **Athena** → ad-hoc SQL on S3 (serverless, pay-per-scan) | **Redshift** → data warehouse (provisioned, complex joins/aggregations)
- **Glue** → managed ETL (Spark) | **EMR** → full Hadoop ecosystem (more control, more complex)
- **Glue Data Catalog** → metadata store (used by Athena, Redshift Spectrum, EMR, Lake Formation) | **Hive Metastore** → legacy alternative on EMR
- **Redshift** → columnar, OLAP | **RDS** → row-based, OLTP
- **Redshift Spectrum** → query S3 from Redshift (compute on Redshift) | **Athena** → query S3 standalone (no cluster needed)
- **Lake Formation** → fine-grained access (column/row) on data lake | **IAM** → coarse access (bucket/prefix level)
- **Kinesis Data Streams** → real-time (200 ms), you manage consumers | **Kinesis Firehose** → near-real-time (60 s buffer), fully managed delivery
- **MSK** → managed Kafka (existing Kafka workloads) | **Kinesis** → AWS-native streaming (simpler, less operational)

---

## Service Decision Tree

```
Need to query S3 data?
  ├── Ad-hoc / infrequent → Athena
  ├── Complex joins + dashboards → Redshift (load) or Redshift Spectrum (query in place)
  └── ML / big data processing → EMR (Spark/Hive)

Need to transform data?
  ├── Serverless ETL → Glue
  ├── Full Hadoop/Spark control → EMR
  └── Simple column mapping → Glue DataBrew (visual, no code)

Need real-time streaming?
  ├── AWS-native, simple → Kinesis Data Streams
  ├── Existing Kafka workloads → MSK
  └── Delivery to S3/Redshift/OpenSearch → Kinesis Firehose

Need data governance?
  ├── Column/row-level security → Lake Formation
  ├── Tag-based access control → Lake Formation LF-Tags
  └── Cross-account data sharing → Lake Formation + RAM
```

---

## Cost-Saving Patterns

| Pattern | Savings |
|---|---|
| Partition S3 data (year/month/day) | Athena scans less → lower cost |
| Convert CSV → Parquet or ORC | 30-90% less data scanned (columnar + compressed) |
| Use Glue Job Bookmarks | Avoid re-processing already-processed data |
| EMR task nodes on Spot | 60-90% compute savings (no HDFS on task nodes) |
| Redshift reserved nodes | Up to 75% vs on-demand |
| Athena workgroups | Set per-query data scan limits to prevent runaway queries |

---

## Exam Traps (Quick List)

- Athena = serverless; you don't manage any infrastructure — just point at S3
- Glue Crawler discovers schema; it does NOT transform data (that's the ETL Job)
- Glue Data Catalog is shared across Athena, Redshift Spectrum, EMR, Lake Formation
- Redshift is NOT serverless by default (Redshift Serverless exists but is separate)
- EMR core node failure = HDFS data loss; task node failure = no data loss
- Lake Formation permissions supersede IAM S3 policies when Lake Formation is enabled
- Kinesis Firehose can transform data with Lambda before delivery
- MSK is for teams already using Kafka — don't pick MSK for new, simple streaming use cases
- QuickSight SPICE = in-memory; queries are fast but data must fit in SPICE capacity
- Redshift COPY command is the fastest way to load data (not INSERT)

---

## Related Files

`concepts/analytics.md` · `diagrams/analytics.md` · `mock-exams/phase5.md` · `concepts/databases-nosql.md` (Redshift)
