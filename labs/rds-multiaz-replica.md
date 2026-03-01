# Lab: RDS — Multi-AZ, Read Replica & Failover

*Phase 3 · Week 5 · Sat Apr 4 · Concept: [concepts/databases-relational.md](../concepts/databases-relational.md)*

---

## Goal

Create a Multi-AZ RDS instance, observe automated failover, add a Read Replica, and see that the standby is NOT readable.

---

## Architecture

```
App (EC2 or CloudShell)
        │
        ▼
  RDS Primary (AZ-a)  ──sync──► RDS Standby (AZ-b)   [Multi-AZ]
        │
        └──async──► Read Replica (AZ-a or AZ-b)       [Read scaling]
```

---

## Tasks

### Part A — Create Multi-AZ RDS

- [ ] Go to RDS → Create database.
- [ ] Engine: MySQL 8.0. Template: **Free tier** (note: Multi-AZ not available on free tier — use Dev/Test for this lab, or note the setting and use a small instance).
- [ ] DB instance identifier: `lab-db`. Master username: `admin`. Master password: choose one.
- [ ] Instance: `db.t3.micro`.
- [ ] **Multi-AZ deployment:** Yes (creates a standby). ← Key setting.
- [ ] VPC: Default VPC. Public access: Yes (for lab access via CloudShell). Security group: allow MySQL (3306) from your IP.
- [ ] Disable automated backups for speed (set to 0 days — note: this also disables Read Replicas; re-enable after testing Multi-AZ).
- [ ] Create.

**While it creates, answer:**
- Where is the standby? (Different AZ — check the Instance details after creation.)
- Can you connect to the standby directly? (No — you only get one endpoint.)

---

### Part B — Verify Multi-AZ Behaviour

**1. Connect to the primary**
- [ ] Open CloudShell. Install MySQL client:
  ```bash
  sudo yum install -y mysql
  ```
- [ ] Connect using the endpoint from RDS console:
  ```bash
  mysql -h <endpoint> -u admin -p
  ```
- [ ] Run: `SELECT @@hostname;` — note the hostname (it's the primary instance).

**2. Create test data**
- [ ] In MySQL:
  ```sql
  CREATE DATABASE testdb;
  USE testdb;
  CREATE TABLE items (id INT, name VARCHAR(50));
  INSERT INTO items VALUES (1, 'apple'), (2, 'banana');
  SELECT * FROM items;
  ```

**3. Force a failover**
- [ ] In RDS console → select your DB → Actions → **Reboot** → check "Reboot with failover" → Confirm.
- [ ] The DB will be unavailable for ~60–120 seconds (DNS update).
- [ ] Reconnect after reboot:
  ```bash
  mysql -h <same endpoint> -u admin -p
  ```
- [ ] Run: `SELECT @@hostname;` — hostname should be DIFFERENT now (standby promoted to primary).
- [ ] Run: `SELECT * FROM items;` — data is intact (synchronous replication kept it in sync).

**Key observation:** The endpoint did NOT change. DNS updated automatically. Your app reconnects to the new primary via the same endpoint.

---

### Part C — Read Replica

*(First: enable automated backups — set retention to 1 day — required for Read Replicas.)*

- [ ] Select your DB → Actions → **Create read replica**.
- [ ] Destination region: same region. Instance: `db.t3.micro`. Identifier: `lab-db-replica`.
- [ ] Note: No Multi-AZ for the replica (it's for reads only).
- [ ] Create and wait (~10 min).

**Test the Read Replica**
- [ ] Find the Read Replica endpoint in RDS console.
- [ ] Connect to the Read Replica:
  ```bash
  mysql -h <replica-endpoint> -u admin -p
  ```
- [ ] Run: `SELECT * FROM items;` → data visible (replicated from primary).
- [ ] Try: `INSERT INTO items VALUES (3, 'cherry');` → should fail with "The MySQL server is running with the --read-only option."

**Key observation:** Read Replica = readable, but read-only. Multi-AZ standby = not accessible at all.

---

### Part D — Observe Encryption Requirement (No Lab, Note Only)

- [ ] In RDS, try to modify an unencrypted DB to enable encryption → you CAN'T.
- [ ] The process: snapshot the DB → Actions → Copy snapshot (check "Enable encryption") → restore from encrypted snapshot.
- [ ] This creates a new encrypted DB. The old unencrypted one is separate.

---

## Key Things to Observe

- Multi-AZ endpoint doesn't change after failover — DNS updates automatically.
- Standby is NOT accessible (not in RDS endpoint list, not queryable).
- Read Replica has its own separate endpoint.
- Read Replica is read-only — all writes rejected.
- Failover = ~60–120s downtime for Multi-AZ. Read Replica promotion = manual, longer.
- Encryption must be set at creation time.

---

## Clean Up

- Delete the Read Replica first (must delete before primary).
- Delete the primary DB instance (uncheck "Create final snapshot" for lab cleanup).
- Delete any manually created snapshots.
