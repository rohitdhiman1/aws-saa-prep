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

#### Concurrency

- **Soft limit:** 1,000 concurrent executions per account per region (can increase).
- **Reserved concurrency** — cap a function's max concurrency (also guarantees capacity).
- **Provisioned concurrency** — pre-warm execution environments; eliminates cold starts.

#### Other Key Concepts

- **Layers** — shared code/deps zipped and reused across functions (up to 5 layers).
- **Lambda@Edge / CloudFront Functions** — run code at CloudFront edge. Lambda@Edge: full runtime, 30 s timeout. CloudFront Functions: JS only, sub-ms, cheaper.
- **Destinations** — on success or failure, route async invocation result to SQS, SNS, EventBridge, or another Lambda.
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
