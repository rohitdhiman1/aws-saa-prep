# Compute — EC2, Auto Scaling, Lambda, Containers

*Phase 2 · Week 3 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

AWS compute covers the full spectrum from raw VMs (EC2) to fully managed serverless (Lambda), with containers in between (ECS, EKS) and a PaaS abstraction on top (Elastic Beanstalk).

---

## How It Works

### EC2 — Elastic Compute Cloud

#### Instance Families

| Family | Optimised for | Example sizes |
|---|---|---|
| `t` (T3, T4g) | Burstable general purpose | t3.micro, t3.large |
| `m` (M6i, M7g) | Balanced general purpose | m6i.xlarge |
| `c` (C6i, C7g) | Compute-intensive (CPU) | c6i.2xlarge |
| `r` (R6i, R7g) | Memory-intensive | r6i.4xlarge |
| `x` (X2idn) | High memory (in-memory DBs) | x2idn.16xlarge |
| `p` / `g` / `inf` | GPU / ML inference | p3.2xlarge |
| `i` / `d` | Storage-optimised (NVMe SSD) | i4i.xlarge |

#### Purchasing Options

| Option | When to use | Discount |
|---|---|---|
| **On-Demand** | Unpredictable, short-term | Baseline price |
| **Reserved (1 or 3 yr)** | Steady-state, predictable | Up to 72% |
| **Savings Plans (Compute)** | Flexible (any instance family/region) | Up to 66% |
| **Savings Plans (EC2)** | Specific family + region | Up to 72% |
| **Spot** | Fault-tolerant, batch, flexible | Up to 90% |
| **Dedicated Host** | Licensing (BYOL), compliance | Most expensive |
| **Dedicated Instance** | Single-tenant hardware, no socket-level control | — |
| **Capacity Reservation** | Reserve capacity in AZ, pay On-Demand rate | — |

> **Spot interruption:** AWS gives 2-minute warning. Use Spot with interruption handlers or Spot Fleet/ASG.

#### AMIs, User Data & Metadata

- **AMI** — snapshot of an EBS root volume + launch permissions. Region-specific; copy across regions.
- **User data** — bash script run at first launch (as root). Used for bootstrapping software.
- **Instance metadata** — accessible at `http://169.254.169.254/latest/meta-data/`. Provides instance ID, IAM role creds, public IP, etc. IMDSv2 is preferred (requires session token).

#### Placement Groups

| Type | Behaviour | Use case |
|---|---|---|
| **Cluster** | Same rack, same AZ | Ultra-low latency (HPC, big data) |
| **Spread** | Different racks, up to 7 per AZ | Critical instances needing isolation |
| **Partition** | Groups of instances on separate partitions | Large distributed systems (Kafka, HDFS) |

---

### Auto Scaling

- **Launch Template** (preferred over Launch Configuration) — defines AMI, instance type, security groups, IAM role, user data.
- **Auto Scaling Group (ASG)** — manages a fleet across AZs. Min, desired, max capacity.

#### Scaling Policies

| Policy | Trigger | Use case |
|---|---|---|
| **Target Tracking** | Keep a metric at a target (e.g. CPU = 50%) | Most common; AWS manages scale in/out |
| **Step Scaling** | Scale by N instances when metric crosses threshold | Fine-grained control |
| **Scheduled** | Scale at a specific time | Predictable load (e.g. 9am rush) |
| **Predictive** | ML-based, scale ahead of time | Cyclical load patterns |

**Cooldown period** — default 300 seconds after a scaling action; prevents rapid thrashing.

**Lifecycle hooks** — pause instances at `Pending:Wait` or `Terminating:Wait` to run custom scripts.

---

### Load Balancers (ELB)

| Type | Layer | Use case |
|---|---|---|
| **ALB** (Application) | L7 (HTTP/HTTPS) | Path-based routing, host-based routing, gRPC, WebSocket, Lambda targets |
| **NLB** (Network) | L4 (TCP/UDP/TLS) | Ultra-low latency, static IP, PrivateLink |
| **GLB** (Gateway) | L3 (IP) | Inline traffic inspection (firewalls, IDS/IPS) |
| **CLB** (Classic) | L4/L7 | Legacy; avoid for new workloads |

**ALB key concepts:**
- **Listeners** — port + protocol rule; must have at least one.
- **Target groups** — EC2 instances, IPs, Lambda functions, or other ALBs.
- **Rules** — path (`/api/*`), host (`api.example.com`), header conditions → forward, redirect, return fixed response.
- **Sticky sessions** — duration-based or application-based cookies.

**NLB key concepts:**
- Preserves source IP; ALB does not (use `X-Forwarded-For`).
- Supports Elastic IPs → useful for whitelisting.
- Health checks: TCP, HTTP, HTTPS.

**Cross-zone load balancing:**
- ALB: on by default (no cost).
- NLB/GLB: off by default; enabling incurs cross-AZ data transfer costs.

---

### Lambda

- **Serverless** — no servers to manage; pay per invocation + GB-seconds.
- Max **execution duration:** 15 minutes.
- Memory: 128 MB – 10,240 MB (CPU scales with memory).
- Ephemeral storage: 512 MB – 10 GB (`/tmp`).

#### Invocation Models

| Model | How triggered | Retry behaviour |
|---|---|---|
| **Synchronous** | API Gateway, SDK direct call | Caller handles retries |
| **Asynchronous** | S3 events, SNS, EventBridge | Lambda retries up to 2 times; use DLQ or destination |
| **Event source mapping** | SQS, Kinesis, DynamoDB Streams | Lambda polls; retries until success or item expires |

#### Concurrency (Deep Dive)

- **Soft limit:** 1,000 concurrent executions per account per region (can increase).
- **Unreserved concurrency** — the pool shared by all functions without reserved concurrency. Always at least 100 are kept unreserved.

| Type | What it does | Cold starts? | Cost |
|---|---|---|---|
| **Reserved concurrency** | Caps max AND guarantees that capacity for this function. Other functions cannot consume it. | Still possible | Free (just a config) |
| **Provisioned concurrency** | Pre-warms N execution environments so they're always ready | Eliminated (up to N) | Extra charge per pre-warmed environment |
| **Neither** | Function uses shared unreserved pool | Yes | No extra cost |

**When to use each:**
- **Reserved** — prevent a high-traffic function from starving others (or vice versa). Example: "limit function X to 100 concurrent so it doesn't overwhelm downstream DB."
- **Provisioned** — latency-sensitive workloads with predictable spikes. Example: "app has traffic spikes at 9 AM; pre-warm 200 instances." Can be combined with Application Auto Scaling to schedule provisioned concurrency.
- **Reserved = 0** — effectively disables the function (all invocations are throttled).

#### Event Source Mappings (Deep Dive)

Lambda **polls** the source and invokes your function with a batch of records. Applies to: SQS, Kinesis, DynamoDB Streams, MSK, self-managed Kafka, MQ.

| Config | What it controls | Default |
|---|---|---|
| **Batch size** | Max records per invocation | 10 (SQS), 100 (Kinesis/DynamoDB) |
| **Batch window** | Max seconds to wait before invoking (even if batch not full) | 0 (invoke immediately) |
| **Parallelization factor** | Concurrent batches per shard (Kinesis/DynamoDB only) | 1 (max 10) |
| **Maximum record age** | Discard records older than this (Kinesis/DynamoDB) | -1 (never discard) |
| **Maximum retry attempts** | Stop retrying a failed batch after N attempts | -1 (infinite) |
| **Bisect batch on error** | On failure, split batch in half and retry each half (Kinesis/DynamoDB) | Disabled |
| **Destination on failure** | Send failed records to SQS or SNS | None |

**Key exam patterns:**
- "Lambda processing Kinesis keeps failing on one bad record, blocking the shard" → enable **bisect batch on error** + set **maximum retry attempts** or **maximum record age** to unblock.
- "Increase Lambda throughput from Kinesis stream without adding shards" → increase **parallelization factor** (up to 10 concurrent batches per shard).

**SQS-specific behaviour:**
- Lambda polls SQS and processes messages in batches (up to 10 for standard, up to 10 for FIFO).
- On success: Lambda deletes the messages from the queue automatically.
- On failure: messages return to the queue after **visibility timeout** expires → will be reprocessed.
- **Partial batch response** (`ReportBatchItemFailures`) — return only the failed message IDs; Lambda deletes the successful ones and retries only failures. Without this, the entire batch is retried.
- Set visibility timeout to **at least 6x the function timeout** to avoid duplicate processing while Lambda is still running.
- After `maxReceiveCount` failures → messages go to the **DLQ** (configured on the SQS queue, not on Lambda).

#### Destinations vs DLQ

Both handle async invocation failures, but destinations are more powerful:

| Feature | DLQ | Destinations |
|---|---|---|
| Triggers on | Failure only | **Success AND failure** |
| Targets | SQS, SNS | SQS, SNS, EventBridge, Lambda |
| Payload | Original event only | Full invocation record (event + response + error + context) |
| Recommendation | Legacy; still works | **Preferred** — more flexible, richer data |

> **Exam tip:** If the question says "route failed async invocations to an SQS queue for analysis" → Destinations (preferred) or DLQ (both work, but destinations give more context).

#### Versions & Aliases

- **Version** — immutable snapshot of function code + configuration. `$LATEST` is the only mutable version.
- **Alias** — named pointer to a version (e.g., `prod` → version 5, `staging` → version 7).

**Traffic shifting with aliases:**
- An alias can split traffic between **two versions** using weighted routing.
- Example: `prod` alias → 90% to v5 + 10% to v6 (**canary deployment**).
- Integrates with **CodeDeploy** for automated canary/linear deployment strategies:
  - `Canary10Percent5Minutes` — shift 10% for 5 min, then 100%.
  - `Linear10PercentEvery1Minute` — add 10% every minute.
  - `AllAtOnce` — immediate shift.

**Why this matters for SAA:**
- "Gradually shift traffic to a new Lambda version" → alias with weighted routing.
- "API Gateway should invoke the stable Lambda version" → point API Gateway stage variable at an alias, not `$LATEST`.

#### Other Key Concepts

- **Layers** — shared code/deps zipped and reused across functions (up to 5 layers).
- **Lambda@Edge / CloudFront Functions** — run code at CloudFront edge. Lambda@Edge: full runtime, 30 s timeout. CloudFront Functions: JS only, sub-ms, cheaper.
- **VPC Lambda** — attach to a VPC subnet; needs a NAT Gateway for internet access; adds cold start latency.

---

### ECS vs EKS

| Feature | ECS | EKS |
|---|---|---|
| Orchestrator | AWS proprietary | Kubernetes |
| Learning curve | Lower | Higher (K8s knowledge needed) |
| Control plane | Managed by AWS (free with Fargate) | Managed K8s control plane ($0.10/hr) |
| Launch types | EC2 or Fargate | EC2 or Fargate |
| Use case | New containerised apps on AWS | Existing K8s workloads or K8s expertise |

**Fargate** — serverless container runtime; no EC2 to manage. Works with both ECS and EKS.
Define CPU + memory at task level; AWS schedules and runs the containers.

**ECS concepts:**
- **Task definition** — blueprint: container image, CPU, memory, networking, IAM role.
- **Task** — running instance of a task definition.
- **Service** — maintains N running tasks; integrates with ALB.
- **Cluster** — logical grouping of tasks/services.

**IAM for ECS:**
- **Task execution role** — ECS agent pulls image, sends logs to CloudWatch.
- **Task role** — the running container's permissions (e.g. S3, DynamoDB access).

---

### Elastic Beanstalk

PaaS layer over EC2, ASG, ALB, RDS. You upload code; Beanstalk handles the rest.

| Deployment mode | Description | Downtime? |
|---|---|---|
| All at once | Deploy to all instances simultaneously | Yes (briefly) |
| Rolling | Deploy to batches; reduces capacity during deploy | No |
| Rolling with additional batch | Spin up new batch first | No; capacity maintained |
| Immutable | Launch new ASG with new version; swap on success | No; safest rollback |
| Blue/Green | Deploy new environment; swap CNAME | No; full environment swap |

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| Lambda max duration | 15 minutes |
| Lambda max memory | 10,240 MB |
| Lambda max deployment package (zip) | 50 MB (compressed), 250 MB unzipped |
| Lambda max layers | 5 |
| Lambda concurrency default | 1,000 per region |
| ASG max instances | 2,500 per ASG |
| ALB target groups per rule | 5 |
| ECS tasks per service | 5,000 (Fargate) |
| EC2 Spot interruption notice | 2 minutes |
| Placement group (Spread) | 7 instances per AZ |

---

## When To Use

| Scenario | Solution |
|---|---|
| Unpredictable workload, short jobs | On-Demand or Spot |
| 24/7 steady workload for 3 years | Reserved Instances or Savings Plans |
| Fault-tolerant batch processing | Spot Fleet |
| Ultra-low latency between instances | Cluster placement group |
| HA critical instances on separate hardware | Spread placement group |
| Path-based HTTP routing | ALB with listener rules |
| Fixed IP for load balancer | NLB with Elastic IP |
| Event-driven, short tasks | Lambda |
| Containerised app, minimal ops overhead | ECS + Fargate |
| Existing Kubernetes workloads | EKS |
| Full app stack with minimal config | Elastic Beanstalk |

---

## Exam Traps

- **Spot interruption = 2 min notice** — not instant; use interruption handler or Spot Fleet.
- **Reserved Instances apply to running instances** — if you change instance type/family, RI may not apply.
- **Lambda is NOT always cheaper** — long-running jobs are cheaper on EC2.
- **Lambda in VPC = cold start** — ENI creation adds latency; use Provisioned Concurrency if latency-sensitive.
- **Reserved concurrency = 0 disables the function** — all invocations throttled. Useful for emergency kill switch.
- **SQS visibility timeout must be ≥ 6x Lambda timeout** — otherwise messages reappear while Lambda is still processing them.
- **Bisect batch on error** — key for Kinesis/DynamoDB Streams when a poison record blocks the shard.
- **Destinations ≠ DLQ** — destinations handle success AND failure; DLQ is failure only. Destinations are preferred.
- **$LATEST is mutable** — never point production at $LATEST. Use an alias pointing to a specific version.
- **Partial batch response** — without `ReportBatchItemFailures`, one failed SQS message causes the entire batch to retry.
- **ALB vs NLB for static IP** — only NLB supports Elastic IPs; ALB IPs are dynamic.
- **Target tracking cooldown** — default 300s; scale-in uses longer cooldown than scale-out.
- **ECS task role ≠ execution role** — execution role is for ECS infrastructure; task role is for your app code.
- **Fargate does not support privileged containers** — some workloads requiring host access must use EC2 launch type.
- **Beanstalk Immutable is safest rollback** — Blue/Green requires DNS swap which can have propagation delay.
- **Cross-zone load balancing cost** — enabled by default on ALB (free); extra cost on NLB.

---

## Related Files

| File | Purpose |
|---|---|
| `labs/compute-asg-alb.md` | Hands-on: ASG behind ALB with target tracking |
| `diagrams/compute.md` | Visual: EC2 → ASG → ALB, ECS Fargate architecture |
| `cheatsheets/compute.md` | Quick-reference card |
| `mock-exams/phase2.md` | Compute + storage practice questions |
| `concepts/storage.md` | Next concept file (Week 4) |
