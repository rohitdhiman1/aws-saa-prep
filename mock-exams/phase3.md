# Practice Questions: Phase 3 — Databases & Caching

*20 questions · Target score: ≥ 70% · Complete end of Week 6*

---

## Instructions

Answer each question before reading the answer. Write your choice, then compare.
Log wrong answers in `docs/bugbase.md` with the correct reasoning.

---

**Q1.**
A company runs a MySQL RDS instance with Multi-AZ enabled. The primary instance fails. How does RDS handle the failover?

A) RDS promotes the Read Replica to primary
B) RDS automatically promotes the standby instance and updates the DNS endpoint — typically within 60–120 seconds
C) RDS restores the primary from the most recent automated snapshot
D) A manual failover must be triggered by the DBA

**Answer: B** — With Multi-AZ, RDS maintains a synchronous standby replica in a different AZ. On failure, RDS automatically promotes the standby and updates the cluster DNS endpoint. Applications using the endpoint reconnect transparently. The failover takes 60–120 seconds. Multi-AZ standby is NOT a Read Replica (A) — it cannot serve read traffic. The standby is hot; no snapshot restore is needed (C). Failover is automatic (D).

---

**Q2.**
A company wants to offload read traffic from their RDS MySQL primary. They create a Read Replica. A developer reports that data read from the replica is sometimes stale. What is the cause?

A) Read Replicas use synchronous replication — staleness indicates a bug
B) Read Replicas use asynchronous replication — there is inherent replication lag
C) The Read Replica is not in the same VPC as the primary
D) Encryption is not enabled on the Read Replica

**Answer: B** — RDS Read Replicas use asynchronous replication. There is always some lag between a write on the primary and its appearance on the replica. Applications must tolerate eventual consistency when using Read Replicas. Multi-AZ standby uses synchronous replication, but that's for HA, not read scaling. VPC or encryption settings (C, D) don't affect replication staleness.

---

**Q3.**
A developer wants to create an encrypted RDS snapshot from an unencrypted RDS instance to share with another team. What is the correct process?

A) Enable encryption on the existing instance — all future snapshots will be encrypted
B) Create a snapshot of the unencrypted instance, copy the snapshot and enable encryption during the copy, share the encrypted snapshot
C) Create a Read Replica and enable encryption on the replica
D) RDS does not support encrypting snapshots from unencrypted instances

**Answer: B** — You cannot encrypt an existing unencrypted RDS instance or snapshot in-place. The process is: create unencrypted snapshot → copy snapshot with encryption enabled → the resulting copy is encrypted. You can then share this encrypted snapshot. Enabling encryption (A) only applies to new instances. A Read Replica (C) cannot be encrypted if the source is not.

---

**Q4.**
An Aurora MySQL cluster has a writer instance and two reader instances. A client application connects to the cluster endpoint for writes and the reader endpoint for reads. The writer instance fails. What happens to the cluster endpoint?

A) The cluster endpoint becomes unavailable until the writer is manually restored
B) Aurora automatically promotes one of the readers to writer and updates the cluster endpoint DNS — typically within 20–30 seconds
C) Aurora creates a brand new writer instance from scratch
D) The reader endpoint takes over write traffic automatically

**Answer: B** — Aurora failover promotes an existing reader to writer. The cluster (writer) endpoint DNS is updated to point to the new writer. This typically takes 20–30 seconds — much faster than standard RDS Multi-AZ (~60–120s) because Aurora's storage is decoupled from compute; the new writer doesn't need to replay logs. The reader endpoint (D) is read-only and does not accept writes.

---

**Q5.**
A company needs Aurora to support both OLTP workloads (consistent, low-latency) and reporting queries (complex, potentially slow). How should this be architected?

A) Run all traffic against the writer instance
B) Use the reader endpoint for reporting — Aurora load-balances across all reader instances
C) Create a custom endpoint that includes only specific reader instances designated for reporting, separate from the OLTP reader endpoint
D) Deploy a second Aurora cluster solely for reporting

**Answer: C** — Aurora custom endpoints let you create an endpoint that maps to a specific subset of reader instances (e.g., larger instance types for reporting). OLTP readers use the standard reader endpoint; reporting queries use the custom endpoint. This isolates slow reporting queries from OLTP reader traffic. Using all readers for reporting (B) can degrade OLTP read performance.

---

**Q6.**
A DynamoDB table has `UserID` (partition key) and `Timestamp` (sort key). A query needs to retrieve all items for a specific `UserID` sorted by `Timestamp` in descending order. Which operation should be used?

A) Scan with a `FilterExpression` on UserID
B) Query on the partition key `UserID` with `ScanIndexForward = false`
C) Query on both `UserID` and `Timestamp` with a begins_with condition
D) BatchGetItem with a list of UserID + Timestamp pairs

**Answer: B** — Query on the partition key returns all items for that `UserID`, and DynamoDB stores sort key values in sorted order. Setting `ScanIndexForward = false` reverses the sort order to descending. Scan (A) reads the entire table — expensive and slow. `begins_with` on Timestamp (C) would filter by timestamp prefix, not sort order. BatchGetItem (D) requires you to know the exact keys upfront.

---

**Q7.**
A DynamoDB table has a `UserID` partition key. The team needs to query by `Email` address (not in the table's key schema). Which is the correct solution?

A) Perform a full table Scan with a `FilterExpression` on Email
B) Add a Global Secondary Index (GSI) with `Email` as the partition key
C) Add a Local Secondary Index (LSI) with `Email` as the sort key
D) Create a new table with `Email` as the partition key and replicate data there

**Answer: B** — A GSI can use any attribute as the partition key and allows efficient Query operations on that attribute. GSIs can be added to existing tables. LSIs (C) use the same partition key as the base table — they can only change the sort key, and they must be created at table creation time. Scan (A) works but reads the whole table (expensive at scale). A separate table (D) requires manual data sync.

---

**Q8.**
A company wants to implement the write-through caching pattern with ElastiCache Redis in front of DynamoDB. Which describes write-through correctly?

A) The application writes to the cache. A background process asynchronously writes to the database.
B) The application writes to both the cache and the database synchronously on every write
C) The application writes to the database only. The cache is populated on the next read.
D) The application writes to the cache. On cache eviction, the data is flushed to the database.

**Answer: B** — Write-through: every write goes to both the cache and the database simultaneously. The cache is always current but writes are slower (two operations). Cache-aside (lazy loading): app checks cache → miss → reads DB → writes to cache (C describes this). Write-behind/write-back (A) is async to the DB — risk of data loss on cache failure.

---

**Q9.**
A DynamoDB table receives heavy write traffic. The team notices `ProvisionedThroughputExceededException` errors at certain times even though total writes are within the provisioned capacity. What is the most likely cause?

A) The table is using On-Demand capacity mode which has hard limits
B) Writes are concentrated on a few partition keys (hot partition), exceeding per-partition throughput limits
C) DynamoDB tables can only handle writes from a single region
D) The GSI's write capacity is consuming the table's capacity

**Answer: B** — DynamoDB distributes data across partitions by partition key. Each partition has a throughput limit. If writes concentrate on a few keys (e.g., using a timestamp as the partition key), those partitions are overloaded while others are idle — even if total capacity looks fine. The solution is to design partition keys with high cardinality (e.g., add a random suffix — "write sharding"). On-Demand mode (A) scales automatically without ProvisionedThroughputExceeded errors.

---

**Q10.**
An ElastiCache Redis cluster has Multi-AZ enabled with one replica. The primary node fails. What is the failover behavior?

A) The cluster is unavailable until the primary is manually restored
B) The replica is automatically promoted to primary; the cluster endpoint's DNS record updates to point to the new primary
C) ElastiCache creates a new primary node from a backup snapshot
D) The replica continues to serve both reads and writes immediately

**Answer: B** — With Multi-AZ and automatic failover enabled, ElastiCache promotes the replica to primary and updates the primary endpoint DNS record. There is a brief outage (~20–30 seconds) during promotion. The replica cannot serve writes until it's promoted (D is wrong). No manual intervention or snapshot restore is needed.

---

**Q11.**
A company's RDS MySQL instance needs to be migrated to Aurora MySQL with minimal downtime (less than 1 minute of outage). Which approach achieves this?

A) Create an Aurora Read Replica from the RDS instance, then promote it
B) Take an RDS snapshot and restore it to a new Aurora cluster
C) Use AWS DMS to replicate data from RDS to Aurora, then cut over
D) Both A and C are valid approaches with minimal downtime

**Answer: D** — Both approaches support minimal downtime. Creating an Aurora Read Replica from RDS: replication happens live, then you promote (brief cutover window). DMS with ongoing replication: continuous sync until you cut over. Snapshot restore (B) requires stopping writes to RDS for a consistent snapshot, which causes more downtime. The exam often accepts both A and C as correct for low-downtime migrations.

---

**Q12.**
A DynamoDB table uses TTL (Time To Live). An item's TTL attribute is set to a timestamp 1 hour in the past. What happens to the item?

A) DynamoDB deletes the item exactly when the TTL timestamp is reached
B) DynamoDB deletes the item within 48 hours of the TTL expiry; deletion is not guaranteed to be immediate
C) The item is moved to an expired items table for audit
D) The item remains in the table but is invisible to Query and Scan operations

**Answer: B** — DynamoDB TTL deletion is eventual, not real-time. Items past their TTL timestamp may persist for up to 48 hours before DynamoDB's background cleanup process removes them. They are still readable by Query/Scan during this window (D is wrong). Applications that need precise expiry must filter on the TTL attribute themselves. There is no audit table (C).

---

**Q13.**
A company wants to add caching to an existing DynamoDB-backed application without changing application code. The data is read-heavy with infrequent updates. Which solution requires no application code changes?

A) ElastiCache Redis with the cache-aside pattern
B) DAX (DynamoDB Accelerator)
C) DynamoDB global tables
D) DynamoDB Streams with Lambda to pre-warm ElastiCache

**Answer: B** — DAX is a fully managed, in-memory cache that sits in front of DynamoDB and is API-compatible. The application uses the same DynamoDB SDK calls — the DAX client intercepts them, checks the cache, and falls back to DynamoDB on a miss. No application code changes beyond swapping the client. ElastiCache (A) requires explicit cache-aside logic in the application. Global tables (C) are for multi-region replication, not caching.

---

**Q14.**
An Aurora Global Database spans us-east-1 (primary) and eu-west-1 (secondary). A regional disaster makes us-east-1 unavailable. What must the operations team do to resume write operations?

A) Nothing — the secondary automatically becomes the primary
B) Manually promote the eu-west-1 secondary cluster to a standalone primary and update application connection strings
C) Restore the us-east-1 cluster from the most recent automated backup
D) Enable Multi-AZ on the eu-west-1 cluster — it will then accept writes

**Answer: B** — Aurora Global Database does NOT automatically fail over cross-region. The secondary is read-only. An operator must manually promote the secondary cluster (which detaches it from the global database and makes it a standalone writable cluster), then update application connection strings to point to eu-west-1. RTO is typically under 1 minute for the promotion itself, but the application switch is manual. This is a common exam distinction vs. Aurora Multi-AZ, which IS automatic.

---

**Q15.**
A company stores session data in ElastiCache Redis. The data must survive an ElastiCache node failure. Which Redis feature should be enabled?

A) Read replicas
B) Redis Cluster Mode
C) Redis persistence (AOF or RDB snapshots)
D) Pub/Sub channels

**Answer: C** — AOF (Append Only File) logs every write to disk; RDB takes periodic snapshots. If a node fails and restarts, Redis replays the AOF or loads the RDB to restore state. Read replicas (A) help with HA for in-memory data but don't persist data to disk — if both primary and replica restart simultaneously, data is lost. Cluster Mode (B) shards data across nodes but doesn't persist it. Pub/Sub (D) is a messaging pattern.

---

**Q16.**
A DynamoDB table processes orders. Each order has a unique `OrderID` (partition key). The team wants to automatically trigger a Lambda function whenever an order is created, updated, or deleted. Which feature enables this?

A) DynamoDB TTL
B) DynamoDB Streams with a Lambda event source mapping
C) EventBridge Pipes connected to DynamoDB
D) DynamoDB Accelerator (DAX)

**Answer: B** — DynamoDB Streams captures a time-ordered record of item modifications (INSERT, MODIFY, REMOVE). An event source mapping on a Lambda function polls the stream and invokes Lambda with batches of change records. This is the standard pattern for change-driven Lambda triggers from DynamoDB. TTL (A) handles expiry. EventBridge Pipes (C) can also connect to Streams but is a newer, less exam-tested pattern. DAX (D) is a read cache.

---

**Q17.**
An RDS Multi-AZ deployment has both the primary and standby in the same VPC but different AZs. During a maintenance window, AWS applies a patch to the primary. What is the expected behavior?

A) The primary is patched in-place with a brief pause — no failover occurs
B) AWS fails over to the standby, patches the former primary (now standby), then fails back
C) Both the primary and standby are patched simultaneously
D) RDS takes a snapshot first, then patches the primary in-place

**Answer: B** — For Multi-AZ deployments during maintenance, AWS performs a rolling update: it fails over to the standby, patches the now-demoted primary (which becomes the new standby), and may or may not fail back. The application experiences only the failover window (~60s). This minimizes downtime compared to patching in-place on the primary.

---

**Q18.**
A company uses DynamoDB with On-Demand capacity mode. What is true about On-Demand mode? (Select TWO)

A) You must specify a maximum capacity limit to prevent cost overruns
B) DynamoDB automatically scales read and write capacity to match the actual request rate
C) On-Demand capacity is more expensive per request-unit than Provisioned at steady-state load
D) On-Demand mode requires DynamoDB Streams to be enabled
E) Switching from Provisioned to On-Demand resets the table's previous peak traffic history

**Answer: B, C** — On-Demand mode auto-scales instantly with no capacity planning, making it ideal for unpredictable or spiky traffic. However, it costs more per request unit than Provisioned mode at steady, predictable load — Provisioned with auto-scaling is cheaper for consistent workloads. There's no max limit to set (A) — that's a pro and a con. Streams (D) are independent of capacity mode. Switching modes (E) retains the previous peak traffic data that DynamoDB uses to estimate initial capacity.

---

**Q19.**
An application uses ElastiCache Redis (cluster mode disabled) with one primary and two read replicas. The team notices that all write traffic is going to the primary, creating a bottleneck. What can they do to scale writes?

A) Add more read replicas to the cluster
B) Enable cluster mode — this allows writes to be distributed across multiple primary shards
C) Upgrade the primary node type to a larger instance
D) Move to ElastiCache Memcached

**Answer: B** — Cluster mode disabled means a single primary handles all writes — you can only scale reads (with replicas) or vertically scale the primary. Cluster mode enabled shards data across multiple primary nodes (each with their own replicas), allowing horizontal write scaling. Adding replicas (A) only helps reads. Vertical scaling (C) helps but has limits. Memcached (D) doesn't support replication or persistence.

---

**Q20.**
A company queries a DynamoDB table using `Scan` with a `FilterExpression`. The team notices that the operation is very slow and consuming a lot of read capacity. What is the reason and the recommended fix?

A) Reason: `FilterExpression` is applied after reading all items. Fix: add a GSI and use `Query` instead.
B) Reason: Scan is limited to 10 items per request. Fix: increase the page size.
C) Reason: `FilterExpression` cannot be used with DynamoDB. Fix: use `ConditionExpression` instead.
D) Reason: The table's provisioned read capacity is too low. Fix: increase RCUs.

**Answer: A** — `Scan` reads the entire table (or a parallel segment) and then applies the `FilterExpression` to discard non-matching items. You pay in RCUs for every item read, including the ones filtered out. For frequent queries on non-key attributes, a GSI with a `Query` operation reads only matching items, dramatically reducing cost and latency. Increasing RCUs (D) makes it faster but doesn't fix the inefficiency.

---

## Score Log

| Date | Score | Notes |
|---|---|---|
| | /20 | |
