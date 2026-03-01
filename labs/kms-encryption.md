# Lab: KMS — Customer-Managed Key, Encryption & CloudTrail Audit

*Phase 5 · Week 10 · Sat May 9 · Concept: [concepts/security.md](../concepts/security.md)*

---

## Goal

Create a customer-managed KMS key, use it to encrypt an S3 object, enforce key policy, and audit usage in CloudTrail. Also set up a Secrets Manager secret with rotation.

---

## Part A — Create a Customer-Managed KMS Key

- [ ] KMS → Customer managed keys → Create key.
- [ ] Key type: **Symmetric** (AES-256, used for encrypt/decrypt). Key usage: Encrypt and decrypt.
- [ ] Alias: `alias/lab-key`.
- [ ] Key administrator: your IAM user. Key users: your IAM user.
- [ ] Create.
- [ ] Note the Key ARN — you'll use it below.

**Observe the key policy (generated automatically):**
- [ ] Go to the key → Key policy tab. Note:
  - The root account has full control (via IAM).
  - The key user can call `kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey`.
  - Without this key policy, IAM policies alone cannot grant access to the key.

---

## Part B — Encrypt an S3 Object with the CMK

**1. Create an S3 bucket**
- [ ] Create bucket: `my-kms-test-<random>`. Block all public access ON.

**2. Upload with SSE-KMS using your CMK**
- [ ] Upload a file → Properties → Server-side encryption → SSE-KMS → **Customer managed key** → select `alias/lab-key`.
- [ ] Upload.

**3. Verify encryption**
- [ ] Click the object → Properties → Encryption: shows SSE-KMS with your key ARN.

**4. Enforce encryption via bucket policy**
- [ ] Go to bucket → Permissions → Bucket policy → Add this policy to deny uploads without encryption:
  ```json
  {
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::my-kms-test-<random>/*",
    "Condition": {
      "StringNotEquals": {
        "s3:x-amz-server-side-encryption": "aws:kms"
      }
    }
  }
  ```
- [ ] Try uploading a file with SSE-S3 (not KMS) → should be denied.
- [ ] Upload with SSE-KMS → should succeed.

---

## Part C — Audit KMS Usage in CloudTrail

- [ ] Go to CloudTrail → Event history.
- [ ] Filter by: Event source = `kms.amazonaws.com`.
- [ ] Find events:
  - `GenerateDataKey` — S3 called KMS when you uploaded the encrypted file.
  - `Decrypt` — KMS called when you viewed/downloaded the file.
- [ ] Click a `Decrypt` event → see: who called it, which key, which resource, timestamp.
- [ ] This is the **audit trail** for SSE-KMS — every decryption is logged. SSE-S3 does NOT appear in CloudTrail.

---

## Part D — Key Rotation

- [ ] KMS → your key → Key rotation tab.
- [ ] Enable automatic key rotation (annual by default).
- [ ] Note: rotation creates new key material but keeps the same key ARN. Existing encrypted data doesn't need to be re-encrypted — KMS tracks which key material version was used.

---

## Part E — Secrets Manager (Quick)

- [ ] Secrets Manager → Store a new secret.
- [ ] Secret type: **Other type of secret**.
- [ ] Key-value: `username=admin`, `password=MySecretPass123`.
- [ ] Secret name: `lab/db-credentials`. KMS key: your `alias/lab-key`.
- [ ] No rotation for now (rotation needs Lambda in a VPC for RDS). Create.

**Retrieve the secret:**
- [ ] Click the secret → Retrieve secret value → you see the plaintext values.
- [ ] In CloudShell:
  ```bash
  aws secretsmanager get-secret-value --secret-id lab/db-credentials
  ```
  Returns the secret as JSON. In an app, you'd call this API instead of hardcoding the password.

**Observe vs SSM Parameter Store:**
- [ ] SSM → Parameter Store → Create parameter → SecureString (encrypted with KMS).
- [ ] Name: `/lab/config/db-password`. Value: `SimplePassword123`. KMS key: your lab key.
- [ ] Retrieve: `aws ssm get-parameter --name /lab/config/db-password --with-decryption`
- [ ] Cost comparison: Parameter Store (SecureString) = $0.05/10,000 API calls. Secrets Manager = $0.40/secret/month.

---

## Key Things to Observe

- KMS key policy is mandatory — IAM policy alone is not sufficient to grant key access.
- Every SSE-KMS decrypt appears in CloudTrail. SSE-S3 decrypts do not.
- Key rotation keeps the same ARN — no disruption to existing encrypted data.
- Secrets Manager: built for credentials with auto-rotation. Parameter Store: built for config with cost-conscious use.
- Envelope encryption: S3 called `GenerateDataKey` (not `Encrypt`) — the data never went through KMS. Only the data key did.

---

## Clean Up

- Secrets Manager → delete secret (30-day recovery window — note this delay; can force delete with AWS CLI).
- SSM Parameter Store → delete parameter.
- S3 → empty and delete bucket.
- KMS → Schedule key deletion (minimum 7-day waiting period — you cannot immediately delete a CMK).
