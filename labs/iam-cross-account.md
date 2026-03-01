# Lab: IAM Cross-Account Role Assumption

*Phase 1 · IAM · Two ACG sandboxes required · Concept: [concepts/iam.md](../concepts/iam.md)*

---

## Goal

Use two separate AWS accounts to actually experience cross-account role assumption — the most tested IAM scenario on the exam. You will set up a role in Account B, assume it from Account A, and enforce MFA as a condition.

---

## Setup

- **Account A** = the *trusting* side (your IAM user lives here, wants access to Account B)
- **Account B** = the *trusted* side (has the resource — an S3 bucket — and the cross-account role)

Open two browser tabs or two incognito windows, one logged into each sandbox.

---

## Part A — Create the Role in Account B

In **Account B**:

**1. Create an S3 bucket to test access**
- [ ] Create bucket: `account-b-cross-acct-<random>`. Upload one file.

**2. Create the cross-account IAM role**
- [ ] IAM → Roles → Create role.
- [ ] Trusted entity type: **AWS account** → choose "Another AWS account".
- [ ] Account ID: paste Account A's account ID (find it in Account A under the top-right dropdown).
- [ ] Check "Require MFA". This adds a condition to the trust policy.
- [ ] Permissions: attach `AmazonS3ReadOnlyAccess`.
- [ ] Role name: `CrossAccountS3ReadRole`. Create.

**3. Note the role ARN**
- [ ] Copy the full role ARN: `arn:aws:iam::<account-B-id>:role/CrossAccountS3ReadRole`

**4. Inspect the trust policy (understand what was created)**
- [ ] Open the role → Trust relationships tab. You'll see:
  ```json
  {
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::<account-A-id>:root" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "Bool": { "aws:MultiFactorAuthPresent": "true" }
    }
  }
  ```
- Key insight: `Principal: root` means any IAM user in Account A can assume it (if they have permission in their own identity policy too). You can narrow this to a specific user ARN.

---

## Part B — Grant AssumeRole Permission in Account A

In **Account A**:

**1. Create an IAM user for testing**
- [ ] IAM → Users → Create user: `cross-acct-tester`. Enable console access + programmatic access.
- [ ] Attach this inline policy to the user:
  ```json
  {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::<account-B-id>:role/CrossAccountS3ReadRole"
  }
  ```
- This is the identity-side permission. The role's trust policy (Account B) is the resource-side permission. **Both must allow** for the assumption to succeed.

**2. Enable MFA for the user**
- [ ] Open `cross-acct-tester` → Security credentials → Assign MFA device (virtual — use Google Authenticator or similar).

---

## Part C — Assume the Role

In **Account A** (logged in as `cross-acct-tester` or using its access keys in CloudShell):

**1. Get a session token with MFA first** (required because the trust policy demands MFA)
```bash
aws sts get-session-token \
  --serial-number arn:aws:iam::<account-A-id>:mfa/<your-mfa-device-name> \
  --token-code <6-digit-MFA-code>
```
This returns: `AccessKeyId`, `SecretAccessKey`, `SessionToken` (valid ~12 hours).

**2. Export the temp credentials**
```bash
export AWS_ACCESS_KEY_ID=<from above>
export AWS_SECRET_ACCESS_KEY=<from above>
export AWS_SESSION_TOKEN=<from above>
```

**3. Now assume the cross-account role**
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<account-B-id>:role/CrossAccountS3ReadRole \
  --role-session-name my-cross-acct-session
```
Returns a new set of credentials (scoped to Account B's role, valid 1 hour by default).

**4. Export the role credentials and test**
```bash
export AWS_ACCESS_KEY_ID=<role AccessKeyId>
export AWS_SECRET_ACCESS_KEY=<role SecretAccessKey>
export AWS_SESSION_TOKEN=<role SessionToken>

# Test: list Account B's S3 bucket
aws s3 ls s3://account-b-cross-acct-<random>     # should work
aws s3 mb s3://test-new-bucket-123               # should FAIL (read-only role)
```

---

## Part D — What Breaks and Why

**Test 1: Assume without MFA**
- [ ] As `cross-acct-tester`, try `sts assume-role` **without** first calling `get-session-token`.
- [ ] Result: `AccessDenied` — the trust policy condition `aws:MultiFactorAuthPresent: true` is not satisfied.

**Test 2: Wrong account trying to assume**
- [ ] From Account B itself, try to assume `CrossAccountS3ReadRole` → `AccessDenied` because the principal in the trust policy is Account A only.

**Test 3: Narrow the trust policy to a specific user**
- [ ] In Account B, edit the trust policy — change `root` to the specific user ARN:
  `arn:aws:iam::<account-A-id>:user/cross-acct-tester`
- [ ] Now a different user in Account A cannot assume the role even if they have the `sts:AssumeRole` IAM policy.

---

## Key Things to Observe

- **Two-sided permission** — both the trust policy (Account B) AND the identity policy (Account A) must allow the assumption. Either side can block it.
- **MFA condition on trust policy** — forces the caller to have authenticated with MFA in their current session. `get-session-token` with MFA token is the mechanism.
- **Session token is a third credential** — access key + secret alone are not enough when a session token is required.
- **Temp creds from AssumeRole** — expire in 1 hour (default). If the role's `MaxSessionDuration` is set higher, you can request longer.
- **`root` vs specific user in trust policy** — `root` means any identity in the account that has `sts:AssumeRole` permission. Specific user ARN is more restrictive and recommended.

---

## Clean Up

Account A: Delete user `cross-acct-tester` and attached policy.
Account B: Delete role `CrossAccountS3ReadRole`. Delete S3 bucket.
