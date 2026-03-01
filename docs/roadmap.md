# Study Roadmap — SAA-C03

**Format:** Work topic by topic at your own pace. Week numbers are a guide, not a deadline.
**Daily target:** ~2 hours weekdays · 3–4 hours weekends (adjust freely)
**Exam target:** May 2026

---

## Phase Overview

| Phase | Topics | Focus | Weeks |
|---|---|---|---|
| 1 | IAM · VPC Basics | Identity + core networking | 1–2 |
| 2 | Compute · Storage | EC2/Lambda/containers + S3/EBS/EFS | 3–4 |
| 3 | Databases | RDS/Aurora + DynamoDB/ElastiCache | 5–6 |
| 4 | Networking Deep Dive | Advanced VPC + Edge/DNS/CDN | 7–8 |
| 5 | Messaging · Analytics · Security | SQS/SNS/Kinesis + Glue/Athena + KMS | 9–10 |
| 6 | Mock Exams | Full timed exams + weak-area review | 11–12 |
| 7 | Final Review | Cheatsheets + exam strategy | 13 |

---

## Score Targets

| Phase | Activity | Target |
|---|---|---|
| 1 | Phase 1 practice | ≥ 70% |
| 2 | Phase 2 practice | ≥ 70% |
| 3 | Phase 3 practice | ≥ 70% |
| 4 | Phase 4 practice | ≥ 75% |
| 5 | Phase 5 practice | ≥ 75% |
| 6 | Mock exam #1 | ≥ 72% |
| 6 | Mock exam #3 | ≥ 80% |
| Exam | SAA-C03 | ≥ 720 / 1000 |

---

---

# Phase 1 — IAM & VPC Basics

---

## IAM
*Week 1*

| Step | File |
|---|---|
| 📖 Read | [`concepts/iam.md`](../concepts/iam.md) |
| 🔬 Lab 1 | [`labs/iam-sandbox.md`](../labs/iam-sandbox.md) |
| 🔬 Lab 2 | [`labs/iam-cross-account.md`](../labs/iam-cross-account.md) |
| 🗺 Draw | [`diagrams/iam.md`](../diagrams/iam.md) |
| ✅ Cheatsheet | [`cheatsheets/iam.md`](../cheatsheets/iam.md) |

**Lab 1 covers:** Users · Groups · Roles · Policy evaluation (explicit deny) · AssumeRole · Permission boundaries
**Lab 2 covers:** Cross-account role assumption · STS · Trust policies · MFA enforcement · Two ACG sandboxes

---

## VPC Basics
*Week 2 · Phase 1 checkpoint*

| Step | File |
|---|---|
| 📖 Read | [`concepts/vpc-basics.md`](../concepts/vpc-basics.md) |
| 🔬 Lab | [`labs/vpc-build.md`](../labs/vpc-build.md) |
| 🗺 Draw | [`diagrams/vpc-basics.md`](../diagrams/vpc-basics.md) |
| ✅ Cheatsheet | [`cheatsheets/vpc-basics.md`](../cheatsheets/vpc-basics.md) |
| 📝 Practice Qs | [`mock-exams/phase1.md`](../mock-exams/phase1.md) · target ≥ 70% |

**What the lab covers:** Custom VPC · Public + private subnets · IGW · NAT Gateway · Route tables · Security Groups · NACLs

---

---

# Phase 2 — Compute & Storage

---

## Compute
*Week 3*

| Step | File |
|---|---|
| 📖 Read | [`concepts/compute.md`](../concepts/compute.md) |
| 🔬 Lab 1 | [`labs/compute-asg-alb.md`](../labs/compute-asg-alb.md) |
| 🔬 Lab 2 | [`labs/alb-advanced.md`](../labs/alb-advanced.md) |
| 🔬 Lab 3 | [`labs/lambda-events.md`](../labs/lambda-events.md) |
| 🔬 Lab 4 | [`labs/lambda-vpc.md`](../labs/lambda-vpc.md) |
| 🔬 Lab 5 | [`labs/ecs-fargate.md`](../labs/ecs-fargate.md) |
| 🗺 Draw | [`diagrams/compute.md`](../diagrams/compute.md) |
| ✅ Cheatsheet | [`cheatsheets/compute.md`](../cheatsheets/compute.md) |

**Lab 1 covers:** Launch template · ASG · ALB · Target tracking scaling policy · Health checks
**Lab 2 covers:** Path-based routing · HTTPS redirect · Sticky sessions · Health check failover
**Lab 3 covers:** Lambda S3 trigger (async) · API Gateway trigger (sync) · Concurrency · Cold start observation
**Lab 4 covers:** Lambda outside VPC vs inside VPC · NAT GW for internet · Lambda → RDS private access · SG rules
**Lab 5 covers:** ECS cluster · Execution role vs task role · Fargate service behind ALB · ECS Exec · Service scaling

---

## Storage
*Week 4 · Phase 2 checkpoint*

| Step | File |
|---|---|
| 📖 Read | [`concepts/storage.md`](../concepts/storage.md) |
| 🔬 Lab | [`labs/s3-versioning-replication.md`](../labs/s3-versioning-replication.md) |
| 🗺 Draw | [`diagrams/storage.md`](../diagrams/storage.md) |
| ✅ Cheatsheet | [`cheatsheets/storage.md`](../cheatsheets/storage.md) |
| 📝 Practice Qs | [`mock-exams/phase2.md`](../mock-exams/phase2.md) · target ≥ 70% |

**What the lab covers:** Versioning · Delete markers · Lifecycle rules · CRR · SSE-KMS + CloudTrail · Pre-signed URLs · Static website hosting

---

---

# Phase 3 — Databases

---

## Relational Databases
*Week 5*

| Step | File |
|---|---|
| 📖 Read | [`concepts/databases-relational.md`](../concepts/databases-relational.md) |
| 🔬 Lab 1 | [`labs/rds-multiaz-replica.md`](../labs/rds-multiaz-replica.md) |
| 🔬 Lab 2 | [`labs/aurora.md`](../labs/aurora.md) |
| 🗺 Draw | [`diagrams/databases.md`](../diagrams/databases.md) |
| ✅ Cheatsheet | [`cheatsheets/databases.md`](../cheatsheets/databases.md) |

**Lab 1 covers:** RDS Multi-AZ creation · Forced failover (observe DNS switch) · Read Replica (readable, write-rejected) · Encryption at creation trap
**Lab 2 covers:** Aurora writer/reader endpoints · Forced failover (~20–30s) · Backtrack (rewind without restore) · Serverless v2 ACU scaling

---

## NoSQL, Caching & Analytics Stores
*Week 6 · Phase 3 checkpoint*

| Step | File |
|---|---|
| 📖 Read | [`concepts/databases-nosql.md`](../concepts/databases-nosql.md) |
| 🔬 Lab 1 | [`labs/dynamodb-table-gsi.md`](../labs/dynamodb-table-gsi.md) |
| 🔬 Lab 2 | [`labs/elasticache-redis.md`](../labs/elasticache-redis.md) |
| 🗺 Draw | [`diagrams/databases.md`](../diagrams/databases.md) |
| ✅ Cheatsheet | [`cheatsheets/databases.md`](../cheatsheets/databases.md) |
| 📝 Practice Qs | [`mock-exams/phase3.md`](../mock-exams/phase3.md) · target ≥ 70% |

**Lab 1 covers:** Composite key table · Query vs Scan · Add GSI · Query GSI (eventual consistency only) · Enable Streams + Lambda trigger · TTL deletion
**Lab 2 covers:** Redis cluster (cluster mode off) · Cache-aside pattern with Lambda + DynamoDB · Cache hit vs miss latency · Forced failover · Cluster mode on vs off

---

---

# Phase 4 — Networking Deep Dive

---

## Advanced VPC & Hybrid Connectivity
*Week 7*

| Step | File |
|---|---|
| 📖 Read | [`concepts/networking-advanced.md`](../concepts/networking-advanced.md) |
| 🔬 Lab 1 | [`labs/vpc-peering.md`](../labs/vpc-peering.md) |
| 🔬 Lab 2 | [`labs/vpc-troubleshooting.md`](../labs/vpc-troubleshooting.md) |
| 🔬 Lab 3 | [`labs/vpc-endpoints.md`](../labs/vpc-endpoints.md) |
| 🗺 Draw | [`diagrams/networking.md`](../diagrams/networking.md) |
| ✅ Cheatsheet | [`cheatsheets/networking.md`](../cheatsheets/networking.md) |

**Lab 1 covers:** Peer two VPCs · Update both route tables · Confirm non-transitivity · Test with a third VPC
**Lab 2 covers:** 5 broken VPC scenarios — missing IGW route · wrong SG CIDR · NACL ephemeral port trap · NAT GW in wrong subnet · one-sided peering routes
**Lab 3 covers:** Gateway endpoint for S3 (free, route-based) · Endpoint policy · Interface endpoint for SSM (PrivateLink, ENI-based) · Private DNS

---

## Edge, DNS & Content Delivery
*Week 8 · Phase 4 checkpoint*

| Step | File |
|---|---|
| 📖 Read | [`concepts/edge-dns.md`](../concepts/edge-dns.md) |
| 🔬 Lab | [`labs/cloudfront-route53.md`](../labs/cloudfront-route53.md) |
| 🗺 Draw | [`diagrams/networking.md`](../diagrams/networking.md) |
| ✅ Cheatsheet | [`cheatsheets/networking.md`](../cheatsheets/networking.md) |
| 📝 Practice Qs | [`mock-exams/phase4.md`](../mock-exams/phase4.md) · target ≥ 75% |

**What the lab covers:** CloudFront + S3 OAC (private bucket) · Cache invalidation · Route 53 weighted routing · Route 53 failover routing + health check

> **Architecture exercise:** Before starting the lab, draw a hybrid multi-VPC architecture from memory (VPC peering / TGW / DX / VPN). Check against `diagrams/networking.md`.

---

---

# Phase 5 — Messaging, Analytics & Security

---

## Messaging & Streaming
*Week 9*

| Step | File |
|---|---|
| 📖 Read | [`concepts/messaging.md`](../concepts/messaging.md) |
| 🔬 Lab | [`labs/sqs-sns-fanout.md`](../labs/sqs-sns-fanout.md) |
| 🗺 Draw | [`diagrams/messaging.md`](../diagrams/messaging.md) |
| ✅ Cheatsheet | [`cheatsheets/messaging.md`](../cheatsheets/messaging.md) |

**What the lab covers:** SNS → dual SQS fan-out · Visibility timeout expiry · Trigger DLQ after 3 failures · SNS message filtering

---

## Analytics
*Week 10 · Part 1*

| Step | File |
|---|---|
| 📖 Read | [`concepts/analytics.md`](../concepts/analytics.md) |
| 🗺 Draw | [`diagrams/analytics.md`](../diagrams/analytics.md) |

**Focus areas:** Athena partitioning (cost impact) · Glue crawler → Data Catalog · EMR node roles (core vs task) · Lake Formation permissions

---

## Security & Compliance
*Week 10 · Part 2 · Phase 5 checkpoint*

| Step | File |
|---|---|
| 📖 Read | [`concepts/security.md`](../concepts/security.md) |
| 🔬 Lab 1 | [`labs/kms-encryption.md`](../labs/kms-encryption.md) |
| 🔬 Lab 2 | [`labs/waf-shield.md`](../labs/waf-shield.md) |
| 🔬 Lab 3 | [`labs/guardduty-eventbridge.md`](../labs/guardduty-eventbridge.md) |
| 🗺 Draw | [`diagrams/security.md`](../diagrams/security.md) |
| ✅ Cheatsheet | [`cheatsheets/security.md`](../cheatsheets/security.md) |
| 📝 Practice Qs | [`mock-exams/phase5.md`](../mock-exams/phase5.md) · target ≥ 75% |

**Lab 1 covers:** Create CMK · SSE-KMS on S3 · Enforce via bucket policy · Read CloudTrail audit log · Secrets Manager vs SSM Parameter Store
**Lab 2 covers:** WAF Web ACL on ALB · IP block · Rate-based rule · AWS Managed Rules (OWASP) · SQL injection test · Shield Standard vs Advanced
**Lab 3 covers:** Enable GuardDuty · Generate sample findings · EventBridge rule on high-severity findings · Lambda auto-remediation (block attacker IP in SG) · Multi-account org pattern

---

---

# Phase 6 — Mock Exams & Weak-Area Review

*Weeks 11–12*

---

## Mock Exam #1
*Week 11*

| Step | File |
|---|---|
| 🧪 Exam | [`mock-exams/exam1.md`](../mock-exams/exam1.md) · 65 Qs · 130 min timed |
| 🐛 Log gaps | [`docs/bugbase.md`](bugbase.md) |
| 📖 Review | Relevant `concepts/` files for wrong answers |

**Target:** ≥ 72%

---

## Mock Exam #2
*Week 11*

| Step | File |
|---|---|
| 🧪 Exam | [`mock-exams/exam2.md`](../mock-exams/exam2.md) · 65 Qs · 130 min timed |
| 🐛 Log gaps | [`docs/bugbase.md`](bugbase.md) |
| 📖 Review | Targeted weak-domain review |

**Compare scores vs Exam #1 — identify trends**

---

## Cheatsheet Review + Cost & Architecture Patterns
*Week 12*

| Step | File |
|---|---|
| ✅ Review all | All `cheatsheets/*.md` |
| ✅ Cost focus | [`cheatsheets/cost.md`](../cheatsheets/cost.md) |
| 🗺 Draw | DR patterns from memory: Pilot Light · Warm Standby · Multi-Site |
| 📖 Read | AWS Well-Architected Framework — 6 pillars (docs.aws.amazon.com) |
| 🐛 Final gaps | [`docs/bugbase.md`](bugbase.md) |

---

## Mock Exam #3 (Official)
*Week 12*

| Step | File |
|---|---|
| 🧪 Exam | [`mock-exams/exam3.md`](../mock-exams/exam3.md) · 65 Qs · 130 min timed |
| 🐛 Log gaps | [`docs/bugbase.md`](bugbase.md) |

**Target: ≥ 80% — if yes, you're ready.**

---

---

# Phase 7 — Final Review & Exam

*Week 13*

---

## Final Prep

| Step | File |
|---|---|
| ✅ Light read | All `cheatsheets/*.md` — skim only, no new material |
| 🐛 Final pass | [`docs/bugbase.md`](bugbase.md) — known weak spots only |
| 🧠 Exam strategy | 2 min/question · flag & return · process of elimination |

**Exam day:** Arrive early · read each question fully before looking at options · trust your preparation.
