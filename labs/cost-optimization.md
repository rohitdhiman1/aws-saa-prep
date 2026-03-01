# Lab: Cost Optimization — Spot, Reserved Instances, Savings Plans & Storage Tiering

*Phase 6 · Review · Concept: [cheatsheets/cost.md](../cheatsheets/cost.md)*

---

## Goal

Explore the four main cost-optimization levers for the SAA-C03: EC2 purchasing options (Spot, RI, Savings Plans), S3 storage class transitions, Compute Optimizer recommendations, and Cost Explorer analysis. This lab is more exploratory than build-heavy — the goal is to understand the decision matrix, not deploy production workloads.

---

## Part A — EC2 Purchasing Options (Console Exploration)

**1. On-Demand pricing baseline**
- [ ] EC2 → Launch instance (don't actually launch — just observe pricing).
- [ ] Select `t3.medium` in `us-east-1` → note the On-Demand price (~$0.0416/hr).
- [ ] This is the baseline everything else is compared against.

**2. Spot Instances**
- [ ] EC2 → Spot Requests → Pricing History.
- [ ] Select `t3.medium` → observe the Spot price (~60–90% discount vs On-Demand).
- [ ] Note: Spot prices vary by AZ and instance type. The price shown is the current market rate.

- [ ] EC2 → Launch instance → Advanced details → Purchasing option: **Spot**.
- [ ] **Key setting:** Interruption behavior — the default is **Terminate**. Options: Terminate, Stop, Hibernate.
- [ ] Don't launch. Just observe the configuration.

**Exam decision tree for Spot:**
| Scenario | Spot? |
|---|---|
| Batch processing, CI/CD, data analysis | Yes — tolerates interruption |
| Stateless web tier behind ASG | Yes — ASG replaces interrupted instances |
| Database server | No — data loss risk |
| Real-time trading app | No — can't tolerate interruption |

**3. Spot Fleet (conceptual)**
- [ ] EC2 → Spot Requests → Request Spot Instances.
- [ ] Observe the fleet configuration: you can mix instance types and AZs.
- [ ] Allocation strategy options:
  - `lowestPrice` — cheapest instances (higher interruption risk)
  - `capacityOptimized` — lowest interruption probability (exam favourite)
  - `diversified` — spread across pools
- [ ] Cancel without creating.

**4. Reserved Instances**
- [ ] EC2 → Reserved Instances → Purchase Reserved Instances.
- [ ] Filter: `t3.medium`, `us-east-1`, Linux.
- [ ] Compare the three payment options:

| Payment | Discount | Flexibility |
|---|---|---|
| All Upfront | ~40% off On-Demand | Locked for 1 or 3 years |
| Partial Upfront | ~35% off | Locked |
| No Upfront | ~30% off | Locked |

- [ ] Standard RI: locked to instance type + AZ. Convertible RI: can change instance family (lower discount, ~20%).
- [ ] Don't purchase. Just note the prices.

**5. Savings Plans**
- [ ] AWS Cost Management → Savings Plans → Purchase Savings Plans.
- [ ] Two types:
  - **Compute Savings Plan** — applies to EC2, Fargate, and Lambda. Most flexible (~20% discount).
  - **EC2 Instance Savings Plan** — locked to instance family + region. Higher discount (~30%).
- [ ] Observe: you commit to a $/hr spend (e.g., $10/hr), not a specific instance.
- [ ] Don't purchase.

**Exam decision matrix:**
| Need | Best option |
|---|---|
| Steady-state workload, known instance type | Standard RI (1 or 3 year) |
| Steady-state but may change instance family | Convertible RI |
| Steady-state across EC2 + Fargate + Lambda | Compute Savings Plan |
| Fault-tolerant batch jobs | Spot Instances |
| Short-term spike (days/weeks) | On-Demand |
| Dedicated hardware (compliance) | Dedicated Hosts (with RI for discount) |

---

## Part B — S3 Storage Class Analysis

**1. Create a bucket with lifecycle rules**
- [ ] Create bucket: `cost-lab-<random>`.
- [ ] Upload 3 test files (any small files).

**2. Set up lifecycle transitions**
- [ ] Bucket → Management → Lifecycle rules → Create rule.
- [ ] Rule name: `cost-transitions`. Apply to all objects.
- [ ] Transitions:
  - Move to **S3 Standard-IA** after **30 days**.
  - Move to **S3 Glacier Instant Retrieval** after **90 days**.
  - Move to **S3 Glacier Deep Archive** after **180 days**.
- [ ] Expiration: delete after **365 days** (optional).
- [ ] Create rule.

**3. Understand the cost ladder**

| Storage Class | $/GB/mo | Retrieval | Min Duration | Use Case |
|---|---|---|---|---|
| Standard | $0.023 | Instant | None | Frequent access |
| Intelligent-Tiering | $0.023 | Instant | None | Unknown access pattern |
| Standard-IA | $0.0125 | Instant | 30 days | Infrequent, needs fast access |
| One Zone-IA | $0.01 | Instant | 30 days | Non-critical, re-creatable |
| Glacier Instant | $0.004 | Instant | 90 days | Archive, rare but immediate |
| Glacier Flexible | $0.0036 | 1–12 hrs | 90 days | Archive, can wait hours |
| Glacier Deep Archive | $0.00099 | 12–48 hrs | 180 days | Compliance, rarely accessed |

**4. S3 Intelligent-Tiering**
- [ ] Upload a file with storage class: **Intelligent-Tiering**.
- [ ] How it works: automatically moves objects between Frequent and Infrequent tiers (and optional Archive tiers) based on access patterns. No retrieval fees.
- [ ] Best for: unpredictable access patterns where you don't want to manage lifecycle rules manually.
- [ ] Cost: same as Standard for frequently accessed. Small monitoring fee ($0.0025/1000 objects/month).

**Exam trap:** Standard-IA and One Zone-IA have a **128 KB minimum object charge** and a **30-day minimum storage charge**. Lots of small files that are deleted quickly can cost MORE than Standard.

---

## Part C — Compute Optimizer (Observation)

- [ ] AWS Compute Optimizer → Dashboard.
- [ ] If you have running EC2 instances, Compute Optimizer shows:
  - Over-provisioned instances (e.g., `m5.xlarge` averaging 5% CPU → recommends `t3.medium`).
  - Under-provisioned instances (CPU pegged at 90% → recommends larger).
  - EBS volumes with excessive IOPS provisioned.
  - Lambda functions with too much/too little memory.
- [ ] If no instances running, review the sample recommendations page.
- [ ] Note: Compute Optimizer needs **14 days** of CloudWatch metrics before making recommendations.

---

## Part D — Cost Explorer Quick Tour

- [ ] AWS Cost Management → Cost Explorer.
- [ ] Filter by service → see which services cost the most.
- [ ] Group by: Usage Type → identify costly dimensions (data transfer, storage, compute hours).
- [ ] Right-size recommendations: Cost Explorer → Recommendations tab (if available with enough history).

**Key cost patterns for the exam:**
- Data transfer OUT to internet is charged. Data transfer IN is free.
- Cross-AZ data transfer: $0.01/GB each direction. Cross-region: $0.02/GB.
- NAT Gateway: $0.045/hr + $0.045/GB processed — one of the most surprising costs.
- VPC endpoints (Gateway type for S3/DynamoDB): free — saves NAT GW costs for S3 traffic.
- CloudFront: often cheaper than direct S3 for high-volume downloads (lower data transfer rate).

---

## Part E — Exam Scenario Practice

Match each scenario to the best cost strategy:

| # | Scenario | Answer |
|---|---|---|
| 1 | Company runs same 20 EC2 instances 24/7 for 3 years | Standard RI, All Upfront, 3-year |
| 2 | Dev team runs batch jobs nightly, can restart if interrupted | Spot Instances |
| 3 | Web app with unpredictable traffic spikes | ASG with On-Demand (base) + Spot (scale) |
| 4 | 10 TB of logs, accessed once a year for audit | S3 Glacier Deep Archive |
| 5 | Media files with unpredictable access pattern | S3 Intelligent-Tiering |
| 6 | Company uses EC2 + Fargate + Lambda, steady baseline | Compute Savings Plan |
| 7 | Compliance requires 7-year retention, never accessed | S3 Glacier Deep Archive with vault lock |
| 8 | Reduce NAT Gateway costs for S3-heavy traffic | S3 Gateway VPC Endpoint (free) |

---

## Key Things to Observe

- Spot Instances save up to 90% but can be interrupted with 2-minute warning. Use `capacityOptimized` allocation strategy.
- Reserved Instances are capacity reservations for 1 or 3 years. Standard RIs give the best discount but least flexibility.
- Savings Plans are the modern alternative to RIs — commit to $/hr spend, not specific instances.
- S3 lifecycle rules automate transitions down the cost ladder. Intelligent-Tiering automates it based on access.
- Data transfer costs are a hidden exam topic — know when cross-AZ, cross-region, and NAT GW charges apply.
- Compute Optimizer requires CloudWatch metrics history — it's not instant.

---

## Clean Up

- S3 → empty and delete `cost-lab-<random>`.
- No other resources to clean (this lab is mostly observational).
