# Diagrams: Databases

*Draw these on paper or in draw.io as part of Week 5–6 study.*

---

## Diagram 1: RDS Multi-AZ vs Read Replicas

```
MULTI-AZ (HA / Failover):

Application ──► Primary DB (AZ-a) ──SYNC replication──► Standby (AZ-b)
                    │                                         │
                Writes/Reads                           NOT READABLE
                                                      (automatic failover if primary fails)

READ REPLICAS (Scale Reads):

Application ──► Primary DB ──ASYNC replication──► Read Replica 1 (same or cross-region)
                               └──────────────► Read Replica 2
                                                (Readable endpoints; can be promoted)
```

---

## Diagram 2: Aurora Cluster Architecture

```
                    Writer Endpoint (always primary)
                             │
              ┌──────────────┘
              ▼
         Primary Instance (Writer)
              │
   ┌──────────┼──────────┐
   ▼          ▼          ▼
Replica-1  Replica-2  Replica-3   ◄── Reader Endpoint (load-balanced)
  (AZ-a)    (AZ-b)    (AZ-c)

Shared Aurora Storage Volume (6 copies across 3 AZs, auto-grows to 128 TiB)
```

**Failover:** Aurora promotes a replica in <30 seconds automatically.

---

## Diagram 3: Aurora Global Database

```
Primary Region (us-east-1)           Secondary Region (eu-west-1)
┌──────────────────────────┐         ┌──────────────────────────┐
│  Aurora Cluster          │         │  Aurora Cluster          │
│  (read + write)          │──<1s──► │  (read only)             │
│                          │         │  (promote for DR)        │
└──────────────────────────┘         └──────────────────────────┘
```

---

## Diagram 4: DynamoDB Key Design

```
TABLE: Orders

Partition Key (PK)    Sort Key (SK)       Attributes
─────────────────────────────────────────────────────
customer#alice        ORDER#2024-01-01    {amount: 99}
customer#alice        ORDER#2024-01-15    {amount: 45}
customer#bob          ORDER#2024-01-03    {amount: 200}

Query: PK = "customer#alice" → returns all Alice's orders (sorted by SK)
GetItem: PK + SK → returns single item

GSI: SK as new PK (scan by order date across all customers)
```

---

## Diagram 5: ElastiCache Redis Topology

```
NON-CLUSTER MODE (single shard):
Primary ──ASYNC──► Replica-1
                ► Replica-2   ◄── Reader endpoint
(Multi-AZ failover: auto-promote replica to primary)

CLUSTER MODE (multiple shards):
Shard-0 (slots 0–5461):     Primary → Replica
Shard-1 (slots 5462–10922): Primary → Replica
Shard-2 (slots 10923–16383):Primary → Replica
(horizontal write scaling; cluster endpoint routes by hash slot)
```
