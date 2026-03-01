# Lab: IAM Sandbox

*Phase 1 · Week 1 · Concept: [concepts/iam.md](../concepts/iam.md)*

---

## Goal

Practise creating and testing IAM primitives, policy evaluation, and role assumption in an ACG/AWS sandbox.

---

## Tasks

### 1. Create Users and Groups
- [ ] Create two IAM users: `alice` and `bob` (console access, no programmatic).
- [ ] Create a group `developers`; attach `PowerUserAccess` managed policy.
- [ ] Add `alice` to `developers`. Verify `alice` can launch EC2 but cannot create IAM users.
- [ ] Verify `bob` (no group) cannot do anything.

### 2. Test Policy Evaluation (Explicit Deny)
- [ ] Attach an inline policy to `alice` that Denies `ec2:TerminateInstances`.
- [ ] Launch an EC2 instance as `alice`; attempt to terminate it — confirm denied.
- [ ] Remove the deny; confirm termination works.

### 3. Create and Assume a Role
- [ ] Create an IAM role `S3ReadRole` with trust policy allowing your IAM user to assume it.
- [ ] Attach `AmazonS3ReadOnlyAccess` to the role.
- [ ] From CLI: `aws sts assume-role --role-arn <ARN> --role-session-name test`
- [ ] Export the temp credentials; run `aws s3 ls` → should succeed.
- [ ] Run `aws s3 mb s3://test-bucket-xxx` → should fail (read-only role).

### 4. Permission Boundary (Stretch)
- [ ] Create a permission boundary policy that only allows `s3:*`.
- [ ] Attach boundary to a new user `charlie` who also has `AdministratorAccess`.
- [ ] Verify `charlie` cannot create EC2 (boundary caps at S3 only).

### 5. SCP Simulation (If Org Access Available)
- [ ] Create an SCP that denies `s3:DeleteBucket`.
- [ ] Attach to a test OU.
- [ ] Verify even admin cannot delete S3 buckets in that OU's accounts.

---

## Key Things to Observe

- Explicit Deny overrides all allows.
- Permission boundary limits max permissions; does not grant.
- Temp credentials from `AssumeRole` include a session token — all three pieces required.

---

## Clean Up

- Delete created users, groups, roles, and policies.
- Terminate any EC2 instances launched.
