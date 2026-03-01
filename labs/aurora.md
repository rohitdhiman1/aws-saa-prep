# Lab: Aurora — Cluster Architecture, Failover & Backtrack

*Phase 3 · Relational Databases · Concept: [concepts/databases-relational.md](../concepts/databases-relational.md)*

---

## Goal

Experience what makes Aurora different from standard RDS: sub-30-second failover, the writer/reader endpoint split, backtrack (rewind without a restore), and Serverless v2 scaling. These are all exam-tested distinctions.

---

## Part A — Create an Aurora MySQL Cluster

- [ ] RDS → Create database → **Amazon Aurora** → MySQL compatibility.
- [ ] Template: Dev/Test (to use small instances).
- [ ] DB cluster identifier: `lab-aurora`. Admin user/password: set one.
- [ ] Instance: `db.t3.medium` (smallest Aurora supports). **Multi-AZ: Yes** — Aurora automatically creates a writer + one reader replica in different AZs.
- [ ] VPC: default or your lab VPC. Public access: Yes (for lab). SG: allow MySQL 3306 from your IP.
- [ ] Additional configuration → Enable backtrack: **24 hours** (only available for MySQL-compat).
- [ ] Create. (~10 min)

---

## Part B — Understand the Endpoints

Once created, open the cluster view (not the instance view):

- [ ] Note the **Cluster (writer) endpoint** — e.g. `lab-aurora.cluster-xxx.us-east-1.rds.amazonaws.com`
- [ ] Note the **Reader endpoint** — e.g. `lab-aurora.cluster-ro-xxx.us-east-1.rds.amazonaws.com`
- [ ] Note the two **Instance endpoints** (one for writer, one for reader). These are for troubleshooting only — your app should never use these.

**Connect via CloudShell:**
```bash
mysql -h <cluster-writer-endpoint> -u admin -p
```
```sql
SELECT @@hostname;       -- shows the writer instance name
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE items (id INT, val VARCHAR(50));
INSERT INTO items VALUES (1,'aurora-test');
SELECT * FROM items;
```

**Connect to reader endpoint:**
```bash
mysql -h <reader-endpoint> -u admin -p
```
```sql
SELECT @@hostname;       -- different hostname (replica)
SELECT * FROM testdb.items;  -- data is there (replicated)
INSERT INTO testdb.items VALUES (2,'write-test');  -- ERROR: read-only
```

---

## Part C — Force Failover and Observe Speed

- [ ] Note the writer instance name from `SELECT @@hostname` above.
- [ ] In the console: RDS → Clusters → `lab-aurora` → Actions → **Failover**. Confirm.
- [ ] Start a timer. Keep running `SELECT @@hostname` every few seconds from the writer endpoint:
  ```bash
  while true; do
    mysql -h <writer-endpoint> -u admin -p<password> -e "SELECT @@hostname;" 2>&1 | tail -1
    sleep 2
  done
  ```
- [ ] You'll see a brief connection error during failover, then the **hostname changes** — the replica was promoted to writer.
- [ ] **Typical time: 20–30 seconds.** Compare this to RDS Multi-AZ (~60–120s).
- [ ] The writer endpoint DNS updated automatically — your app uses the same endpoint throughout.

**Why is Aurora failover faster?**
Aurora's storage is separate from compute — the new writer doesn't need to replay transaction logs. It just resumes serving the shared storage volume.

---

## Part D — Backtrack (Rewind Without Restore)

This is unique to Aurora MySQL and heavily tested.

**1. Set a recovery point**
```sql
-- Note the current time (you'll backtrack to just before the next step)
SELECT NOW();   -- e.g. 2026-04-04 14:30:00
```

**2. Cause "accidental" damage**
```sql
DELETE FROM testdb.items;      -- oops, deleted everything
SELECT * FROM testdb.items;    -- empty table
```

**3. Backtrack the cluster**
- [ ] RDS → Clusters → `lab-aurora` → Actions → **Backtrack**.
- [ ] Enter a time 30 seconds BEFORE the DELETE (e.g. `2026-04-04 14:29:30`).
- [ ] Confirm. The cluster goes into `backtracking` state briefly.
- [ ] Reconnect:
  ```sql
  SELECT * FROM testdb.items;    -- data is back!
  ```

**Key exam point:** Backtrack rewinds the entire cluster in place — no new cluster created, no restore from snapshot. It's fast (minutes) but causes a brief outage. It only works for MySQL-compatible Aurora and only within the configured backtrack window.

---

## Part E — Aurora Serverless v2 (Observe Scaling)

- [ ] Modify the writer instance: change instance class to **Serverless v2**.
- [ ] Set minimum ACU: 0.5, maximum ACU: 4.
- [ ] Apply immediately.

**Generate load and watch scaling:**
```bash
# From CloudShell, run a loop to generate queries
for i in {1..500}; do
  mysql -h <writer-endpoint> -u admin -p<password> -e "SELECT SLEEP(0.1);" &
done
```
- [ ] In RDS → Monitoring → `ServerlessDatabaseCapacity` metric — watch ACU climb from 0.5 toward 4.
- [ ] Stop the load → ACU scales back down within minutes.

---

## Key Things to Observe

- Aurora has two endpoints per cluster: writer and reader. Your app should use these, never instance endpoints.
- Failover promotes a reader to writer. The writer endpoint's DNS updates automatically (~20–30s).
- After failover, the old writer becomes a reader replica.
- Backtrack is not a backup — it can only rewind within the configured window and causes brief downtime.
- Aurora Serverless v2 scales continuously (not in discrete steps like v1). Even replicas scale.

---

## Clean Up

- RDS → Delete cluster (uncheck "Create final snapshot"). Confirm deletion.
