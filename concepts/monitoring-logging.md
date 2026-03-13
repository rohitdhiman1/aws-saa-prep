# Monitoring & Logging — CloudWatch, CloudTrail, VPC Flow Logs, X-Ray

*Phase 5 · Week 10 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

AWS observability stack covering: metrics & alarms (CloudWatch), API audit logging (CloudTrail), network traffic capture (VPC Flow Logs), and distributed tracing (X-Ray). These services appear across all four SAA-C03 exam domains.

---

## How It Works

### CloudWatch — Metrics, Alarms & Logs

Central monitoring service. Collects metrics from nearly every AWS service automatically.

#### Metrics

- **Namespace** — container for metrics (e.g., `AWS/EC2`, `AWS/RDS`).
- **Dimensions** — key-value pairs that identify a metric (e.g., `InstanceId=i-12345`).
- **Default metrics** — free, 5-minute granularity (EC2), 1-minute (ELB, RDS).
- **Detailed monitoring** — EC2: 1-minute granularity ($3.50/instance/month).
- **Custom metrics** — push with `PutMetricData` API. Standard resolution: 1-minute. High resolution: up to 1-second.
- **Metric retention:**
  - 1-second data → 3 hours
  - 1-minute data → 15 days
  - 5-minute data → 63 days
  - 1-hour data → 455 days (15 months)

#### Important: EC2 Metrics NOT Available by Default

CloudWatch does **not** collect memory or disk-level metrics from EC2. You must install the **CloudWatch Agent** (unified agent) to collect:
- Memory utilisation
- Disk swap, usage, inodes
- Custom application logs

> **Exam trap:** "Monitor memory usage" → answer is CloudWatch Agent, not default CloudWatch.

#### Alarms

- **States:** OK → INSUFFICIENT_DATA → ALARM
- **Actions on ALARM:** SNS notification, Auto Scaling action, EC2 action (stop/terminate/reboot/recover).
- **Composite alarms** — combine multiple alarms with AND/OR logic to reduce alarm noise.
- **Period** — evaluation period (e.g., 5 minutes). Data points to alarm: e.g., "3 out of 5 data points breaching."
- **EC2 instance recovery** — alarm action `ec2:RecoverInstances`: moves instance to new host, same private IP, EIP, metadata, placement group.

#### Logs

- **Log Groups** — collection of log streams (e.g., `/aws/lambda/my-function`).
- **Log Streams** — sequence of events from a single source.
- **Retention** — configurable: 1 day to 10 years, or never expire.
- **Metric Filters** — extract metrics from log data (e.g., count occurrences of "ERROR"). Can trigger alarms.
- **Subscription Filters** — real-time stream of log events to:
  - Lambda (processing)
  - Kinesis Data Streams / Firehose (analytics)
  - OpenSearch (search/visualisation)
- **Cross-account log sharing** — subscription filter in source account → Kinesis in destination account.
- **Logs Insights** — query language for ad-hoc log analysis. Serverless, pay-per-query.
- **Log export to S3** — batch export (not real-time; takes up to 12 hours). Use subscription filter + Firehose for near-real-time.

> **Exam trap:** "Real-time log delivery to S3" → Subscription Filter → Kinesis Firehose → S3 (NOT CreateExportTask).

#### Dashboards

- Custom dashboards with metrics from multiple regions and accounts.
- Up to 3 dashboards (50 metrics each) free. Then $3/dashboard/month.
- **Cross-account dashboards** — view metrics from multiple accounts in one dashboard (requires cross-account access).

#### EventBridge (formerly CloudWatch Events)

- See `concepts/messaging.md` for full coverage.
- Key point: EventBridge is the successor to CloudWatch Events. Same underlying service, expanded with partner/custom buses.

---

### CloudTrail — API Audit Logging

Records **every API call** made in your AWS account (console, CLI, SDK, service-to-service).

#### Trail Types

| Type | Scope | Cost |
|---|---|---|
| **Event history** (default) | 90 days, management events, free, no setup | Free |
| **Trail** (single-region) | Logs to S3 bucket, configurable | First trail free (management events) |
| **Trail** (all-regions) | Best practice — one trail for all regions | First trail free |
| **Organisation trail** | All accounts in AWS Organisations | First trail free |

#### Event Types

| Type | What | Default | Cost |
|---|---|---|---|
| **Management events** | Control plane (CreateBucket, RunInstances, AttachPolicy) | On by default | First copy free |
| **Data events** | Data plane (GetObject, PutObject, Invoke Lambda) | Off by default | $0.10 per 100,000 events |
| **Insights events** | Unusual API activity detection | Off by default | $0.35 per 100,000 events analysed |

#### Key Features

- **S3 delivery** — logs delivered to S3 bucket (gzip JSON). ~5 minutes delivery lag.
- **CloudWatch Logs integration** — stream trail events to CloudWatch Logs for metric filters and alarms.
- **SNS notifications** — notify on every log file delivery.
- **Log file integrity validation** — SHA-256 hash chain to detect tampering.
- **Organisation trail** — single trail for all member accounts; logs to management account S3.
- **Event selectors** — filter which events are logged (e.g., only S3 data events for specific buckets).

#### Common Exam Patterns

- "Who deleted the S3 bucket?" → CloudTrail management events.
- "Real-time alert on root login" → CloudTrail → CloudWatch Logs → Metric Filter → Alarm → SNS.
- "Prove logs haven't been tampered with" → CloudTrail log file integrity validation.
- "All API calls across all accounts" → Organisation trail (all-regions).

---

### VPC Flow Logs

Capture IP traffic information for network interfaces in your VPC.

#### Capture Levels

| Level | What it captures |
|---|---|
| **VPC** | All ENIs in the VPC |
| **Subnet** | All ENIs in the subnet |
| **ENI** | Single network interface |

#### Flow Log Record (Default Fields)

```
<version> <account-id> <interface-id> <srcaddr> <dstaddr> <srcport> <dstport>
<protocol> <packets> <bytes> <start> <end> <action> <log-status>
```

- **action** — `ACCEPT` or `REJECT` (based on SG/NACL evaluation).
- **Custom fields** — add `vpc-id`, `subnet-id`, `instance-id`, `type`, `pkt-srcaddr`, `pkt-dstaddr`, `tcp-flags`, `traffic-path`.

#### Destinations

| Destination | Use case |
|---|---|
| **CloudWatch Logs** | Metric filters, alarms, Logs Insights queries |
| **S3** | Long-term storage, Athena queries, cost-effective |
| **Kinesis Firehose** | Near-real-time analytics |

#### What Flow Logs Do NOT Capture

- DNS queries to Route 53 Resolver (use DNS query logging separately).
- DHCP traffic.
- Traffic to the instance metadata service (169.254.169.254).
- Traffic to the VPC DNS server (AmazonProvidedDNS).
- Windows license activation traffic.

#### Common Exam Patterns

- "Troubleshoot why EC2 can't reach the internet" → check Flow Logs for REJECT on the ENI.
- "Analyse network traffic patterns" → Flow Logs → S3 → Athena (SQL queries).
- "Monitor for suspicious traffic" → Flow Logs → CloudWatch Logs → Metric Filter for REJECT spikes.
- REJECT on inbound = **NACL or SG** blocked it. REJECT on outbound = **NACL** blocked it (SGs are stateful, they allow return traffic).

---

### X-Ray — Distributed Tracing

Trace requests as they travel through your distributed application.

#### Key Concepts

- **Trace** — end-to-end request path.
- **Segment** — work done by a single service.
- **Subsegment** — more granular (e.g., an HTTP call to a downstream service, a SQL query).
- **Service map** — visual graph of your application's services and their connections.
- **Annotations** — indexed key-value pairs for filtering traces.
- **Metadata** — non-indexed data attached to segments.

#### Integration

- **Lambda** — built-in; enable Active Tracing in function config.
- **API Gateway** — enable tracing in stage settings.
- **ECS/EKS** — run X-Ray daemon as a sidecar container.
- **EC2** — install X-Ray daemon.
- **Elastic Beanstalk** — built-in option.

#### Sampling

- Default: first request each second + 5% of additional requests.
- Custom sampling rules to control cost and volume.

> **Exam focus:** X-Ray is the answer when the question asks about tracing requests across microservices, identifying bottlenecks, or debugging latency in distributed architectures.

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| CloudWatch custom metric resolution | 1 second (high-res) to 1 minute (standard) |
| CloudWatch Logs retention | 1 day to 10 years, or never expire |
| CloudWatch Alarms per region | 5,000 (soft limit) |
| CloudTrail event delivery lag | ~5 minutes (not real-time) |
| CloudTrail event history | 90 days (free, no setup) |
| CloudTrail max trails per region | 5 |
| VPC Flow Log max aggregation interval | 1 minute or 10 minutes |
| X-Ray trace retention | 30 days |
| X-Ray segment max size | 64 KB |

---

## When To Use

| Scenario | Solution |
|---|---|
| Monitor EC2 CPU, network, disk I/O | CloudWatch default metrics |
| Monitor EC2 memory or disk usage | CloudWatch Agent (unified) |
| Alert when CPU > 80% for 5 minutes | CloudWatch Alarm → SNS |
| Real-time log analysis | CloudWatch Logs Insights |
| Stream logs to S3 in near-real-time | Subscription Filter → Kinesis Firehose → S3 |
| "Who made this API call?" | CloudTrail |
| Detect unusual API activity | CloudTrail Insights |
| Audit all API calls across org | Organisation trail (all-regions) |
| Troubleshoot network connectivity | VPC Flow Logs |
| Analyse network traffic with SQL | Flow Logs → S3 → Athena |
| Trace request across microservices | X-Ray |
| Identify latency bottleneck in Lambda→DynamoDB→SQS | X-Ray service map |

---

## Exam Traps

- **CloudWatch does NOT collect EC2 memory/disk by default** — you need the CloudWatch Agent. This is the #1 trap.
- **CloudTrail is NOT real-time** — ~5 minute delivery lag. For real-time alerting: CloudTrail → CloudWatch Logs → Metric Filter → Alarm.
- **VPC Flow Logs capture metadata, NOT packet contents** — for packet inspection, use VPC Traffic Mirroring or a third-party tool.
- **Flow Logs cannot be modified after creation** — delete and recreate with new config.
- **Log export to S3 (CreateExportTask) is NOT real-time** — up to 12 hours. Use subscription filter for real-time.
- **CloudWatch Logs ≠ CloudTrail** — CloudWatch Logs stores application/OS logs; CloudTrail logs API calls. They integrate but are different services.
- **X-Ray traces requests, CloudWatch monitors metrics** — different tools for different problems. X-Ray = "why is this request slow?" CloudWatch = "is my CPU high?"
- **VPC Flow Log REJECT analysis** — inbound REJECT = SG or NACL; outbound REJECT = NACL only (SG is stateful).
- **CloudTrail data events are OFF by default** — S3 GetObject, Lambda Invoke, DynamoDB GetItem are NOT logged unless you enable data events.
- **Organisation trail** — logs management events for all accounts automatically. Data events still need explicit enable per account.

---

## Related Files

| File | Purpose |
|---|---|
| `concepts/security.md` | GuardDuty consumes CloudTrail + Flow Logs |
| `concepts/analytics.md` | Athena queries on Flow Logs / CloudTrail in S3 |
| `concepts/messaging.md` | EventBridge (successor to CloudWatch Events) |
| `diagrams/security.md` | GuardDuty → EventBridge → Lambda flow |
| `cheatsheets/security.md` | Quick-reference for detection services |
| `labs/guardduty-eventbridge.md` | Hands-on with CloudTrail-based threat detection |
