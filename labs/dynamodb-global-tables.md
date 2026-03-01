# Lab: DynamoDB Global Tables — Multi-Region Replication & Capacity Modes

*Phase 3 · NoSQL · Concept: [concepts/databases-nosql.md](../concepts/databases-nosql.md)*

---

## Goal

Create a DynamoDB Global Table that replicates across two regions. Write in one region, read in the other, and observe replication latency. Compare on-demand vs provisioned capacity modes and understand when to use each.

---

## Architecture

```
us-east-1 (primary)                us-west-2 (replica)
┌─────────────────┐                ┌─────────────────┐
│  DynamoDB Table  │ ◄──────────►  │  DynamoDB Table  │
│  global-lab      │   bi-dir      │  global-lab      │
│  (read/write)    │   replication  │  (read/write)    │
└─────────────────┘   ~1 second    └─────────────────┘
```

Both regions are read/write (multi-active). Replication is asynchronous, typically under 1 second.

---

## Part A — Create the Base Table

- [ ] DynamoDB (in **us-east-1**) → Create table.
- [ ] Table name: `global-lab`.
- [ ] Partition key: `PK` (String). Sort key: `SK` (String).
- [ ] Table settings: **Customize settings**.
- [ ] Capacity mode: **On-demand** (required for Global Tables creation — you can switch later).
- [ ] Encryption: AWS owned key (default).
- [ ] Create table. Wait for status: ACTIVE.

**Why on-demand is required:** Global Tables require on-demand mode (or provisioned with auto-scaling) at creation time. This ensures replicas can handle unpredictable write traffic during initial sync.

---

## Part B — Add a Global Table Replica

- [ ] DynamoDB → Tables → `global-lab` → Global Tables tab.
- [ ] Create replica → Region: **us-west-2**.
- [ ] Create. Wait for replica status to become ACTIVE (~2–5 minutes).
- [ ] While waiting, observe: DynamoDB automatically creates a table with the same name, schema, and indexes in us-west-2.

**Verify the replica exists:**
- [ ] Switch to the **us-west-2** region in the console.
- [ ] DynamoDB → Tables → `global-lab` should appear with a "Global table" badge.

---

## Part C — Test Multi-Region Replication

**1. Write in us-east-1**
- [ ] Switch back to **us-east-1**.
- [ ] DynamoDB → `global-lab` → Explore table items → Create item:
  ```json
  {
    "PK": "USER#alice",
    "SK": "PROFILE",
    "name": "Alice",
    "region": "us-east-1"
  }
  ```

**2. Read in us-west-2**
- [ ] Switch to **us-west-2**.
- [ ] DynamoDB → `global-lab` → Explore table items.
- [ ] The item should appear within 1–2 seconds (async replication).

**3. Write in us-west-2**
- [ ] In us-west-2, create another item:
  ```json
  {
    "PK": "USER#bob",
    "SK": "PROFILE",
    "name": "Bob",
    "region": "us-west-2"
  }
  ```

**4. Read in us-east-1**
- [ ] Switch back to us-east-1 → both items visible.
- [ ] This confirms **bi-directional** replication — both regions are read/write.

**5. Conflict resolution test**
- [ ] In us-east-1, update Alice's item: set `name` to `Alice-East`.
- [ ] Immediately in us-west-2, update the same item: set `name` to `Alice-West`.
- [ ] Wait a few seconds. Check both regions.
- [ ] Result: **last writer wins** — DynamoDB uses the timestamp of the write to resolve conflicts. The most recent update is kept in both regions.

---

## Part D — Observe Replication Metrics

- [ ] CloudWatch → Metrics → DynamoDB → `ReplicationLatency`.
- [ ] Select table `global-lab`, receiving region `us-west-2`.
- [ ] Typical value: 500ms–1.5s. This is the time from write in source to available in replica.
- [ ] Note: this is **eventually consistent** — not strongly consistent across regions. A read immediately after a write in a different region may return stale data.

---

## Part E — On-Demand vs Provisioned Capacity

**1. Current mode: On-Demand**
- [ ] DynamoDB → `global-lab` → Additional settings → Read/write capacity.
- [ ] Mode: On-demand. You pay per request:
  - Write: $1.25 per million WRUs
  - Read: $0.25 per million RRUs
- [ ] No capacity planning needed. Scales instantly. Best for unpredictable traffic.

**2. Switch to Provisioned (observation only — don't switch on a Global Table mid-lab)**
- [ ] Note the settings you'd configure if provisioned:
  - Read Capacity Units (RCUs): 1 RCU = 1 strongly consistent read/sec (4 KB item).
  - Write Capacity Units (WCUs): 1 WCU = 1 write/sec (1 KB item).
  - Auto-scaling: min 5, max 100, target utilization 70%.
- [ ] Provisioned is cheaper for **steady, predictable** workloads.

**Decision matrix:**

| Workload | Best Mode |
|---|---|
| Unpredictable / spiky traffic | On-demand |
| New table, unknown traffic pattern | On-demand (switch later) |
| Steady traffic, well-understood | Provisioned with auto-scaling |
| Batch writes (large imports) | On-demand (or provisioned with high WCU burst) |

**Exam trap:** You can only switch between on-demand and provisioned **once every 24 hours**. If the exam says "traffic is unpredictable and spiky," the answer is on-demand.

---

## Part F — Global Tables Requirements and Limits

Review these constraints — they appear in exam questions:

| Requirement | Detail |
|---|---|
| DynamoDB Streams | Automatically enabled (uses NEW_AND_OLD_IMAGES) |
| Capacity mode | Must be on-demand or provisioned with auto-scaling |
| Table must be empty or new | Can add replica to existing table (since v2019) |
| Encryption | Must use KMS (AWS owned, AWS managed, or CMK) |
| GSIs | Must be identical across all replicas |
| TTL | Deletes replicate across regions (item deleted everywhere) |
| Max replicas | No hard limit (practical: most use 2–3 regions) |

**Exam patterns:**
| Scenario | Solution |
|---|---|
| Multi-region low-latency reads | DynamoDB Global Tables |
| Multi-region active-active for NoSQL | DynamoDB Global Tables |
| Multi-region with strong consistency | NOT Global Tables (eventually consistent). Consider Aurora Global DB for relational. |
| RPO near zero for DynamoDB | Global Tables (<1s replication) |
| Global user base, each region writes locally | Global Tables (multi-active) |

---

## Key Things to Observe

- Global Tables provide multi-region, multi-active replication with sub-second latency.
- Conflict resolution is **last writer wins** (based on timestamp) — no manual conflict handling.
- Replication is eventually consistent — reads in the replica region may be slightly behind.
- DynamoDB Streams is automatically enabled for Global Tables and cannot be disabled.
- On-demand mode is best for unpredictable traffic. Provisioned with auto-scaling is cheaper for steady workloads.
- TTL deletes replicate — if you delete in one region via TTL, it's deleted everywhere.

---

## Clean Up

- [ ] DynamoDB (us-east-1) → `global-lab` → Global Tables → Delete replica (us-west-2). Wait for deletion.
- [ ] Then delete the table in us-east-1.
- [ ] Verify in us-west-2 that the table is gone.
