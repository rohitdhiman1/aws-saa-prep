# Diagrams: Storage

*Draw these on paper or in draw.io as part of Week 4 study.*

---

## Diagram 1: S3 Storage Class Lifecycle

```
Object Created
      │
      ▼
  Standard ──(30 days)──► Standard-IA ──(60 days)──► Glacier Instant ──(90 days)──► Glacier Deep Archive
      │
      └──(Intelligent-Tiering: AWS auto-moves based on access pattern)
```

**Cost comparison (rough):**
```
Standard > Standard-IA > Glacier Instant > Glacier Flexible > Glacier Deep Archive
(storage cost direction) ─────────────────────────────────────────────────────────►
(retrieval cost direction) ◄─────────────────────────────────────────────────────
```

---

## Diagram 2: EBS vs EFS vs FSx — Decision Tree

```
Need shared storage across multiple instances?
  ├── YES → Linux workloads? ──YES──► EFS (NFS, multi-AZ, auto-scale)
  │          └── Windows / SMB? ──► FSx for Windows File Server
  │          └── HPC / Lustre? ──► FSx for Lustre
  └── NO (single instance) ──► EBS
        └── High IOPS DB? ──► io2 Block Express
        └── General purpose? ──► gp3 (default choice)
        └── Sequential big data? ──► st1
        └── Cold/archive? ──► sc1
```

---

## Diagram 3: S3 Encryption Options

```
Data upload request
        │
        ▼
┌────────────────────────────────────────────────────┐
│  SSE-S3:   AWS manages key (AES-256). Automatic.  │
│  SSE-KMS:  KMS CMK. Audit trail in CloudTrail.    │
│  SSE-C:    Customer sends key per request.         │
│  Client:   Encrypt before upload. AWS sees cipher. │
└────────────────────────────────────────────────────┘
```

**Cross-Region Replication with SSE-KMS:**
```
Source bucket (us-east-1)          Destination bucket (eu-west-1)
  SSE-KMS (key-A)       ──CRR──►   SSE-KMS (key-B, eu-west-1 CMK)
                                    (separate key; objects re-encrypted)
```

---

## Diagram 4: Storage Gateway Types

```
On-Premises                    AWS

NFS/SMB clients                S3 (Standard, IA, Glacier)
    │                               ▲
    ▼                               │
S3 File Gateway ────────────────────┘ (local cache for hot files)

SMB clients                    FSx for Windows
    │                               ▲
    ▼                               │
FSx File Gateway ───────────────────┘

iSCSI clients                  S3 (EBS snapshots)
    │                               ▲
    ▼                               │
Volume Gateway ─────────────────────┘ (Stored: primary on-prem | Cached: primary S3)

VTL clients                    S3 Glacier
    │                               ▲
    ▼                               │
Tape Gateway ───────────────────────┘ (virtual tape library)
```
