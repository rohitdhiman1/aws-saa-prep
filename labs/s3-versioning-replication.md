# Lab: S3 — Versioning, Lifecycle, Replication & Encryption

*Phase 2 · Week 4 · Sat Mar 28 · Concept: [concepts/storage.md](../concepts/storage.md)*

---

## Goal

Experience S3 versioning behaviour, lifecycle transitions, cross-region replication, and encryption options hands-on.

---

## Part A — Versioning

**1. Create a versioned bucket**
- [ ] Create a bucket: `my-versioned-bucket-<random>` in `us-east-1`. Block all public access ON.
- [ ] Properties → Bucket Versioning → Enable.

**2. Upload the same file multiple times**
- [ ] Create a file `notes.txt` locally with content `version 1`.
- [ ] Upload to S3.
- [ ] Edit the file locally: `version 2`. Upload again (same key `notes.txt`).
- [ ] Edit again: `version 3`. Upload again.

**3. Inspect versions**
- [ ] In the bucket, toggle "Show versions" ON.
- [ ] See 3 versions of `notes.txt` each with a unique Version ID.
- [ ] Download the oldest version ID directly — confirm it says `version 1`.

**4. Observe delete behaviour**
- [ ] Toggle "Show versions" OFF. Delete `notes.txt`.
- [ ] Toggle "Show versions" ON — the file is not gone! A **delete marker** was created.
- [ ] To restore: delete the delete marker → file reappears.
- [ ] To permanently delete: delete a specific version by Version ID.

---

## Part B — Lifecycle Rules

- [ ] Go to bucket → Management → Lifecycle rules → Create rule.
- [ ] Rule name: `archive-old-versions`. Scope: all objects.
- [ ] Add transitions:
  - Current version: After 30 days → S3 Standard-IA.
  - Current version: After 90 days → S3 Glacier Instant Retrieval.
  - Non-current versions: After 1 day → S3 Glacier Flexible Retrieval (simulates old version cleanup).
- [ ] Save. Note: transitions won't happen immediately (they're time-based), but the rule is now set.
- [ ] Observe: lifecycle rules appear in Management tab with their configured actions.

---

## Part C — Cross-Region Replication (CRR)

**1. Create destination bucket**
- [ ] Create a second bucket in `eu-west-1` (different region): `my-replica-bucket-<random>`.
- [ ] Enable versioning on the destination bucket (required for CRR).

**2. Set up replication on source bucket**
- [ ] On source bucket → Management → Replication rules → Create rule.
- [ ] Status: Enabled. Replicate entire bucket.
- [ ] Destination: choose `my-replica-bucket-<random>` (different region).
- [ ] IAM role: create new role (AWS creates it automatically).
- [ ] Save.

**3. Test**
- [ ] Upload a new file `test-replication.txt` to the source bucket.
- [ ] Wait ~1–2 minutes. Check the destination bucket in `eu-west-1` — the file should appear.
- [ ] Note: Files uploaded BEFORE the rule was created are NOT replicated.

---

## Part D — Encryption

**1. SSE-S3 (default)**
- [ ] Upload any file. View its Properties → Encryption → see `SSE-S3` by default (since Jan 2023).

**2. SSE-KMS**
- [ ] Upload a new file → Properties → Server-side encryption → SSE-KMS → AWS managed key (`aws/s3`).
- [ ] Upload. View properties → confirm `SSE-KMS` and the KMS key ID.
- [ ] Go to CloudTrail (or KMS → CloudTrail) → find `Decrypt` API call log entry when you download the file.

**3. Pre-signed URL**
- [ ] With the bucket still blocking public access, try opening the S3 URL directly → Access Denied.
- [ ] In the console, select the object → Object actions → Share with pre-signed URL → expiry: 5 minutes.
- [ ] Copy the URL → open in an incognito browser → file downloads (even without AWS credentials).
- [ ] Wait 5 minutes → try again → URL expired, access denied.

---

## Part E — Static Website Hosting (Quick)

- [ ] Create a third bucket: `my-static-site-<random>`. **Disable** Block all public access.
- [ ] Properties → Static website hosting → Enable. Index document: `index.html`.
- [ ] Upload `index.html`:
  ```html
  <h1>Hello from S3!</h1>
  ```
- [ ] Bucket policy → add:
  ```json
  {
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-static-site-<random>/*"
  }
  ```
- [ ] Open the website endpoint URL (shown in Static website hosting settings) → see your page.
- [ ] Note: this is HTTP only. HTTPS requires CloudFront.

---

## Key Things to Observe

- DELETE with versioning = delete marker; object is not gone.
- CRR doesn't replicate pre-existing objects.
- SSE-KMS creates a CloudTrail entry per download (important for audit).
- Pre-signed URLs bypass bucket privacy — time-limited, credential-bound.
- S3 static hosting = HTTP only. CloudFront = HTTPS.

---

## Clean Up

- Delete all objects in all 3 buckets (including all versions and delete markers).
- Delete all 3 buckets.
