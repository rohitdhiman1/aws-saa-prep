# Cheatsheet: Storage

*Fill in as you complete Week 4. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| S3 max object size | 5 TB |
| S3 multipart required | >5 GB (recommended >100 MB) |
| S3 durability | 11 9s |
| S3 pre-signed URL max | 7 days |
| S3 Standard-IA min duration | 30 days |
| Glacier Deep Archive min | 180 days |
| EBS max gp3 volume | 16 TiB |
| EFS | Linux only, NFS, multi-AZ, auto-scales |
| FSx for Windows | SMB, AD integration |
| FSx for Lustre | HPC, S3 integration |

---

## Must-Know Distinctions

- **EBS** → single instance, AZ-locked | **EFS** → multi-instance, multi-AZ (Linux) | **FSx** → Windows or HPC
- **SSE-S3** → AWS manages key, free | **SSE-KMS** → audit trail, extra cost | **SSE-C** → you manage key
- **CRR** → cross-region replication | **SRR** → same-region replication
- **DataSync** → migration/sync | **Storage Gateway** → ongoing hybrid access
- **st1/sc1** → HDD, NOT bootable | **gp3/io2** → SSD, bootable

---

## Exam Traps (Quick List)

- DELETE with versioning = delete marker, not permanent deletion
- CRR doesn't replicate existing objects retroactively
- Standard-IA: retrieval fee; frequent access = more expensive than Standard
- Object Lock Compliance: even root cannot delete
- S3 static site: no HTTPS natively; use CloudFront

---

## Related Files

`concepts/storage.md` · `diagrams/storage.md` · `mock-exams/phase2.md`
