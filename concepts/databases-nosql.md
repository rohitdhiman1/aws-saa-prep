# Databases — NoSQL, Caching & Analytics Stores

*Phase 3 · Week 6 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

Non-relational and specialised data stores: **DynamoDB** (key-value/document), **ElastiCache** (in-memory caching), **Redshift** (data warehousing), and **Neptune** (graph).

---

## How It Works

### DynamoDB

- Fully managed, serverless, **key-value and document** store.
- Single-digit millisecond latency at any scale.
- Data replicated across 3 AZs automatically.
- No joins; design for access patterns first (unlike relational design).

#### Keys & Indexes

- **Primary key** — either:
  - **Partition key (PK)** only — must be unique per item.
  - **Composite key: PK + Sort key (SK)** — PK can repeat; PK+SK must be unique.
- DynamoDB hashes the PK to determine the partition — distribute writes by choosing high-cardinality PKs.

| Index | Keys | Storage | Reads |
|---|---|---|---|
| **LSI (Local Secondary Index)** | Same PK, different SK | Same partition | Strongly consistent possible |
| **GSI (Global Secondary Index)** | Any PK + optional SK | Separate partition storage | Eventually consistent only |

- Can create up to 5 LSIs (at table creation only) and 20 GSIs (can add later).
- GSIs have their own provisioned or on-demand capacity.

#### Read/Write Capacity Modes

| Mode | Description | Use case |
|---|---|---|
| **Provisioned** | Set RCU (Read Capacity Units) + WCU (Write Capacity Units) | Predictable, steady traffic; cheaper |
| **On-demand** | Pay per request; scales automatically | Unpredictable/spiky traffic; simpler ops |

- **1 RCU** = 1 strongly consistent read/sec of item ≤ 4 KB (or 2 eventually consistent reads).
- **1 WCU** = 1 write/sec of item ≤ 1 KB.
- **Auto Scaling** available in Provisioned mode.

#### Consistency Models

- **Eventually consistent** — default; reads may return stale data (up to ~1 s lag).
- **Strongly consistent** — always returns latest data; costs 2x RCUs; not available on GSIs.
- **Transactional** — ACID across multiple items/tables; costs 2x reads and 2x writes.

#### Advanced Features

- **DynamoDB Streams** — ordered stream of item-level changes (insert, update, delete). Retained 24 hours. Triggers Lambda for real-time processing.
- **DAX (DynamoDB Accelerator)** — in-memory cache (microsecond latency). Write-through cache. Compatible with DynamoDB API. No need to change app code significantly.
- **TTL (Time To Live)** — set an expiry attribute; DynamoDB auto-deletes expired items (within 48 hrs; no cost).
- **Global Tables** — multi-region, multi-active (all regions read and write). Requires on-demand mode or auto scaling in provisioned mode. Uses Streams under the hood.
- **Point-in-Time Recovery (PITR)** — continuous backups for up to 35 days. Restore to any second.
- **On-demand backup** — full table backup; no performance impact; retained until deleted.
- **Transactions** — `TransactWriteItems` / `TransactGetItems`; all-or-nothing across up to 100 items.

---

### ElastiCache

Managed in-memory caching — **Redis** or **Memcached**.

#### Redis vs Memcached

| Feature | Redis | Memcached |
|---|---|---|
| Data structures | Strings, lists, sets, sorted sets, hashes, streams | Simple key-value only |
| Persistence | Yes (RDB + AOF) | No |
| Replication | Yes (primary + replicas) | No |
| Multi-AZ / Failover | Yes (automatic failover) | No |
| Cluster mode | Yes (horizontal sharding) | Yes (multi-node, no replication) |
| Pub/Sub | Yes | No |
| Sorted sets | Yes | No |
| Use case | Session store, leaderboards, pub/sub, queues, ML feature store | Simple caching, horizontal scale |

#### Caching Strategies

| Strategy | How it works | When data might be stale |
|---|---|---|
| **Lazy loading** | Read from cache; on miss, load from DB → write to cache | Yes — first read after eviction |
| **Write-through** | Write to DB AND cache on every write | Never stale (but may cache unused data) |
| **Session store** | Store user session in Redis with TTL | Managed via TTL |

- **ElastiCache with Redis Cluster Mode:** data sharded across multiple shards (each shard = 1 primary + up to 5 replicas).
- **ElastiCache Global Datastore** — cross-region replication for Redis; near-real-time replication.

---

### Redshift

Managed **columnar data warehouse** for analytics (OLAP), not OLTP.
Based on PostgreSQL; supports standard SQL.

#### Architecture

- **Leader node** — receives queries, parses, develops execution plan.
- **Compute nodes** — execute the plan; store data (columnar).
- **Node types:** RA3 (managed storage, scales independently of compute), DC2 (dense compute, fast SSD).
- **Redshift Serverless** — no cluster to manage; pay per RPU-second (Redshift Processing Unit).

#### Distribution Styles (key for performance)

| Style | How data is distributed | Best for |
|---|---|---|
| **AUTO** | Redshift decides | Default; good for new tables |
| **EVEN** | Round-robin across nodes | No clear join key; large table |
| **KEY** | Rows with same key on same node | Tables frequently joined on that key |
| **ALL** | Entire table on every node | Small dimension tables |

#### Other Redshift Concepts

- **Redshift Spectrum** — query S3 data directly from Redshift without loading it. Needs external table defined in Glue Data Catalog.
- **COPY command** — bulk load from S3, EMR, DynamoDB, or remote hosts. Much faster than INSERT.
- **Concurrency Scaling** — automatically adds clusters to handle query surges; first 1 hour/day free.
- **Snapshots** — automated (with retention) or manual. Can restore to new cluster. Cross-region copy supported.
- **Enhanced VPC Routing** — forces all COPY/UNLOAD traffic through VPC (required for VPC endpoints / security compliance).

---

### Neptune

- Managed **graph database** (property graph + RDF).
- Supports **Gremlin** (Apache TinkerPop) and **SPARQL** (RDF/SPARQL), plus **openCypher**.
- Use cases: social networks, fraud detection, recommendation engines, knowledge graphs.
- Multi-AZ; up to 15 read replicas; Global Database available.
- Not commonly tested deeply — know the use case (graph relationships) vs DynamoDB (key-value).

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| DynamoDB item max size | 400 KB |
| DynamoDB LSI | 5 per table (create at table creation only) |
| DynamoDB GSI | 20 per table |
| DynamoDB Streams retention | 24 hours |
| DynamoDB PITR window | 35 days |
| DynamoDB Global Tables max regions | No hard limit (practical: pick based on latency) |
| ElastiCache Redis max cluster nodes | 500 |
| ElastiCache Redis max replicas per shard | 5 |
| Redshift max nodes per cluster | 128 (RA3) |
| Redshift Spectrum | Requires Glue Data Catalog or Athena |

---

## When To Use

| Scenario | Solution |
|---|---|
| High-scale key-value, serverless | DynamoDB on-demand |
| Predictable, high-volume key-value | DynamoDB provisioned + Auto Scaling |
| Cache DB query results, session store | ElastiCache Redis |
| Simple horizontal caching, no persistence | ElastiCache Memcached |
| Leaderboard, sorted scores | ElastiCache Redis (sorted sets) |
| Ad-hoc analytics on large dataset | Redshift |
| Query S3 data lake without loading | Redshift Spectrum or Athena |
| Social network, fraud patterns | Neptune |
| Real-time DB event triggers | DynamoDB Streams + Lambda |

---

## Exam Traps

- **GSIs only support eventual consistency** — cannot do strongly consistent reads on GSI.
- **LSIs must be created at table creation** — cannot add after. GSIs can be added later.
- **DAX is write-through** — it caches reads; writes go to DynamoDB AND invalidate/update cache. Does NOT help write-heavy workloads.
- **DynamoDB Streams retention = 24 hours** — not configurable; process promptly.
- **Global Tables requires Streams enabled** — Streams is the replication mechanism.
- **DynamoDB TTL is not instant** — items expire within 48 hours of TTL timestamp; they may still appear in reads until actually deleted.
- **Redshift is OLAP, not OLTP** — for transactional workloads use RDS/Aurora.
- **Redshift Spectrum needs Glue** — or Athena data catalog; it does not auto-discover S3.
- **ElastiCache Redis cluster mode vs non-cluster** — cluster mode = sharding (horizontal scale of writes); non-cluster = single shard with replicas (scale reads only).
- **Neptune is not a general-purpose DB** — only use when data is fundamentally graph-shaped (relationships between entities matter more than the entities themselves).

---

## Related Files

| File | Purpose |
|---|---|
| `diagrams/databases.md` | Visual: DynamoDB key design, ElastiCache topology, Redshift cluster |
| `cheatsheets/databases.md` | Quick-reference card |
| `mock-exams/phase3.md` | Database practice questions |
| `concepts/networking-advanced.md` | Next concept file (Week 7) |
