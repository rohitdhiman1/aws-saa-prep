# AWS Solutions Architect – Associate (SAA-C03) Study Repo

Personal study repository for the **AWS Certified Solutions Architect – Associate (SAA-C03)** exam.

---

## Exam At a Glance

| Field | Detail |
|---|---|
| Cert | AWS Certified Solutions Architect – Associate |
| Code | SAA-C03 |
| Cost | $150 USD |
| Format | 65 questions · 130 minutes · multiple choice + multi-select |
| Passing score | 720 / 1000 |
| Booking | [aws.training](https://aws.training) |
| Target date | **May 2026** |

---

## Domain Breakdown (official SAA-C03 weights)

| Domain | Weight |
|---|---|
| 1. Design Secure Architectures | 30% |
| 2. Design Resilient Architectures | 26% |
| 3. Design High-Performing Architectures | 24% |
| 4. Design Cost-Optimized Architectures | 20% |

---

## Topics

- **IAM** — users, roles, policies, STS, identity federation, Organizations
- **Compute** — EC2 (types, placement, pricing), Auto Scaling, Lambda, ECS, EKS, Elastic Beanstalk
- **Storage** — S3 (storage classes, lifecycle, replication, encryption), EBS, EFS, FSx, Storage Gateway
- **Databases** — RDS, Aurora, DynamoDB, ElastiCache, Redshift, Neptune
- **Networking** — VPC, subnets, NACLs, Security Groups, NAT, routing tables, VPN, Direct Connect, Transit Gateway, Route 53, CloudFront
- **Messaging** — SQS, SNS, EventBridge, Kinesis (Streams, Firehose, Analytics)
- **Data & Analytics** — EMR, Glue, Athena, MSK, Lake Formation, OpenSearch
- **Security** — KMS, Secrets Manager, WAF, Shield, GuardDuty, Inspector, Macie
- **Architecture Patterns** — HA, fault tolerance, disaster recovery (RTO/RPO), cost optimisation, Well-Architected Framework

---

## Repo Structure

```
aws-saa-prep/
├── README.md                     # This file — project overview & guide
├── docs/
│   ├── roadmap.md                # Day-by-day schedule (Mar 1 – May 31) with file references
│   ├── next-session.md           # What to study in the next session
│   └── bugbase.md                # Knowledge gaps and weak areas tracker
├── concepts/                     # PRIMARY study files — one per service/topic
│   ├── iam.md
│   ├── vpc-basics.md
│   ├── compute.md
│   ├── storage.md
│   ├── databases-relational.md
│   ├── databases-nosql.md
│   ├── networking-advanced.md
│   ├── edge-dns.md
│   ├── messaging.md
│   ├── analytics.md
│   └── security.md
├── labs/                         # Hands-on exercises (step-by-step)
│   ├── iam-sandbox.md            # W1 Sat Mar 7
│   ├── vpc-build.md              # W2 Sat Mar 14
│   ├── compute-asg-alb.md        # W3 Sat Mar 21
│   ├── lambda-events.md          # W3/W4 Sun Mar 22
│   ├── s3-versioning-replication.md # W4 Sat Mar 28
│   ├── rds-multiaz-replica.md    # W5 Sat Apr 4
│   ├── dynamodb-table-gsi.md     # W6 Sat Apr 11
│   ├── vpc-peering.md            # W7 Sat Apr 18
│   ├── cloudfront-route53.md     # W8 Sat Apr 25
│   ├── sqs-sns-fanout.md         # W9 Sat May 2
│   └── kms-encryption.md         # W10 Sat May 9
├── diagrams/                     # ASCII architecture diagrams (draw on paper too)
│   ├── iam.md · vpc-basics.md · compute.md · storage.md
│   ├── databases.md · networking.md · messaging.md
│   ├── security.md · analytics.md
├── cheatsheets/                  # Quick-reference cards (fill in after each phase)
│   ├── iam.md · vpc-basics.md · compute.md · storage.md
│   ├── databases.md · networking.md · messaging.md
│   ├── security.md · cost.md
└── mock-exams/                   # Practice question sets + full timed exams
    ├── phase1.md … phase5.md     # Per-phase practice questions
    └── exam1.md · exam2.md · exam3.md  # Full 65-question timed exams
```

### Folder purposes

| Folder | Purpose | When to use |
|---|---|---|
| `docs/` | Planning, tracking, progress | Open `roadmap.md` at the start of every session |
| `concepts/` | Core theory — structured, exam-focused | Primary reading for each topic |
| `labs/` | Hands-on tasks in AWS console/sandbox | Weekly lab day |
| `diagrams/` | Architecture visuals to study and reproduce | Draw on paper; review before exams |
| `cheatsheets/` | Condensed exam-day reference cards | Fill after each phase; re-read in Week 11–13 |
| `mock-exams/` | Scored practice and full-length exams | End of each phase + Weeks 11–12 |

### Concept file template (every `concepts/*.md` follows this structure)

```
## What It Is     → brief definition
## How It Works   → mechanisms, sub-services, key flows
## Key Configs & Limits → numbers the exam tests
## When To Use    → decision criteria
## Exam Traps     → common wrong answers
## Related Files  → links to lab, diagram, cheatsheet, mock-exam
```

---

## How to Use This Repo (Daily Workflow)

1. **Open `docs/roadmap.md`** — find today's date row. It tells you exactly: Read → Do → Draw → Summarise.
2. **Read the concept file** — `concepts/<service>.md` for today's topic.
3. **Do the lab** (on lab days) — `labs/<lab>.md` in ACG sandbox or AWS console.
4. **Draw the diagram** — `diagrams/<topic>.md` has ASCII versions; redraw on paper from memory.
5. **Fill the cheatsheet** — `cheatsheets/<topic>.md` at end of each phase.
6. **Log gaps** — anything confusing goes in `docs/bugbase.md`.
7. **Update next session** — `docs/next-session.md` at end of each session.

---

## Study Resources

| Resource | Notes |
|---|---|
| [A Cloud Guru (ACG)](https://acloudguru.com/) | Primary resource for hands-on labs using their 4-hour AWS Sandbox environments safely without personal billing |
| [Adrian Cantrill — cantrill.io](https://cantrill.io) | Recommended primary course; very visual and thorough |
| [Stephane Maarek — Udemy](https://www.udemy.com) | Good supplementary course; concise and exam-focused |
| [AWS Official Docs](https://docs.aws.amazon.com) | Ground truth for any nuance or edge case |
| [AWS Skill Builder](https://skillbuilder.aws) | Official practice exams and question sets |
| [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/) | Must-read for architecture domain questions |

---

## Progress Tracker

Update as phases are completed.

| Phase | Topic Focus | Due | Status |
|---|---|---|---|
| 1 | IAM & Core Networking foundations | Mar 14 | In progress |
| 2 | Compute & Storage | Mar 28 | Not started |
| 3 | Databases & Caching | Apr 11 | Not started |
| 4 | Networking deep dive | Apr 25 | Not started |
| 5 | Messaging, Analytics & Security | May 9 | Not started |
| 6 | Practice exams & weak-area review | May 23 | Not started |
| 7 | Final review & exam prep | May 27–29 | Not started |

---

## Useful Commands

```bash
# Find notes on a specific service
grep -ri "S3" notes/

# List all cheatsheets
ls cheatsheets/

# Count mock exam questions in a set
grep -c "^##" mock-exams/<set-name>.md
```
