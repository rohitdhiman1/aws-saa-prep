# Messaging & Streaming — SQS, SNS, EventBridge, Kinesis, Step Functions

*Phase 5 · Week 9 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

AWS messaging services decouple application components. Choose based on: push vs pull, ordering, fan-out, real-time vs near-real-time, and throughput.

---

## How It Works

### SQS — Simple Queue Service

- **Pull-based** — consumers poll the queue.
- Stores messages until a consumer processes and deletes them (or they expire).
- Max message size: **256 KB**. Use SQS Extended Client + S3 for larger messages.
- Default message retention: **4 days** (max: 14 days).

#### Standard vs FIFO

| Feature | Standard | FIFO |
|---|---|---|
| Ordering | Best-effort (not guaranteed) | Strict FIFO within message group |
| Delivery | At-least-once (duplicates possible) | Exactly-once processing |
| Throughput | Unlimited | 300 msg/s (or 3,000 with batching) |
| Deduplication | No built-in | 5-min deduplication window |
| Use case | High throughput, order not critical | Financial transactions, order processing |

#### Key SQS Concepts

- **Visibility timeout** — after a consumer receives a message, it's hidden from other consumers for this period (default 30 s; max 12 hours). Consumer must delete the message before timeout or it reappears.
- **Dead Letter Queue (DLQ)** — messages that fail processing N times (`maxReceiveCount`) are moved here. Useful for debugging.
- **Long polling** — consumer waits up to 20 s for a message (`WaitTimeSeconds`). Reduces empty responses and costs.
- **Short polling** — returns immediately; may return empty; higher cost.
- **Delay queue** — delay delivery of all new messages by 0–900 s.
- **Message timer** — per-message delay (overrides queue delay).
- **Message group ID** — in FIFO queues; messages with same group ID are processed in order; different group IDs can be processed in parallel.

---

### SNS — Simple Notification Service

- **Push-based** — publishes to **topics**; delivers to all subscribers immediately.
- **Fan-out pattern** — one SNS topic → multiple SQS queues, Lambda, HTTP/HTTPS, email, SMS, mobile push.

#### Key SNS Concepts

| Feature | Details |
|---|---|
| **Subscribers** | SQS, Lambda, HTTP/S, Email, SMS, Mobile push |
| **Message filtering** | Each subscription can have a filter policy (JSON); only receives matching messages |
| **FIFO topics** | Ordered delivery; subscribers must be FIFO SQS queues; up to 300 msg/s |
| **Message retention** | No retention; if subscriber is unavailable, message is lost (except SQS subscribers) |
| **Large message payloads** | SNS Extended Client + S3 |
| **Cross-region delivery** | Yes — SNS topic in one region can deliver to SQS in another |

**Classic fan-out pattern:**
```
S3 Event → SNS Topic → [SQS Queue A (processing), SQS Queue B (audit), Lambda (real-time)]
```

---

### EventBridge

Event bus for event-driven architectures. Formerly CloudWatch Events.

- **Event buses:**
  - **Default bus** — receives AWS service events (e.g., EC2 state change, S3 PutObject).
  - **Custom bus** — your application events.
  - **Partner bus** — SaaS events (Zendesk, Datadog, etc.).
- **Rules** — match events on the bus using event patterns (JSON) → route to targets.
- **Targets** — Lambda, Step Functions, SQS, SNS, API Gateway, Kinesis, ECS, CodePipeline, and more.
- **EventBridge Scheduler** — schedule tasks (cron or rate expressions) without needing a running instance.
- **Schema registry** — auto-discovers event schemas; generates code bindings.
- **Pipes** — point-to-point integration between sources (SQS, DynamoDB Streams, Kinesis) and targets with optional filtering and enrichment.
- **Archive & replay** — archive events; replay to debug or reprocess.

**EventBridge vs SNS:**
- EventBridge: richer filtering (on any JSON field); schema registry; archive/replay; more targets.
- SNS: higher throughput; simpler; mobile push support.

---

### Kinesis Data Streams

Real-time streaming of data (logs, clickstreams, IoT telemetry).

- Data divided into **shards**. Each shard: 1 MB/s write, 2 MB/s read, up to 1,000 records/s.
- **Retention:** default 24 hours (up to 365 days with extended).
- Data in a shard is ordered; consumers read sequentially.

#### Consumers

| Type | Model | Throughput |
|---|---|---|
| **Standard (GetRecords)** | Pull; shared across consumers | 2 MB/s per shard shared |
| **Enhanced Fan-Out** | Push; dedicated throughput per consumer | 2 MB/s per shard per consumer |

- **Partition key** — determines which shard a record goes to. High cardinality = even distribution.
- **Sequence number** — unique ID for each record within a shard.

#### Kinesis Data Firehose

- Fully managed; **no consumers to write** — AWS delivers to destination automatically.
- Destinations: S3, Redshift (via S3), OpenSearch, Splunk, HTTP endpoint, Datadog.
- Optional **Lambda transformation** before delivery.
- Buffer: by time (60 s – 900 s) or size (1 MB – 128 MB).
- Near-real-time (not real-time); minimum ~60 s latency.

#### Kinesis Data Analytics (Managed Apache Flink)

- Run Apache Flink apps on streaming data from Kinesis Streams or MSK.
- SQL or Java/Python/Scala Flink apps.
- Real-time analytics, anomaly detection, windowing.

#### Kinesis vs SQS

| Feature | Kinesis Data Streams | SQS |
|---|---|---|
| Ordering | Per shard | FIFO only with FIFO queues |
| Replay | Yes (retention window) | No (deleted after consumption) |
| Multiple consumers | Yes (fan-out) | One consumer per message |
| Throughput | Provisioned (shards) | Unlimited (Standard) |
| Use case | Real-time analytics, log ingestion | Decoupled task processing |

---

### MSK — Managed Streaming for Apache Kafka

- Fully managed Apache Kafka — for teams that already know Kafka or need Kafka-specific features.
- Supports Kafka APIs, Connect, Streams.
- Multi-AZ with replication; automated broker replacement.
- **MSK Serverless** — no capacity planning; auto-scales.
- **MSK Connect** — run Kafka Connect connectors (sink/source) without managing workers.

**MSK vs Kinesis:**
- Kinesis: AWS-native, simpler, tight AWS integration. MSK: Kafka ecosystem, open-source, more config flexibility.

---

### API Gateway

Managed service to create, publish, and manage REST, HTTP, and WebSocket APIs.

| Type | Use case |
|---|---|
| **REST API** | Full features (caching, request validation, usage plans, API keys, WAF) |
| **HTTP API** | Lower cost, lower latency; OIDC/JWT auth; no usage plans |
| **WebSocket API** | Persistent bidirectional connections |

- **Stages** — deployment environments (dev, prod). Stage variables for per-stage config.
- **Authorizers** — Lambda authorizer (custom logic) or Cognito User Pool authorizer (JWT).
- **Throttling** — default 10,000 RPS per account; 5,000 burst. Per-stage/method limits available.
- **Caching** — response cache per stage; TTL 0–3600 s.
- **Edge-optimised** — deployed via CloudFront (default); **regional** — same region as client; **private** — accessible only within VPC via interface endpoint.

---

### Step Functions

Serverless orchestration of workflows using **state machines**.

| Type | Duration | Use case | Pricing |
|---|---|---|---|
| **Standard** | Up to 1 year | Long-running, exactly-once, human approval | Per state transition |
| **Express** | Up to 5 minutes | High-volume, event processing, at-least-once | Per execution duration |

- States: Task, Choice, Parallel, Map, Wait, Pass, Succeed, Fail.
- **Task state** — integrates with Lambda, ECS, DynamoDB, SQS, SNS, API Gateway, and more via SDK integrations.
- **Optimistic locking / retries** — built-in retry and catch logic per state.

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| SQS max message size | 256 KB |
| SQS max retention | 14 days |
| SQS FIFO throughput | 300 msg/s (3,000 with batching) |
| SQS visibility timeout max | 12 hours |
| SNS max message size | 256 KB |
| Kinesis shard write | 1 MB/s or 1,000 records/s |
| Kinesis shard read | 2 MB/s |
| Kinesis max retention | 365 days |
| Kinesis Firehose min buffer | 60 seconds |
| API Gateway REST default throttle | 10,000 RPS / 5,000 burst |
| Step Functions Standard max duration | 1 year |

---

## When To Use

| Scenario | Solution |
|---|---|
| Decouple microservices, async | SQS Standard |
| Order matters, no duplicates | SQS FIFO |
| Fan-out to multiple services | SNS → SQS (fan-out) |
| Event-driven from AWS services | EventBridge (default bus) |
| Schedule tasks (cron) | EventBridge Scheduler |
| Real-time log/clickstream ingestion | Kinesis Data Streams |
| Load data to S3/Redshift near-real-time | Kinesis Firehose |
| Kafka workloads | MSK |
| Orchestrate multi-step workflows | Step Functions |
| Serverless API | API Gateway + Lambda |

---

## Exam Traps

- **SQS visibility timeout ≠ message expiry** — visibility timeout hides the message temporarily; it reappears if not deleted.
- **FIFO SQS throughput is limited** — 300/s (or 3,000 with batching); not unlimited like Standard.
- **Kinesis Firehose is near-real-time** — minimum 60 s buffer; not instant. Use Kinesis Streams for sub-second.
- **SNS does not retain messages** — if a subscriber is down (not SQS), the message is lost. SQS subscriber is more durable.
- **EventBridge default bus ≠ custom bus** — you cannot send custom app events to the default bus directly without a rule.
- **Kinesis shard hot partition** — bad partition key choice causes uneven load. Choose high-cardinality keys.
- **Step Functions Express = at-least-once** — Standard = exactly-once; don't use Express for idempotency-sensitive workflows.
- **API Gateway + Lambda integration** — Lambda proxy integration passes the full request; non-proxy requires mapping templates.
- **SQS DLQ must be same type** — Standard DLQ for Standard queue; FIFO DLQ for FIFO queue.
- **Long polling reduces cost** — use `WaitTimeSeconds=20` to avoid empty polls and reduce API costs.

---

## Related Files

| File | Purpose |
|---|---|
| `diagrams/messaging.md` | Visual: fan-out, Kinesis pipeline, Step Functions workflow |
| `cheatsheets/messaging.md` | Quick-reference card |
| `mock-exams/phase5.md` | Messaging + analytics + security practice questions |
| `concepts/analytics.md` | Next concept file (Week 10) |
| `concepts/security.md` | Week 10 (part 2) |
