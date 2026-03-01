# Storage — S3, EBS, EFS, FSx, Storage Gateway

*Phase 2 · Week 4 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

AWS storage covers object (S3), block (EBS), file (EFS, FSx), and hybrid (Storage Gateway) storage.
Choosing the right type depends on access pattern, durability needs, and cost.

---

## How It Works

### S3 — Simple Storage Service

- **Object storage** — flat namespace with key-value pairs. No true folders (just key prefixes).
- **Unlimited capacity**; max object size 5 TB (multipart upload required for >5 GB).
- **Global service** — bucket names must be globally unique; data stays in chosen region.
- **Durability:** 11 9s (99.999999999%). **Availability** varies by storage class.

#### Storage Classes

| Class | Use case | Min duration | Retrieval |
|---|---|---|---|
| **Standard** | Frequent access | None | Instant |
| **Standard-IA** | Infrequent access, millisecond retrieval | 30 days | Instant |
| **One Zone-IA** | Infrequent, single AZ (re-creatable data) | 30 days | Instant |
| **Glacier Instant** | Archive, millisecond retrieval | 90 days | Instant |
| **Glacier Flexible** | Archive, occasional access | 90 days | Minutes to hours |
| **Glacier Deep Archive** | Long-term archive | 180 days | Up to 12 hours |
| **Intelligent-Tiering** | Unknown/changing access pattern | None | Instant (frequent/infrequent tier) |

#### Versioning, Replication & Events

- **Versioning** — once enabled on a bucket, cannot be fully disabled (only suspended). Keeps all versions; DELETE creates a delete marker.
- **CRR (Cross-Region Replication)** — replicate objects to a bucket in another region. Requires versioning on source AND destination. Does NOT replicate retroactively.
- **SRR (Same-Region Replication)** — same region; useful for log aggregation, test→prod sync.
- **Replication Time Control (RTC)** — 99.99% of objects replicated within 15 minutes; SLA-backed.
- **Event Notifications** — on `s3:ObjectCreated`, `s3:ObjectRemoved`, etc. → SNS, SQS, Lambda, or EventBridge.

#### Encryption

| Method | Key managed by | Notes |
|---|---|---|
| **SSE-S3** | AWS (AES-256) | Default since Jan 2023; no extra cost |
| **SSE-KMS** | AWS KMS (CMK) | Audit via CloudTrail; KMS API calls add cost |
| **DSSE-KMS** | AWS KMS | Dual-layer encryption |
| **SSE-C** | Customer (you send key per request) | HTTPS only; AWS does not store key |
| **Client-side** | Customer | Encrypted before upload |

#### Access Control

- **Bucket policy** — resource-based JSON; can allow cross-account or public access.
- **ACLs** — legacy; avoid. Bucket owner enforced setting disables ACLs.
- **Block Public Access** — 4 settings; should be enabled on all non-public buckets.
- **Pre-signed URLs** — time-limited URL to allow temporary GET or PUT without credentials. Max 7 days (12 hours with STS token).
- **OAC (Origin Access Control)** — restrict CloudFront-served S3 to only CloudFront; replaces OAI.

#### Advanced S3 Features

- **Transfer Acceleration** — uploads routed through CloudFront edge → S3 over AWS backbone. Extra cost.
- **Multipart Upload** — required for >5 GB; recommended for >100 MB. Parts 5 MB–5 GB; up to 10,000 parts.
- **S3 Select** — filter object contents server-side (CSV, JSON, Parquet) before downloading.
- **Object Lock** — WORM (Write Once Read Many). **Retention mode:** Compliance (cannot delete, even root) or Governance (can delete with special permission). **Legal hold:** indefinite, no retention period.
- **Static website hosting** — serve HTML/CSS/JS from S3. Endpoint format: `bucket.s3-website-region.amazonaws.com`. Does NOT support HTTPS natively — use CloudFront.
- **Requester Pays** — the requester (not bucket owner) pays for data transfer and requests.

---

### EBS — Elastic Block Store

Block storage attached to **one EC2 instance** in the same AZ (except `io2 Block Express` multi-attach).
Persists independently of instance lifecycle.

#### Volume Types

| Type | Max IOPS | Max Throughput | Use case |
|---|---|---|---|
| `gp3` | 16,000 | 1,000 MB/s | General purpose (default) |
| `gp2` | 16,000 | 250 MB/s | General purpose (legacy) |
| `io2 Block Express` | 256,000 | 4,000 MB/s | Critical DBs (Oracle, SQL Server) |
| `io1` | 64,000 | 1,000 MB/s | High-performance DBs |
| `st1` | 500 | 500 MB/s | Big data, log processing (sequential) |
| `sc1` | 250 | 250 MB/s | Cold data (cheapest HDD) |

> **gp3 vs gp2:** gp3 lets you independently set IOPS and throughput; gp2 ties IOPS to size (3 IOPS/GB). gp3 is cheaper and preferred.

- **Snapshots** — incremental backups to S3. Can copy across regions (for DR/migration). Can create AMI from snapshot.
- **Encryption** — at-rest encryption using KMS. Snapshots of encrypted volumes are encrypted. Enable account-level default encryption to encrypt all new volumes automatically.
- **Multi-attach** — `io1`/`io2` only; same volume → up to 16 Nitro EC2 instances in same AZ. Requires cluster-aware file system.
- **EBS-optimised** — dedicated network bandwidth for EBS I/O; enabled by default on most modern instances.

---

### EFS — Elastic File System

- **Managed NFS** (NFSv4.1/4.2) — shared across **multiple EC2 instances** simultaneously, and across AZs.
- Linux only. POSIX-compliant.
- Scales automatically; no capacity planning.
- Mounted via DNS name; access via mount target in each AZ's subnet.

#### Performance Modes

| Mode | When to use |
|---|---|
| **General Purpose** (default) | Web serving, CMS, dev; lower latency |
| **Max I/O** | Big data, parallel workloads; higher throughput but higher latency |

#### Throughput Modes

| Mode | Description |
|---|---|
| **Bursting** | Throughput scales with storage size; good for most workloads |
| **Provisioned** | Set throughput independently of storage size |
| **Elastic** | Automatically scales with workload; pay per actual use (recommended) |

- **Storage tiers:** Standard, Infrequent Access (EFS-IA), Archive. **Lifecycle policy** moves files automatically.
- **EFS Access Points** — application-specific entry points with enforced UID/GID and root directory.
- **Mount targets** — one per AZ; attach to a security group. EC2 must be in same VPC (or VPC peering).

---

### FSx — Managed Third-Party File Systems

| Variant | Protocol | Use case |
|---|---|---|
| **FSx for Windows** | SMB | Windows apps, Active Directory integration, NTFS |
| **FSx for Lustre** | Lustre | HPC, ML training, high-throughput (data from S3) |
| **FSx for NetApp ONTAP** | NFS, SMB, iSCSI | Multi-protocol, lift-and-shift NetApp workloads |
| **FSx for OpenZFS** | NFS | ZFS features, low-latency Linux workloads |

**FSx for Lustre + S3 integration** — link to S3 bucket; data lazy-loaded on first access; write back to S3 on completion.

---

### Storage Gateway

Hybrid storage bridge between on-prem and AWS.

| Gateway type | Protocol | Backed by | Use case |
|---|---|---|---|
| **S3 File Gateway** | NFS / SMB | S3 (Standard, IA, etc.) | On-prem file shares backed by S3; local cache for recent files |
| **FSx File Gateway** | SMB | FSx for Windows | On-prem Windows file shares backed by FSx |
| **Volume Gateway (Stored)** | iSCSI | S3 + EBS snapshots | Primary data on-prem; async backup to S3 |
| **Volume Gateway (Cached)** | iSCSI | S3 | Primary data in S3; cache frequently accessed on-prem |
| **Tape Gateway** | VTL (iSCSI) | S3 Glacier | Replace physical tape libraries with virtual tapes |

---

### Migration Tools

| Tool | Use case |
|---|---|
| **AWS DataSync** | Online transfer: on-prem NFS/SMB/HDFS/S3 → AWS (S3, EFS, FSx). Scheduled, incremental, encrypted. |
| **Snowball Edge Storage Optimised** | 80 TB; offline transfer when bandwidth limited |
| **Snowball Edge Compute Optimised** | 42 TB usable + EC2/Lambda at edge; ruggedised |
| **Snowmobile** | Exabyte-scale; physical truck; for >10 PB |
| **S3 Transfer Acceleration** | Online; faster uploads over long distances via CloudFront edge |
| **Application Discovery Service** | Discover on-prem servers, dependencies before migration planning |

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| S3 max object size | 5 TB |
| S3 multipart required above | 5 GB (recommended above 100 MB) |
| S3 pre-signed URL max | 7 days (604,800 seconds) |
| S3 Intelligent-Tiering min size | 128 KB (smaller objects not monitored) |
| EBS max volume size (gp3/io2) | 16 TiB |
| EBS `io2` max IOPS | 64,000 (256,000 with Block Express) |
| EFS max throughput (Elastic mode) | Scales automatically |
| FSx for Lustre throughput | Up to 1 TB/s |
| Snowball Edge capacity | 80 TB (Storage), 42 TB (Compute) |

---

## When To Use

| Scenario | Solution |
|---|---|
| Static website / media hosting | S3 with CloudFront |
| Shared file storage for Linux EC2 | EFS |
| Shared file storage for Windows EC2 | FSx for Windows |
| HPC workload with S3 data | FSx for Lustre |
| Single EC2 high-IOPS database | io2 EBS |
| Archive, rarely accessed, cheapest | S3 Glacier Deep Archive |
| Hybrid NFS to S3 | S3 File Gateway |
| Migrate 100 TB on-prem data | Snowball Edge |
| Migrate 50 PB | Snowmobile |
| Continuous sync on-prem → AWS | DataSync |

---

## Exam Traps

- **S3 is not a file system** — it's object storage; no true directories, no append operations.
- **EBS is AZ-locked** — to move, take snapshot and restore in another AZ/region.
- **EFS is Linux only** — FSx for Windows for SMB/Windows workloads.
- **S3 Standard-IA charges per retrieval** — frequent access is more expensive than Standard.
- **CRR does not replicate existing objects** — only objects created after replication is enabled.
- **Delete with versioning** — DELETE creates a delete marker; it does NOT permanently delete the object.
- **Object Lock compliance mode** — even root cannot delete before retention period. Use Governance for flexibility.
- **SSE-KMS = extra API calls** — heavy S3 usage with SSE-KMS can hit KMS rate limits.
- **Pre-signed URL validity** — generated with IAM user creds: up to 7 days; with STS temp creds: limited to the credential expiry (up to 12 hours).
- **DataSync vs Storage Gateway** — DataSync for bulk migration/sync; Storage Gateway for ongoing hybrid access.
- **st1/sc1 not bootable** — only SSD types (gp2, gp3, io1, io2) can be boot volumes.

---

## Related Files

| File | Purpose |
|---|---|
| `diagrams/storage.md` | Visual: S3 classes lifecycle, EBS vs EFS vs FSx decision tree |
| `cheatsheets/storage.md` | Quick-reference card |
| `mock-exams/phase2.md` | Compute + storage practice questions |
| `concepts/databases-relational.md` | Next concept file (Week 5) |
