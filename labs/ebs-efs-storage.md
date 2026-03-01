# Lab: EBS Volume Types, EFS Shared Storage & FSx Decision Matrix

*Phase 2 · Storage · Concept: [concepts/storage.md](../concepts/storage.md)*

---

## Goal

Create and compare EBS volume types (gp3, io2, st1), set up an EFS file system shared across two EC2 instances, and understand when to use EBS vs EFS vs FSx. This lab fills the storage gap beyond S3.

---

## Architecture

```
                  ┌── EC2-A (us-east-1a)
EFS File System ──┤   mount: /mnt/shared
(multi-AZ)        └── EC2-B (us-east-1b)
                      mount: /mnt/shared
                      (same files visible on both)

EC2-A also has:
  /dev/xvdf → gp3 (general purpose)
  /dev/xvdg → io2 (high IOPS)
```

---

## Part A — EBS Volume Types (Create and Compare)

**1. Launch an EC2 instance**
- [ ] EC2 → Launch instance → Amazon Linux 2023 → `t3.micro`.
- [ ] Name: `storage-lab`. Key pair: select or create one. VPC: default. Public subnet.
- [ ] Security group: allow SSH (22) from your IP.
- [ ] Launch.

**2. Create a gp3 volume**
- [ ] EC2 → Volumes → Create volume.
- [ ] Type: **gp3**. Size: **10 GB**. AZ: same as your instance.
- [ ] Note the defaults: 3,000 IOPS, 125 MB/s throughput (included in the price).
- [ ] Create. Attach to `storage-lab` as `/dev/xvdf`.

**3. Create an io2 volume**
- [ ] EC2 → Volumes → Create volume.
- [ ] Type: **io2**. Size: **10 GB**. IOPS: **5,000** (io2 lets you provision up to 64,000 IOPS).
- [ ] AZ: same as your instance. Create. Attach as `/dev/xvdg`.

**4. Format and mount both volumes**
- [ ] SSH into the instance:
  ```bash
  # gp3
  sudo mkfs -t xfs /dev/xvdf
  sudo mkdir /mnt/gp3
  sudo mount /dev/xvdf /mnt/gp3

  # io2
  sudo mkfs -t xfs /dev/xvdg
  sudo mkdir /mnt/io2
  sudo mount /dev/xvdg /mnt/io2

  df -h  # both should appear
  ```

**5. Benchmark (simple write test)**
- [ ] ```bash
  # Write test on gp3
  sudo dd if=/dev/zero of=/mnt/gp3/testfile bs=1M count=500 oflag=direct
  # Note the throughput (MB/s)

  # Write test on io2
  sudo dd if=/dev/zero of=/mnt/io2/testfile bs=1M count=500 oflag=direct
  # Note the throughput — io2 should be faster with provisioned IOPS
  ```

**6. Understand the volume type decision matrix**

| Type | IOPS (max) | Throughput (max) | Use Case | Cost Model |
|---|---|---|---|---|
| **gp3** | 16,000 | 1,000 MB/s | Boot volumes, dev/test, most workloads | $0.08/GB + IOPS/throughput if above baseline |
| **gp2** | 16,000 (burst) | 250 MB/s | Legacy — gp3 is always better | $0.10/GB (IOPS tied to size) |
| **io2** | 64,000 | 1,000 MB/s | Databases, latency-sensitive apps | $0.125/GB + $0.065/IOPS |
| **io2 Block Express** | 256,000 | 4,000 MB/s | Largest databases (SAP, Oracle) | Same as io2 |
| **st1** | 500 | 500 MB/s | Big data, log processing, sequential reads | $0.045/GB |
| **sc1** | 250 | 250 MB/s | Cold storage, infrequent access | $0.015/GB |

**Exam traps:**
- gp3 gives 3,000 IOPS free (regardless of size). gp2 ties IOPS to volume size (3 IOPS/GB).
- **st1 and sc1 cannot be boot volumes** — only gp2/gp3/io1/io2 can.
- io2 supports **Multi-Attach** (attach one volume to multiple instances in the same AZ) — but only for Nitro instances with io2.
- EBS volumes are **AZ-locked**. To move to another AZ: snapshot → restore in new AZ.

---

## Part B — EBS Snapshots and Migration

**1. Create a snapshot**
- [ ] EC2 → Volumes → select gp3 volume → Actions → Create snapshot.
- [ ] Description: `gp3-lab-snapshot`. Create.

**2. Restore to a different AZ**
- [ ] EC2 → Snapshots → select your snapshot → Actions → Create volume.
- [ ] Change the AZ to a different one (e.g., `us-east-1b`).
- [ ] Optionally change the volume type (e.g., restore as io2 from a gp3 snapshot).
- [ ] Create.

**3. Copy snapshot to another region**
- [ ] Snapshots → select snapshot → Actions → Copy snapshot.
- [ ] Destination region: `us-west-2`. Copy.
- [ ] This is how you migrate EBS data across regions — snapshot + copy + restore.

**Exam pattern:** "Move an EC2 instance to another region" → Snapshot EBS → Copy snapshot to target region → Restore volume → Launch new EC2 with the volume.

---

## Part C — EFS (Elastic File System) — Shared Storage

**1. Create an EFS file system**
- [ ] EFS → Create file system.
- [ ] Name: `lab-efs`.
- [ ] VPC: your VPC.
- [ ] Performance mode: **General Purpose** (default, lowest latency).
- [ ] Throughput mode: **Elastic** (auto-scales, recommended).
- [ ] Encryption: enable encryption at rest (default KMS key).
- [ ] Create.

**2. Configure mount targets**
- [ ] EFS → `lab-efs` → Network tab.
- [ ] Verify mount targets exist in at least 2 AZs (created automatically with the default VPC).
- [ ] Note the security group — it must allow **NFS (port 2049)** from your EC2 security group.
- [ ] Edit the mount target security group → add inbound rule: NFS (2049) from the EC2 SG.

**3. Launch a second EC2 instance in a different AZ**
- [ ] Launch another `t3.micro` → Name: `storage-lab-b`.
- [ ] Place it in a **different AZ** (e.g., `us-east-1b`).
- [ ] Same security group (SSH allowed).

**4. Mount EFS on both instances**
- [ ] SSH into `storage-lab` (instance A):
  ```bash
  sudo yum install -y amazon-efs-utils
  sudo mkdir /mnt/shared
  sudo mount -t efs <file-system-id>:/ /mnt/shared
  ```
- [ ] SSH into `storage-lab-b` (instance B):
  ```bash
  sudo yum install -y amazon-efs-utils
  sudo mkdir /mnt/shared
  sudo mount -t efs <file-system-id>:/ /mnt/shared
  ```

**5. Test shared access**
- [ ] On instance A:
  ```bash
  echo "written from instance A" | sudo tee /mnt/shared/hello.txt
  ```
- [ ] On instance B:
  ```bash
  cat /mnt/shared/hello.txt
  # Output: "written from instance A"
  ```
- [ ] Both instances see the same file — this is the key difference from EBS (which is single-instance, single-AZ).

**6. EFS lifecycle policy**
- [ ] EFS → `lab-efs` → Lifecycle management.
- [ ] Transition to IA: **30 days** (moves infrequently accessed files to EFS-IA, ~85% cost savings).
- [ ] Transition out of IA: **On first access** (moves back to Standard when read).

---

## Part D — EBS vs EFS vs FSx Decision Matrix

| Feature | EBS | EFS | FSx for Windows | FSx for Lustre |
|---|---|---|---|---|
| Type | Block storage | NFS file system | SMB file system | High-perf parallel FS |
| Multi-AZ | No (AZ-locked) | Yes (multi-AZ) | Multi-AZ option | Single-AZ (or scratch) |
| Multi-instance | No (except io2 Multi-Attach, same AZ) | Yes (thousands of instances) | Yes (Windows instances) | Yes (HPC workloads) |
| Protocol | Attached block device | NFS v4 | SMB / NTFS | POSIX-compliant |
| OS support | Linux + Windows | **Linux only** | **Windows only** | Linux |
| Use case | Boot vol, databases, single-instance storage | Shared content, CMS, web serving, containers | Windows file shares, AD integration, SQL Server | ML training, HPC, video processing |
| Pricing | Per GB provisioned | Per GB used (pay for what you store) | Per GB provisioned | Per GB provisioned |

**Exam decision shortcuts:**
| Scenario | Answer |
|---|---|
| "Shared file system across multiple EC2 Linux instances" | EFS |
| "Shared file system for Windows instances with Active Directory" | FSx for Windows File Server |
| "High-performance computing, ML training data" | FSx for Lustre |
| "Database storage, single instance, high IOPS" | EBS io2 |
| "Boot volume for EC2" | EBS gp3 |
| "Container shared storage (ECS/EKS)" | EFS |
| "Migrate on-premises NFS to AWS" | EFS (or FSx for Lustre if HPC) |
| "Migrate on-premises Windows file server to AWS" | FSx for Windows |
| "Log processing, sequential large reads" | EBS st1 |

---

## Key Things to Observe

- EBS is single-AZ, single-instance (except io2 Multi-Attach). EFS is multi-AZ, multi-instance.
- gp3 decouples IOPS from volume size (3,000 free). gp2 ties them together (3 IOPS/GB).
- st1/sc1 are throughput-optimized and cold HDD — they CANNOT be boot volumes.
- EFS is Linux-only (NFS). FSx for Windows is Windows-only (SMB).
- EFS pricing is per-GB-used (no provisioning). EBS pricing is per-GB-provisioned (you pay even if the volume is empty).
- EFS-IA lifecycle policy saves ~85% on infrequently accessed files — similar concept to S3-IA.
- EBS snapshots are incremental and stored in S3 (managed by AWS). You can copy them cross-region.

---

## Clean Up

- Unmount EFS on both instances: `sudo umount /mnt/shared`
- Terminate both EC2 instances.
- EFS → delete `lab-efs` (must delete mount targets first if manually created).
- EC2 → Volumes → detach and delete gp3 and io2 volumes.
- EC2 → Snapshots → delete the snapshot.
- Delete the restored volume and any cross-region snapshot copies.
