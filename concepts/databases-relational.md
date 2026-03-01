# Databases — RDS & Aurora (Relational)

*Phase 3 · Week 5 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

AWS managed relational database services: **RDS** (managed RDBMS on EC2 under the hood) and **Aurora** (cloud-native relational DB with decoupled storage).

---

## How It Works

### RDS — Relational Database Service

Supports: **MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2**.
AWS handles: OS patching, backups, replication, minor version updates.

#### Multi-AZ vs Read Replicas

| Feature | Multi-AZ | Read Replica |
|---|---|---|
| Purpose | **High availability / failover** | **Read scaling / reporting** |
| Replication | Synchronous (standby is always in sync) | Asynchronous (slight lag) |
| Standby accessible? | No — standby is passive | Yes — readable endpoint |
| Failover time | ~60–120 seconds (DNS update) | Manual promotion required |
| Regions | Same region (or Multi-AZ Cluster) | Cross-region supported |
| Cross-account | No | No (but cross-region yes) |

- **Multi-AZ Cluster** — 1 writer + 2 readable standbys in different AZs. Faster failover (<35 s).
- **Up to 15 Read Replicas** per RDS instance.
- Read Replicas can be promoted to standalone DB (breaks replication).

#### Backups

- **Automated backups** — daily full snapshot + transaction logs. Retention 1–35 days. Enables point-in-time recovery (PITR) to any second within retention window.
- **Manual snapshots** — persist after DB is deleted; no expiry unless you delete them.
- Backups stored in S3 (managed by AWS, not visible in your S3 console).

#### Encryption & Security

- Encryption at rest via KMS; must be enabled at creation time.
- To encrypt an unencrypted DB: snapshot → copy with encryption → restore.
- **SSL/TLS** in transit; enforce with `rds.force_ssl` parameter.
- **IAM database authentication** — MySQL, PostgreSQL only. Use IAM auth token (15-min valid) instead of password.
- Security groups control network access; deploy in private subnet.

#### Parameter Groups & Option Groups

- **Parameter group** — DB engine parameters (e.g. `max_connections`, `innodb_buffer_pool_size`).
- **Option group** — engine-specific features (e.g. Oracle TDE, MSSQL SSRS). MySQL/Postgres rarely need this.

#### RDS Proxy

- Sits between app and RDS; pools and shares DB connections.
- Reduces connection overhead for serverless/Lambda workloads (many short-lived connections).
- Enforces IAM authentication; integrates with Secrets Manager for credential rotation.
- Supports MySQL, PostgreSQL, MariaDB, SQL Server, Aurora.
- Deployed in VPC; no internet exposure.

---

### Aurora

Cloud-native relational DB; compatible with **MySQL** and **PostgreSQL** APIs.
**Storage is separate from compute** — 6 copies of data across 3 AZs; self-healing.

#### Cluster Architecture

```
Aurora Cluster Endpoint (writer) ─────► Writer Instance
Aurora Reader Endpoint (load-balanced) ► Reader Instances (up to 15)
```

- **Cluster volume** — automatically grows in 10 GB increments up to 128 TiB.
- **Writer endpoint** — always points to current primary; auto-updates on failover.
- **Reader endpoint** — load-balances across all read replicas.
- **Custom endpoints** — subset of replicas (e.g. for analytics workloads).
- **Failover** — Aurora promotes a replica in <30 seconds (faster than RDS Multi-AZ).
- **Backtrack** — rewind DB to a point in time WITHOUT restoring from backup (MySQL-compatible only). Works within configured backtrack window; not a backup replacement.

#### Aurora Serverless v2

- Scales compute capacity (ACUs — Aurora Capacity Units) up/down in seconds based on demand.
- **v2** (current): scales in fine-grained increments; no cold starts; replicas also scale.
- **v1** (legacy): pauses to zero; slow scale-up; limited compatibility.
- Pay per ACU-second; good for variable/unpredictable workloads.
- Supports both MySQL and PostgreSQL compatibility.

#### Aurora Global Database

- One **primary region** (read/write) + up to 5 **secondary regions** (read-only).
- Replication lag <1 second.
- **Managed failover** — promote secondary to primary in <1 minute RTO.
- Used for: multi-region reads, DR.

#### Aurora Replicas vs RDS Read Replicas

| Feature | Aurora Replica | RDS Read Replica |
|---|---|---|
| Failover | Automatic (if replica exists) | Manual promotion |
| Replication lag | Milliseconds | Seconds to minutes |
| Max replicas | 15 | 15 |
| Endpoints | Reader endpoint (load-balanced) | Individual endpoint per replica |
| Cross-region | Via Global Database | Direct cross-region replication |

---

### Database Migration Service (DMS)

- Migrate databases to AWS with minimal downtime.
- Supports homogeneous (MySQL → RDS MySQL) and heterogeneous (Oracle → Aurora PostgreSQL) migrations.
- **Schema Conversion Tool (SCT)** — converts schema and stored procedures for heterogeneous migrations (run before DMS).
- **CDC (Change Data Capture)** — replicate ongoing changes during migration; keep source running.
- DMS runs on a replication instance (EC2); choose appropriate size for throughput.

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| RDS max storage (gp3) | 64 TiB (MySQL/PostgreSQL); 16 TiB (SQL Server) |
| RDS Read Replicas per instance | 15 |
| RDS automated backup retention | 0–35 days |
| RDS Multi-AZ failover time | ~60–120 s |
| Aurora cluster volume max | 128 TiB |
| Aurora replicas | 15 |
| Aurora Global Database secondary regions | 5 |
| Aurora Global failover RTO | < 1 minute |
| Aurora Serverless v2 min capacity | 0.5 ACU |
| DMS replication instance types | t3, r5, r6i (choose based on data size) |

---

## When To Use

| Scenario | Solution |
|---|---|
| MySQL/PostgreSQL, managed, HA | RDS Multi-AZ |
| Read-heavy workload | RDS + Read Replicas |
| Need full MySQL/PostgreSQL compat + better performance | Aurora |
| Variable/unpredictable DB load | Aurora Serverless v2 |
| Multi-region reads or DR | Aurora Global Database |
| Lambda → RDS (too many connections) | RDS Proxy |
| Rewind DB after bad query | Aurora Backtrack |
| Migrate on-prem Oracle → Aurora PostgreSQL | SCT + DMS with CDC |

---

## Exam Traps

- **Multi-AZ standby is not readable** — it's for failover only. Use Read Replicas for reads.
- **Encryption can't be enabled on existing unencrypted RDS** — must snapshot + copy + restore.
- **Automated backups deleted when DB is deleted** — unless you take a final snapshot. Manual snapshots persist.
- **Aurora Backtrack is MySQL-compatible only** — PostgreSQL Aurora does not support it.
- **Aurora Serverless v2 replicas also scale** — v1 replicas did not scale; this is a key v2 improvement.
- **Aurora Global Database failover = managed promotion** — but you still need to update app connection strings.
- **DMS requires schema first** — SCT (or manual DDL) creates tables before DMS migrates data.
- **RDS Proxy uses IAM + Secrets Manager** — not just a connection pooler; it also handles credential rotation.
- **RDS storage autoscaling** — must be enabled; not on by default for all instance types.
- **Multi-AZ failover is automatic; Read Replica promotion is manual** — these are different features.

---

## Related Files

| File | Purpose |
|---|---|
| `diagrams/databases.md` | Visual: RDS Multi-AZ, Aurora cluster, Global DB topology |
| `cheatsheets/databases.md` | Quick-reference card |
| `mock-exams/phase3.md` | Database practice questions |
| `concepts/databases-nosql.md` | Next concept file (Week 6) |
