# Diagrams: Compute

*Draw these on paper or in draw.io as part of Week 3 study.*

---

## Diagram 1: EC2 + ASG + ALB Architecture

```
                        Internet
                           │
                    ┌──────┴──────┐
                    │  ALB        │
                    │  (public)   │
                    └──────┬──────┘
                           │ (target group)
             ┌─────────────┴──────────────┐
             │                            │
      ┌──────┴──────┐              ┌──────┴──────┐
      │  AZ-a       │              │  AZ-b       │
      │  Private    │              │  Private    │
      │  Subnet     │              │  Subnet     │
      │  ┌───────┐  │              │  ┌───────┐  │
      │  │ EC2-1 │  │              │  │ EC2-2 │  │
      │  └───────┘  │              │  └───────┘  │
      └─────────────┘              └─────────────┘
             └──────────────────────────────────┘
                          Auto Scaling Group
                      min=1  desired=2  max=4
```

---

## Diagram 2: ECS Fargate Architecture

```
┌───────────────────────────────────────────────┐
│  ECS Cluster                                  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │  ECS Service (desired: 2 tasks)         │  │
│  │                                         │  │
│  │  ┌────────────────┐  ┌────────────────┐ │  │
│  │  │  Task (Fargate)│  │  Task (Fargate)│ │  │
│  │  │  Container A   │  │  Container A   │ │  │
│  │  │  Container B   │  │  Container B   │ │  │
│  │  └───────┬────────┘  └───────┬────────┘ │  │
│  └──────────┼───────────────────┼──────────┘  │
└─────────────┼───────────────────┼─────────────┘
              └─────────┬─────────┘
                    ALB Target Group
                        │
                       ALB
```

**Two IAM roles for ECS:**
- **Task execution role** → ECS agent (pull image from ECR, write logs to CloudWatch)
- **Task role** → Container code (access S3, DynamoDB, etc.)

---

## Diagram 3: Lambda Invocation Models

```
SYNCHRONOUS:
Client ──request──► API GW ──► Lambda ──► Response ──► Client
(client waits; handles errors itself)

ASYNCHRONOUS:
S3 event ──► Lambda (internal queue) ──► Lambda execution
                                              │ fails? retry 2x → DLQ/Destination

EVENT SOURCE MAPPING (poll-based):
Lambda ──polls──► SQS / Kinesis / DynamoDB Streams
              ◄── batch of records ──
              processes, deletes from queue
```

---

## Diagram 4: EC2 Purchasing Options — Decision Tree

```
Is workload predictable and running > 1 year?
  ├── YES → Reserved Instance or Savings Plans
  │         (Convertible RI for flexibility)
  └── NO  → Is it fault-tolerant / can tolerate interruption?
              ├── YES → Spot Instance (up to 90% discount)
              └── NO  → On-Demand
                         └── Needs dedicated hardware?
                               ├── YES (licensing/compliance) → Dedicated Host
                               └── YES (single-tenant, no socket control) → Dedicated Instance
```
