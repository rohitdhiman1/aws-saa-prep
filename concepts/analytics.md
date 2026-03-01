# Analytics — Athena, Glue, EMR, Lake Formation

*Phase 5 · Week 10 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

AWS analytics services for querying, transforming, and governing large-scale data stored primarily in S3 (the data lake) or streaming sources.

---

## How It Works

### Athena

- **Serverless, interactive SQL** on data stored in S3.
- Uses **Presto** under the hood. Supports: CSV, JSON, ORC, Avro, Parquet, compressed formats.
- Pay per TB scanned (~$5/TB). Use compression + columnar formats (Parquet, ORC) to reduce cost.
- Integrated with **Glue Data Catalog** for table/schema definitions.

#### Key Athena Concepts

- **Database + table** = external table pointing to S3 prefix. Data stays in S3; Athena just queries it.
- **Partitioning** — organise S3 data by partition columns (e.g. `year=2024/month=01/day=01/`). Athena prunes partitions → scans less data → cheaper and faster.
- **Partition projection** — define partition values in table properties so Athena doesn't need to query the Glue catalog for every partition; faster for time-series data.
- **Federated queries** — query data in RDS, DynamoDB, Redshift, and other sources via Lambda data source connectors.
- **Workgroups** — isolate queries per team; enforce per-query data scan limits; separate billing.
- **Athena for Spark** — run Apache Spark notebooks via Athena console; serverless Spark.
- **CTAS (Create Table As Select)** — write query results to S3 in a new format (e.g., convert CSV to Parquet).

---

### Glue

Fully managed **ETL (Extract, Transform, Load)** service with a central metadata catalog.

#### Glue Data Catalog

- Central metadata repository for all AWS analytics services.
- **Databases + Tables** — schema definitions pointing to S3, RDS, Redshift, etc.
- Used by: Athena, Redshift Spectrum, EMR, Lake Formation.
- **Glue Crawlers** — scan data sources, infer schema, create/update catalog tables automatically. Scheduled or on-demand.

#### Glue ETL

- **Glue Jobs** — Spark-based ETL scripts (Python or Scala). Run on managed Spark clusters.
- **Glue Studio** — visual drag-and-drop ETL editor; generates code.
- **Glue DataBrew** — no-code data preparation tool (profiling, cleaning, normalising).
- **Glue Streaming** — ETL on Kinesis or Kafka streams (Spark Structured Streaming).
- **Job bookmarks** — track previously processed data to avoid reprocessing on re-runs.
- **Glue Triggers** — schedule jobs or chain them (on-demand, scheduled, event-based).

---

### EMR — Elastic MapReduce

Managed **big data cluster** running open-source frameworks: Hadoop, Spark, Hive, HBase, Flink, Presto, Zeppelin.

#### Cluster Modes

| Mode | Description |
|---|---|
| **EMR on EC2** | Traditional cluster; master + core + optional task nodes |
| **EMR Serverless** | No cluster to manage; auto-scales; pay per vCPU-hour + memory-hour |
| **EMR on EKS** | Run Spark jobs on existing EKS clusters |

#### Node Types (EMR on EC2)

| Node | Role |
|---|---|
| **Primary (master)** | Manages cluster; coordinates YARN/HDFS; 1 per cluster |
| **Core** | Run tasks AND store HDFS data; termination = data loss risk |
| **Task** | Run tasks only; no HDFS; safe for Spot (can be terminated without data loss) |

- Use **Spot for task nodes** (stateless); On-Demand or Reserved for core/master.
- **EMRFS** — use S3 as the persistent store instead of HDFS (decouples storage from compute).
- **Instance fleets** — mix instance types with weighted capacity; more Spot capacity options.
- **Instance groups** — single instance type per group; simpler.

---

### Lake Formation

Build and secure a **data lake** on S3 with fine-grained access control.

- Sits on top of Glue Data Catalog; adds a permissions layer.
- **Column-level and row-level security** — restrict what data specific users/roles can query (via Athena, Redshift Spectrum, EMR).
- **Tag-based access control (LF-tags)** — assign tags to databases/tables/columns; grant permissions via tags (scalable for large catalogs).
- **Data lake administrator** — sets up Lake Formation; grants permissions to analysts.
- **Cross-account access** — share catalogs/tables with other accounts via RAM (Resource Access Manager).
- **Governed tables** — ACID transactions on S3; row-level filtering; automatic compaction.

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| Athena query timeout | 30 minutes |
| Athena max concurrent queries | 25 DDL + 25 DML (default) |
| Glue crawler max concurrency | 25 concurrent crawlers |
| Glue job workers | Up to 299 DPUs (Data Processing Units) |
| EMR max cluster size | No hard limit on EC2 (practical thousands of nodes) |
| Athena price | ~$5 per TB scanned |

---

## When To Use

| Scenario | Solution |
|---|---|
| Ad-hoc SQL on S3 data | Athena |
| Convert CSV to Parquet in S3 | Glue ETL job or Athena CTAS |
| Discover schema of new S3 data | Glue Crawler |
| Large-scale Spark/Hadoop processing | EMR |
| Spark without cluster management | EMR Serverless |
| Query S3 from Redshift | Redshift Spectrum (uses Glue catalog) |
| Fine-grained column/row security on data lake | Lake Formation |
| Share data lake tables cross-account | Lake Formation + RAM |

---

## Exam Traps

- **Athena queries S3; it doesn't move data** — results written to a results S3 bucket. Use CTAS if you want a new table.
- **Partitioning is critical for Athena cost** — without partitions, Athena scans entire dataset.
- **Glue Crawler does not load data** — it only creates/updates metadata in the Data Catalog.
- **EMR core node termination = HDFS data loss** — safe to terminate task nodes; never terminate core nodes without EMRFS.
- **Glue job bookmarks** — without bookmarks, rerunning a job reprocesses all data.
- **Lake Formation ≠ S3 bucket policy** — Lake Formation permissions are additional; S3 bucket policy must also allow access.
- **EMR Serverless = no always-on cluster** — good for batch jobs; not for interactive notebooks needing instant response.
- **Athena Federated Queries need Lambda** — a data source connector (Lambda function) is required; not just a config.

---

## Related Files

| File | Purpose |
|---|---|
| `diagrams/analytics.md` | Visual: S3 data lake → Glue Crawler → Athena → QuickSight pipeline |
| `cheatsheets/analytics.md` | Quick-reference card (add to cheatsheets/security.md or separate) |
| `mock-exams/phase5.md` | Practice questions |
| `concepts/security.md` | Week 10 (part 2) |
