# Lab: VPC Endpoints — Gateway vs Interface, Endpoint Policies

*Phase 4 · Networking Advanced · Concept: [concepts/networking-advanced.md](../concepts/networking-advanced.md)*

---

## Goal

Understand when and why to use VPC endpoints instead of routing traffic through a NAT Gateway. Experience the cost and security difference between Gateway endpoints (free, for S3/DynamoDB) and Interface endpoints (paid, for any AWS service). Lock down access using an endpoint policy.

---

## Architecture

```
WITHOUT endpoint:
EC2 (private subnet) → NAT GW → IGW → public internet → S3 (costs money, leaves VPC)

WITH Gateway endpoint:
EC2 (private subnet) → VPC Gateway Endpoint → S3 (free, stays inside AWS network)

WITH Interface endpoint (PrivateLink):
EC2 (private subnet) → ENI in subnet → SSM service (private IP, no internet needed)
```

---

## Part A — Baseline: S3 Access Through NAT Gateway

**1. Setup**
- [ ] Use a private subnet EC2 from the vpc-troubleshooting lab (or create one):
  - EC2 in private subnet (no public IP). Security group: allow SSH from bastion or SSM.
  - Private route table has `0.0.0.0/0 → NAT GW`.

**2. Confirm S3 works through NAT**
- [ ] SSM Session Manager → shell into the private EC2.
- [ ] Check what IP reaches S3:
  ```bash
  # List a bucket — goes through NAT GW → internet → S3 public endpoint
  aws s3 ls
  ```
- [ ] In VPC → NAT Gateways → select your NAT GW → Monitoring → `BytesOutToDestination`. You'll see traffic after the S3 call.

**Note the cost implication:** NAT Gateway charges $0.045/GB processed. All S3 traffic from private subnets costs money this way.

---

## Part B — Gateway Endpoint for S3 (Free)

**1. Create the Gateway endpoint**
- [ ] VPC → Endpoints → Create endpoint.
- [ ] Service name: search `com.amazonaws.<region>.s3` → Type: **Gateway**.
- [ ] VPC: your VPC. Route tables: select the **private route table** (the one your EC2 uses).
- [ ] Policy: Full access (default). Create endpoint.

**2. What just happened**
- [ ] Go to the private route table → Routes. You'll see a new entry:
  - Destination: `pl-xxxxx` (AWS managed prefix list for S3)
  - Target: `vpce-xxxxx` (your Gateway endpoint)
  - This is automatically injected — you didn't add it manually.
- [ ] The route is more specific than `0.0.0.0/0 → NAT GW`, so S3 traffic now takes the endpoint route.

**3. Test — S3 still works but now stays private**
- [ ] Shell into EC2:
  ```bash
  aws s3 ls
  aws s3 cp /etc/hostname s3://your-bucket/test.txt
  aws s3 ls s3://your-bucket/
  ```
  → Works. Traffic now goes through the endpoint, not NAT GW.

**4. Verify NAT GW is not involved**
- [ ] Check NAT GW monitoring again → no new bytes for S3 traffic.

**Key exam point:** Gateway endpoints are free, and no ENI is created. They work by injecting routes into your route table. Only S3 and DynamoDB have Gateway endpoints.

---

## Part C — Endpoint Policy: Restrict Which S3 Buckets Are Accessible

The exam tests this: you can lock down the endpoint so only specific buckets are reachable.

**1. Edit the endpoint policy**
- [ ] VPC → Endpoints → select your S3 endpoint → Policy tab → Edit.
- [ ] Replace with:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": [
          "arn:aws:s3:::your-allowed-bucket",
          "arn:aws:s3:::your-allowed-bucket/*"
        ]
      }
    ]
  }
  ```
- [ ] Save.

**2. Test the restriction**
- [ ] Shell into EC2:
  ```bash
  aws s3 ls s3://your-allowed-bucket/    # Works
  aws s3 ls s3://any-other-bucket/       # Access Denied
  ```
  → The endpoint policy blocks access to any bucket not listed, even if the EC2's IAM role would normally allow it.

**Exam trap:** Endpoint policy AND IAM policy must both allow the action. The more restrictive one wins.

---

## Part D — Interface Endpoint for SSM (PrivateLink)

Without an Interface endpoint, EC2 in a private subnet (no NAT GW) can't reach SSM → you can't use Session Manager. With the Interface endpoint, SSM becomes reachable via a private IP inside your VPC.

**1. Create 3 Interface endpoints (SSM needs all three)**
- [ ] VPC → Endpoints → Create endpoint for each:
  - `com.amazonaws.<region>.ssm` — Type: **Interface**
  - `com.amazonaws.<region>.ssmmessages` — Type: **Interface**
  - `com.amazonaws.<region>.ec2messages` — Type: **Interface**
- [ ] For each: VPC: your VPC. Subnet: private subnet. Security group: allow HTTPS (443) inbound from your EC2's security group.
- [ ] Enable DNS: **Yes** (private DNS name — critical).
- [ ] Create all three.

**2. Remove NAT Gateway route (to prove no internet needed)**
- [ ] Private route table → Routes → Delete `0.0.0.0/0 → NAT GW` route.

**3. Verify EC2 has no internet**
- [ ] SSM Session Manager → connect to EC2:
  ```bash
  curl https://checkip.amazonaws.com --connect-timeout 3    # Timeout — no internet
  ```

**4. Verify SSM still works via the Interface endpoint**
- [ ] The Session Manager shell worked — that means SSM connectivity is going through the Interface endpoint, not the internet.
- [ ] Inside the shell:
  ```bash
  # Check the SSM endpoint DNS
  nslookup ssm.<region>.amazonaws.com
  # Returns a 10.x.x.x private IP — your endpoint ENI in the subnet
  ```

**Why private DNS matters:** When `Enable private DNS` is checked, the standard SSM hostname resolves to your endpoint's private IP inside the VPC. No code changes needed.

---

## Part E — Compare Gateway vs Interface Endpoints

| | Gateway Endpoint | Interface Endpoint (PrivateLink) |
|---|---|---|
| Services | S3, DynamoDB only | Any AWS service |
| Cost | **Free** | ~$0.01/AZ/hour + data |
| Mechanism | Route table entry | ENI in your subnet (private IP) |
| DNS | No change needed | Needs private DNS enabled |
| Endpoint policy | Yes | Yes |
| Cross-region | No | Yes (via PrivateLink) |

**Exam tip:** If the question involves S3 or DynamoDB traffic from private subnets → Gateway endpoint (free). Everything else → Interface endpoint.

---

## Key Things to Observe

- Gateway endpoint adds a route automatically to your route table — no ENI, no cost.
- Interface endpoint creates an ENI in your subnet — costs money but works for any AWS service.
- Endpoint policies are separate from IAM policies — both must allow for access to succeed.
- SSM Session Manager requires 3 Interface endpoints: `ssm`, `ssmmessages`, `ec2messages`.
- Private DNS on Interface endpoints means existing SDK/CLI calls work with zero code change.
- Without either endpoint, private subnet EC2s need a NAT GW to reach AWS services — incurring NAT data costs.

---

## Clean Up

- Delete the 4 VPC endpoints (S3, SSM ×3).
- Re-add NAT GW route to private route table if you want internet access back.
- Terminate EC2 if created for this lab.
