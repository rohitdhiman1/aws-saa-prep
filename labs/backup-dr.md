# Lab: Backup & Disaster Recovery — AWS Backup, Snapshots & DR Patterns

*Phase 6 · Review · Concept: [cheatsheets/cost.md](../cheatsheets/cost.md) · [diagrams/databases.md](../diagrams/databases.md)*

---

## Goal

Set up AWS Backup for RDS and EBS, perform a cross-region backup copy, practice restoring from a backup, and walk through the four DR architecture patterns with RTO/RPO trade-offs.

---

## Architecture

```
AWS Backup
    ├── Backup Plan: "lab-daily-plan"
    │       ├── Rule: daily at 03:00 UTC, retain 7 days
    │       └── Rule: weekly, copy to us-west-2, retain 30 days
    │
    └── Resource Assignment
            ├── RDS instance (tagged: backup=true)
            └── EBS volume (tagged: backup=true)
```

---

## Part A — Create Resources to Back Up

**1. Create an RDS instance**
- [ ] RDS → Create database → MySQL → Free tier.
- [ ] DB instance identifier: `backup-lab-db`.
- [ ] Master username: `admin`. Password: any.
- [ ] Storage: 20 GB gp3. Disable Multi-AZ (cost savings for the lab).
- [ ] Additional configuration → Backup retention: **1 day** (RDS default is 7).
- [ ] Tags: Key = `backup`, Value = `true`.
- [ ] Create.

**2. Create an EBS volume**
- [ ] EC2 → Volumes → Create volume.
- [ ] Type: gp3. Size: 1 GB. AZ: same as your default.
- [ ] Tags: Key = `backup`, Value = `true`.
- [ ] Create.

---

## Part B — Set Up AWS Backup

**1. Create a backup vault**
- [ ] AWS Backup → Backup vaults → Create backup vault.
- [ ] Name: `lab-vault`. Encryption: AWS managed key (default).
- [ ] Create.

**2. Create a backup plan**
- [ ] AWS Backup → Backup plans → Create backup plan → Build a new plan.
- [ ] Plan name: `lab-daily-plan`.

**Rule 1 — Daily backup:**
- [ ] Rule name: `daily-backup`.
- [ ] Frequency: Daily. Backup window: 03:00 UTC, 1 hour.
- [ ] Lifecycle: transition to cold storage: Never. Expire after: **7 days**.
- [ ] Backup vault: `lab-vault`.

**Rule 2 — Weekly cross-region copy:**
- [ ] Rule name: `weekly-cross-region`.
- [ ] Frequency: Weekly.
- [ ] Lifecycle: expire after **30 days**.
- [ ] Copy to another Region: **us-west-2** (or any secondary region).
- [ ] Create plan.

**3. Assign resources**
- [ ] AWS Backup → Backup plans → `lab-daily-plan` → Assign resources.
- [ ] Assignment name: `tagged-resources`.
- [ ] IAM role: Default role (AWS Backup creates it).
- [ ] Resource selection: **By tags** → Key = `backup`, Value = `true`.
- [ ] Assign.

---

## Part C — Run an On-Demand Backup and Restore

**1. Trigger an on-demand backup of the RDS instance**
- [ ] AWS Backup → Protected resources → `backup-lab-db` → Create on-demand backup.
- [ ] Backup vault: `lab-vault`. IAM role: Default.
- [ ] Create backup. Wait for it to complete (~5–10 min for a small DB).

**2. Verify the recovery point**
- [ ] AWS Backup → Backup vaults → `lab-vault` → Recovery points.
- [ ] You should see a recovery point for `backup-lab-db`.
- [ ] Click it → note: Recovery point ARN, Creation time, Status: COMPLETED.

**3. Restore the RDS instance**
- [ ] Select the recovery point → Restore.
- [ ] New DB instance identifier: `backup-lab-db-restored`.
- [ ] Keep defaults. Restore.
- [ ] Wait for the new instance to become Available.
- [ ] Observe: the restored instance is a **new** RDS instance (not an in-place restore). It has a new endpoint.

**Exam trap:** RDS restores always create a **new instance**. You must update your application's connection string. The original instance is untouched.

**4. Trigger an on-demand EBS snapshot**
- [ ] AWS Backup → Protected resources → select your EBS volume → Create on-demand backup.
- [ ] Or: EC2 → Snapshots → Create snapshot → select your volume.
- [ ] Wait for completion.

**5. Restore EBS from snapshot**
- [ ] EC2 → Snapshots → select the snapshot → Create volume from snapshot.
- [ ] Note: you can change the AZ and volume type during restore.
- [ ] You can also copy the snapshot to another region (Actions → Copy snapshot → select target region).

---

## Part D — EBS Snapshot Lifecycle with DLM

- [ ] EC2 → Lifecycle Manager → Create lifecycle policy.
- [ ] Policy type: **EBS snapshot policy**.
- [ ] Target: Tags → Key = `backup`, Value = `true`.
- [ ] Schedule: every 24 hours, retain **5** snapshots.
- [ ] Cross-region copy: optionally copy to a second region.
- [ ] Create.

**DLM vs AWS Backup for EBS:**
| Feature | DLM | AWS Backup |
|---|---|---|
| Scope | EBS snapshots only | EBS, RDS, DynamoDB, EFS, S3, Aurora, FSx, etc. |
| Cost | Free | Free (pay only for storage) |
| Cross-service policy | No | Yes — one plan for all resources |
| Cross-account copy | No | Yes (with AWS Organizations) |

**Exam takeaway:** If the question mentions backing up multiple AWS services with a single policy, the answer is **AWS Backup**. If it's EBS-only automation, DLM also works.

---

## Part E — DR Patterns (RTO/RPO Decision Matrix)

No resources to build here — this is a conceptual exercise to cement the four patterns.

**Draw these from memory, then check against the table:**

| Pattern | RPO | RTO | Cost | How It Works |
|---|---|---|---|---|
| **Backup & Restore** | Hours | Hours | $ (cheapest) | Back up data to S3/snapshots. Restore when disaster hits. No running infra in DR region. |
| **Pilot Light** | Minutes | 10s of min | $$ | Core services (DB) running in DR region but minimal. Scale up on failure. |
| **Warm Standby** | Seconds | Minutes | $$$ | Scaled-down copy of full production running in DR. Scale up on failure. |
| **Multi-Site Active-Active** | Near zero | Near zero | $$$$ (most expensive) | Full production in both regions. Route 53 routes to both. Instant failover. |

**Exam scenario mapping:**
| Scenario | Pattern |
|---|---|
| "Minimize cost, RTO of 24 hours acceptable" | Backup & Restore |
| "RTO under 1 hour, budget-conscious" | Pilot Light |
| "RTO under 15 minutes, some cost acceptable" | Warm Standby |
| "Zero downtime, cost is not a concern" | Multi-Site Active-Active |
| "RPO of zero for database" | Aurora Global Database (cross-region replication <1s lag) |

**Services that support multi-region:**
- Aurora Global Database — 1-second replication lag, promotes in <1 min.
- DynamoDB Global Tables — multi-region, multi-active.
- S3 Cross-Region Replication — async, minutes lag.
- Route 53 — health checks + failover routing for DNS cutover.

---

## Key Things to Observe

- AWS Backup provides a single pane of glass for backup across RDS, EBS, DynamoDB, EFS, FSx, and more.
- Tag-based resource assignment means new resources with the right tag are automatically included.
- Cross-region copy in AWS Backup enables DR without custom scripts.
- RDS restore = new instance (new endpoint). EBS restore = new volume (attach to EC2 manually).
- DLM is free and EBS-specific. AWS Backup is the unified solution.
- DR pattern selection is driven by RTO/RPO requirements vs cost tolerance — this is a guaranteed exam topic.

---

## Clean Up

- AWS Backup → delete backup plan → delete recovery points in vault → delete vault.
- EC2 → delete Lifecycle Manager policy.
- EC2 → delete snapshots, delete restored volume, delete original volume.
- RDS → delete both `backup-lab-db` and `backup-lab-db-restored` (skip final snapshot).
