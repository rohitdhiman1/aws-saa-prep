# Lab: Lambda in a VPC — Private Access & Internet Loss

*Phase 2 · Compute · Concept: [concepts/compute.md](../concepts/compute.md)*

---

## Goal

Experience the exact behaviour the exam tests: a Lambda function placed inside a VPC loses public internet access by default. You'll break it, understand why, and fix it with a NAT Gateway. Then use it to access RDS in a private subnet — the key production pattern.

---

## Architecture

```
BEFORE fix:
Lambda (private subnet) ──► tries to call external API ──► TIMEOUT (no route to internet)

AFTER fix:
Lambda (private subnet) ──► NAT GW (public subnet) ──► Internet ──► external API ✓

PRIVATE ACCESS (no internet needed):
Lambda (private subnet) ──► RDS (private subnet, same VPC) ──► direct private IP ✓
```

---

## Part A — Lambda Outside VPC (Baseline)

**1. Create a Lambda function (no VPC)**
- [ ] Lambda → Create function → `test-internet-access`. Runtime: Python 3.12. No VPC.
- [ ] Code:
  ```python
  import urllib.request

  def lambda_handler(event, context):
      try:
          response = urllib.request.urlopen('https://httpbin.org/get', timeout=5)
          return {'status': 'OK', 'response': str(response.read()[:100])}
      except Exception as e:
          return {'status': 'FAILED', 'error': str(e)}
  ```
- [ ] Test → returns `status: OK`. Lambda outside VPC has internet access by default.

---

## Part B — Move Lambda into VPC (Break Internet Access)

**1. Set up a VPC with a private subnet**
- [ ] Use the VPC from the vpc-troubleshooting lab, or create a new one:
  - VPC `10.0.0.0/16` · Public subnet `10.0.1.0/24` · Private subnet `10.0.2.0/24`
  - IGW attached to VPC. Public route table: `0.0.0.0/0 → IGW`.
  - Private route table: **no internet route** (no NAT GW yet).

**2. Configure Lambda to run inside the VPC**
- [ ] Lambda → `test-internet-access` → Configuration → VPC → Edit.
- [ ] VPC: your VPC. Subnets: **private subnet only**. Security group: create new, allow all outbound.
- [ ] Save. Lambda now deploys an ENI in your private subnet.

**3. Test — expect failure**
- [ ] Test the function → `status: FAILED` with a timeout error.
- [ ] Why: Lambda is in the private subnet, which has no route to the internet.

---

## Part C — Fix with NAT Gateway

**1. Create a NAT Gateway**
- [ ] EC2 → NAT Gateways → Create. **Subnet: public subnet** (critical). Allocate Elastic IP. Create.
- [ ] Wait for NAT GW to become Available (~1–2 min).

**2. Add route to private subnet**
- [ ] Private route table → Routes → Edit → Add route: `0.0.0.0/0 → nat-xxx`.

**3. Test Lambda again**
- [ ] Test `test-internet-access` → `status: OK` again.
- [ ] The path: Lambda ENI → private subnet → NAT GW (public subnet) → IGW → internet.

**Key insight:** The Lambda function's IP seen by `httpbin.org` is the NAT GW's Elastic IP — not a Lambda IP. This is how you whitelist Lambda's outbound IP at external services.

---

## Part D — Lambda Accessing RDS in Private Subnet (No Internet Needed)

**1. Create an RDS instance (MySQL) in the private subnet**
- [ ] Create an RDS MySQL instance (free tier). Subnet group: private subnet. No public access.
- [ ] Security group: allow MySQL (3306) inbound from the **Lambda security group**.

**2. Update Lambda to connect to RDS**
- [ ] Create a new Lambda `rds-query-test` in the same private subnet. Runtime: Python 3.12.
- [ ] Add this code (replace placeholders):
  ```python
  import pymysql  # Layer or include in deployment package

  def lambda_handler(event, context):
      try:
          conn = pymysql.connect(
              host='<rds-endpoint>',
              user='admin',
              password='<password>',
              database='mysql',
              connect_timeout=5
          )
          cursor = conn.cursor()
          cursor.execute("SELECT VERSION()")
          version = cursor.fetchone()
          conn.close()
          return {'status': 'connected', 'mysql_version': str(version)}
      except Exception as e:
          return {'status': 'FAILED', 'error': str(e)}
  ```
- [ ] Add `pymysql` as a Lambda Layer (or create a deployment package zip with it).
- [ ] Test → `status: connected`. Lambda reached RDS via private IP, no internet involved.

**3. Break the security group to feel the exam trap**
- [ ] Remove the inbound rule from the RDS security group that allows Lambda's SG.
- [ ] Test Lambda → connection timeout.
- [ ] Add it back → works again.
- [ ] **Exam trap:** Lambda and RDS are in the same VPC — this is NOT a routing issue. It's a security group issue. RDS SG must explicitly allow Lambda's SG on port 3306.

---

## Key Things to Observe

- Lambda in a VPC has no internet by default — it needs a NAT GW in a public subnet.
- Lambda outside a VPC always has internet — it runs on AWS-managed infrastructure.
- Private-to-private communication within the same VPC (Lambda → RDS) needs no internet, only correct security group rules.
- The NAT GW Elastic IP becomes Lambda's public IP — useful for IP whitelisting at 3rd-party APIs.
- Lambda function cold start is slightly longer when in a VPC (ENI creation), but this has improved significantly since 2019.

---

## Clean Up

- Delete Lambda functions.
- Delete RDS instance.
- Delete NAT Gateway. Release the Elastic IP.
- Delete Lambda security group.
