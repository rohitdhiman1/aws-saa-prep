# Lab: ElastiCache Redis — Cache-Aside Pattern, Failover & Cluster Mode

*Phase 3 · NoSQL & Caching · Concept: [concepts/databases-nosql.md](../concepts/databases-nosql.md)*

---

## Goal

Create an ElastiCache Redis cluster, implement the cache-aside (lazy loading) pattern using Lambda + DynamoDB, observe cache hit vs miss latency, force a failover, and see replica promotion.

---

## Architecture

```
Lambda (reads data)
  │
  ├──► Redis (ElastiCache) ── hit? return immediately
  │         │ miss?
  └──────────► DynamoDB (source of truth) → write to Redis → return
```

---

## Part A — Create the ElastiCache Redis Cluster

- [ ] ElastiCache → Redis caches → Create.
- [ ] Cluster mode: **Disabled** (single shard = simpler; cluster mode = sharding).
- [ ] Name: `lab-redis`. Node type: `cache.t3.micro`.
- [ ] Replicas: **1** (1 primary + 1 replica = Multi-AZ failover).
- [ ] Multi-AZ: **Yes**. Automatic failover: **Yes**.
- [ ] Subnet group: select subnets in your VPC (private subnets ideal).
- [ ] Security group: create one — allow TCP 6379 inbound from your Lambda SG.
- [ ] Create. (~5 min)

Note the **Primary endpoint** after creation (e.g. `lab-redis.xxxxx.ng.0001.use1.cache.amazonaws.com:6379`).

---

## Part B — Create a DynamoDB Table as the Data Source

- [ ] DynamoDB → Create table: `Products`. PK: `ProductID` (String). On-demand.
- [ ] Add 3 items:
  - `{ProductID: "P001", Name: "Widget", Price: 9.99}`
  - `{ProductID: "P002", Name: "Gadget", Price: 24.99}`
  - `{ProductID: "P003", Name: "Doohickey", Price: 4.99}`

---

## Part C — Lambda: Cache-Aside Pattern

**1. Create Lambda function in the same VPC (private subnet)**
- [ ] Lambda → Create function: `cache-aside-demo`. Runtime: Python 3.12.
- [ ] VPC: same as Redis (so it can reach the private endpoint). SG: allow outbound to Redis SG on 6379.
- [ ] Execution role: attach `AmazonDynamoDBReadOnlyAccess`.

**2. Add the `redis` library as a Lambda Layer**
- [ ] Create a Lambda Layer with `redis` package, or use a public ARN if available, or package it in a zip.
  Quickest: add the layer ARN `arn:aws:lambda:us-east-1:770693421928:layer:Klayers-p312-redis:1` (community layer).

**3. Lambda code:**
```python
import boto3, redis, json, time

REDIS_HOST = "<your-primary-endpoint-without-port>"
REDIS_PORT = 6379
CACHE_TTL = 60  # seconds

r = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=True)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Products')

def lambda_handler(event, context):
    product_id = event.get('ProductID', 'P001')
    cache_key = f"product:{product_id}"

    # 1. Check cache
    start = time.time()
    cached = r.get(cache_key)
    if cached:
        elapsed = round((time.time() - start) * 1000, 2)
        return {'source': 'CACHE (Redis)', 'data': json.loads(cached), 'ms': elapsed}

    # 2. Cache miss - fetch from DynamoDB
    response = table.get_item(Key={'ProductID': product_id})
    item = response.get('Item')
    if not item:
        return {'error': 'Product not found'}

    # 3. Write to cache
    r.setex(cache_key, CACHE_TTL, json.dumps(item, default=str))

    elapsed = round((time.time() - start) * 1000, 2)
    return {'source': 'DB (DynamoDB)', 'data': item, 'ms': elapsed}
```
- [ ] Deploy.

**4. Test cache hit vs miss:**
- [ ] Test with `{"ProductID": "P001"}` → `source: DB (DynamoDB)` — cache miss, slow (~30–50ms).
- [ ] Test again immediately → `source: CACHE (Redis)` — cache hit, fast (~1–3ms).
- [ ] Wait 60 seconds → test again → cache miss again (TTL expired).

---

## Part D — Force Redis Failover

- [ ] ElastiCache → `lab-redis` → Primary node → Actions → **Failover primary**.
- [ ] Confirm. Replica becomes the new primary. Takes ~20–30 seconds.
- [ ] During failover, your Lambda will get a brief connection error (Redis unavailable).
- [ ] After failover, test Lambda again → cache is empty (replica didn't have the data you just cached on old primary — slight inconsistency window). DynamoDB returns fresh data.
- [ ] Test again → cache populated again on new primary.

**Exam lesson:** ElastiCache Redis failover is automatic with Multi-AZ enabled. The primary endpoint DNS updates to point to the new primary. Brief downtime (~20–30s) expected.

---

## Part E — Understand Cluster Mode Off vs On

**Current setup (Cluster Mode Disabled):**
```
Primary ──async──► Replica
         ↑ all writes here; all data on one node
```
- Scale reads: yes (replicas).
- Scale writes: no (only one primary, can't shard).
- Max data: limited to one node's memory.

**Cluster Mode Enabled (not creating — just observe the config):**
- [ ] Create a new ElastiCache Redis with Cluster mode **Enabled**.
- [ ] Notice: you now set number of shards (e.g. 3). Each shard has a primary + replicas.
- [ ] Data is split across shards by hash slot (16,384 slots divided by shards).
- [ ] Use case: write-heavy workloads or datasets larger than one node.

---

## Key Things to Observe

- Cache-aside pattern: app checks cache first, falls back to DB on miss, populates cache.
- Cache hit is 10–50x faster than a DynamoDB read (1–3ms vs 30–50ms).
- TTL (`setex`) is critical — without it, stale data accumulates forever.
- Failover promotes replica automatically; brief connection interruption expected.
- Cluster mode off = scale reads; cluster mode on = scale reads + writes.

---

## Clean Up

- Delete Lambda function.
- Delete DynamoDB table.
- Delete ElastiCache cluster (takes ~5 min).
- Release Elastic IPs if any.
