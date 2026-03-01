# Practice Questions: Phase 5 — Messaging, Analytics & Security

*20 questions · Target score: ≥ 75% · Complete end of Week 10*

---

## Instructions

Answer each question before reading the answer. Write your choice, then compare.
Log wrong answers in `docs/bugbase.md` with the correct reasoning.

---

**Q1.**
An application publishes order events to an SQS queue. Multiple consumers are polling the queue. The same order is being processed by two different consumers simultaneously. What should the solutions architect adjust?

A) Enable FIFO mode on the SQS queue
B) Increase the visibility timeout to be greater than the maximum processing time
C) Enable long polling on the queue
D) Reduce the number of consumers

**Answer: B** — The visibility timeout hides a message from other consumers while one consumer is processing it. If processing takes longer than the visibility timeout, the message becomes visible again and another consumer picks it up — causing duplicate processing. Setting the visibility timeout higher than the worst-case processing time prevents this. FIFO (A) ensures ordering and exactly-once processing but requires code changes. Long polling (C) reduces empty receives but doesn't fix duplicate processing.

---

**Q2.**
A company uses SNS to fan out messages to multiple downstream services. A new requirement says that the billing service should only receive order events where the total is greater than $1,000. Which SNS feature enables this without creating a separate topic?

A) SNS message filtering using a subscription filter policy
B) Create a Lambda function to evaluate each message and forward selectively
C) Use an SQS FIFO queue to re-order messages before the billing service
D) Enable SNS raw message delivery

**Answer: A** — SNS subscription filter policies allow each subscription to declare which messages it wants to receive based on message attributes. The billing service's SQS subscription gets a filter policy `{"order_total": [{"numeric": [">=", 1000]}]}`. Messages not matching the filter are not delivered to that subscriber, at no additional cost. Lambda filtering (B) works but adds complexity and cost.

---

**Q3.**
A company processes streaming clickstream data from a website. They need to analyze the data in real-time and store it in S3 for batch analytics later. Which combination of services should they use?

A) SQS → Lambda → S3
B) Kinesis Data Streams → Kinesis Data Firehose → S3, with Lambda or KDA for real-time processing
C) SNS → SQS → Lambda → S3
D) EventBridge → Step Functions → S3

**Answer: B** — Kinesis Data Streams is designed for real-time streaming data ingestion with multiple consumers. Kinesis Data Firehose delivers data to S3 (and other destinations) with optional transformation. Kinesis Data Analytics (KDA) or Lambda can process the stream in real-time. SQS (A, C) is for message queuing, not streaming analytics — it doesn't support multiple independent consumers reading the same data.

---

**Q4.**
An SQS queue has a Dead Letter Queue (DLQ) configured with `maxReceiveCount = 3`. A Lambda function fails to process a message on the first two attempts. On the third attempt, it succeeds. What happens to the message?

A) The message is moved to the DLQ after 3 total receive counts
B) The message is successfully processed and deleted from the queue — it never reaches the DLQ
C) The message is moved to the DLQ after the first failure regardless of eventual success
D) The message remains in the queue until the DLQ retention period expires

**Answer: B** — `maxReceiveCount = 3` means the message is moved to the DLQ only if it is received (and returned to the queue, i.e., not deleted) 3 times without being processed successfully. If Lambda processes and deletes the message on the third attempt, the receive count resets and the DLQ is never involved. The DLQ only receives messages that fail all 3 processing attempts.

---

**Q5.**
A company runs AWS Glue ETL jobs to transform data from S3 and load it into Redshift. The Glue job reads data but the Redshift team reports missing records. What is the most likely cause?

A) Glue does not support writing to Redshift
B) The Glue job's IAM role lacks permission to write to Redshift
C) The Glue Data Catalog is not configured
D) Redshift Spectrum is not enabled

**Answer: B** — Glue ETL jobs use an IAM role for all operations. If the role lacks the permission to connect to and write to Redshift (typically via JDBC with credentials from Secrets Manager or a Glue connection), the write silently fails or throws an error. Glue does support Redshift (A is wrong). The Data Catalog (C) is for metadata; Glue can write to Redshift without it. Redshift Spectrum (D) is for querying S3 from Redshift, not for receiving Glue writes.

---

**Q6.**
A company wants to query data in S3 using standard SQL without loading it into a database. The data is in Parquet format, partitioned by year/month/day. Which service should they use, and what should they do to minimize query cost?

A) Redshift Spectrum — add the S3 data as an external table
B) Amazon Athena — define the table with partition projection or run `MSCK REPAIR TABLE` to load partitions, then query with a `WHERE` clause on the partition columns
C) AWS Glue — run a crawler then query via the Data Catalog
D) EMR — launch a cluster and run Hive queries

**Answer: B** — Athena is the serverless SQL query service for S3. Athena charges per data scanned ($5/TB). Parquet is a columnar format that reduces data scanned. Querying with `WHERE year = 2025 AND month = 3` prunes partitions so Athena only scans the matching S3 prefixes — dramatically reducing cost. Redshift Spectrum (A) also works but requires a running Redshift cluster. Glue (C) catalogs data but doesn't query it directly. EMR (D) is more complex and expensive for ad-hoc queries.

---

**Q7.**
A company creates a Customer Managed Key (CMK) in KMS. A developer encrypts an S3 object using the CMK. The CMK is later deleted (after the 7-day waiting period). What happens to the encrypted S3 object?

A) The object is automatically decrypted and stored in plaintext
B) The object is permanently unreadable — the encrypted data key cannot be decrypted without the CMK
C) S3 automatically re-encrypts the object with the AWS managed key
D) KMS provides a recovery window of 30 days to restore the object

**Answer: B** — When a CMK is deleted, any data encrypted with that CMK becomes permanently unreadable. The encrypted data key stored alongside the S3 object cannot be decrypted, so the data is lost. This is why KMS has a mandatory waiting period (7–30 days) before deletion — to prevent accidental key deletion. There is no automatic re-encryption (C) or recovery (D) after a CMK is deleted.

---

**Q8.**
An application must store database credentials securely and automatically rotate them every 30 days. The application retrieves credentials at runtime. Which service is most appropriate?

A) AWS Systems Manager Parameter Store (Standard, plaintext)
B) AWS Systems Manager Parameter Store (SecureString with KMS)
C) AWS Secrets Manager
D) Environment variables in Lambda

**Answer: C** — Secrets Manager is purpose-built for credentials with built-in automatic rotation support for RDS, Redshift, DocumentDB, and custom secrets. It integrates with the AWS SDK so applications retrieve the current secret at runtime. SSM Parameter Store (B) can store secrets but does not have native automatic rotation. Lambda environment variables (D) are not automatically rotated and are visible in the console.

---

**Q9.**
A company wants to use AWS WAF to block SQL injection attacks targeting their ALB. Which WAF rule type should they add?

A) A rate-based rule with a limit of 100 requests per 5 minutes
B) An IP set rule blocking known malicious IPs
C) The AWS Managed Rules `AWSManagedRulesSQLiRuleSet` rule group
D) A regex match rule with a custom SQL injection pattern

**Answer: C** — AWS Managed Rules provide pre-built rule groups maintained by the AWS threat intelligence team, including `AWSManagedRulesSQLiRuleSet` which covers common SQL injection patterns. Using managed rules requires no custom regex writing. A rate-based rule (A) limits request rates but doesn't inspect SQL patterns. IP set rules (B) block specific IPs, not injection content. Custom regex (D) works but is harder to maintain and may miss new attack variants.

---

**Q10.**
GuardDuty generates a `UnauthorizedAccess:EC2/SSHBruteForce` finding for an EC2 instance. What does this finding indicate?

A) An EC2 instance is running unauthorized software
B) An external IP is making many SSH login attempts against an EC2 instance's port 22
C) The EC2 instance's SSH key has been compromised
D) GuardDuty detected that SSH is disabled on the instance

**Answer: B** — The `SSHBruteForce` finding means GuardDuty detected an unusually high number of SSH connection attempts from an external IP, which is characteristic of a brute force attack. GuardDuty doesn't know if the key is compromised (C) or whether SSH is enabled/disabled (D) — it analyzes VPC Flow Logs and sees the connection pattern. The finding indicates a potential attack, not confirmed unauthorized access.

---

**Q11.**
A company wants to automatically remediate a GuardDuty finding by isolating the affected EC2 instance. Which combination of services achieves automated remediation?

A) GuardDuty → SNS → Email to the security team
B) GuardDuty → EventBridge → Lambda (modifies the instance's security group to block traffic)
C) GuardDuty → CloudWatch → CloudTrail
D) GuardDuty → AWS Config → Auto Remediation

**Answer: B** — GuardDuty publishes findings as events to EventBridge. An EventBridge rule triggers a Lambda function that reads the finding, identifies the affected instance, and modifies its security group (e.g., removes inbound rules to isolate it). This is fully automated. SNS email (A) is notification, not remediation. CloudTrail (C) records API calls, not remediation. Config (D) remediates configuration compliance issues, not GuardDuty findings.

---

**Q12.**
A company processes financial transactions with an SQS Standard queue and Lambda. They report that occasionally, transactions are processed twice. What is the cause and the recommended solution?

A) Cause: Lambda retries. Solution: Increase Lambda reserved concurrency.
B) Cause: SQS Standard delivers at-least-once — duplicate deliveries are possible. Solution: Implement idempotency in the Lambda function or switch to SQS FIFO.
C) Cause: The visibility timeout is too long. Solution: Decrease it.
D) Cause: The Lambda function is timing out. Solution: Increase Lambda memory.

**Answer: B** — SQS Standard guarantees at-least-once delivery — messages may occasionally be delivered more than once. The solutions are: (1) make the consumer idempotent (check if the transaction ID was already processed), or (2) use SQS FIFO which guarantees exactly-once processing. Concurrency (A) and memory (D) don't prevent duplicate delivery at the SQS level.

---

**Q13.**
A company uses Kinesis Data Streams with 4 shards. Producers are hitting `ProvisionedThroughputExceededException` errors. What is the cause and the fix?

A) Cause: Too many consumers. Fix: Add more consumer applications.
B) Cause: The stream is at capacity (4 shards × 1 MB/s write = 4 MB/s total). Fix: Increase the shard count.
C) Cause: Kinesis doesn't support more than 2 producers. Fix: Use SQS instead.
D) Cause: The data retention period is too short. Fix: Increase to 7 days.

**Answer: B** — Each Kinesis shard supports 1 MB/s or 1,000 records/s for writes. With 4 shards, the stream handles 4 MB/s of writes total. If producers exceed this, they get throttled. The fix is to increase the shard count (resharding). Consumer count (A) doesn't affect producer write capacity. Retention period (D) affects how long data is available, not throughput.

---

**Q14.**
A company needs to collect application logs from thousands of EC2 instances, transform them, and deliver to S3 and an OpenSearch Service cluster. Which service is best suited for this pipeline?

A) SQS with Lambda
B) Kinesis Data Firehose
C) AWS Glue
D) SNS with multiple subscriptions

**Answer: B** — Kinesis Data Firehose is a managed delivery service that can receive streaming data, optionally transform it with Lambda, and deliver to multiple destinations including S3, OpenSearch Service, Redshift, and others. It scales automatically and requires no shard management. Glue (C) is batch ETL. SQS+Lambda (A) requires custom delivery logic to S3 and OpenSearch. SNS (D) delivers messages but doesn't buffer, transform, or batch-deliver to data stores.

---

**Q15.**
An S3 bucket policy requires `"aws:SecureTransport": "true"` in all requests. A developer using the AWS CLI to copy files gets an `Access Denied` error even though their IAM role has S3 write access. What is the cause?

A) The IAM role doesn't have permission to use HTTPS
B) The CLI is sending requests over HTTP instead of HTTPS — the bucket policy denies non-HTTPS requests
C) The bucket is in a different region from the CLI's configured region
D) The bucket policy condition doesn't apply to IAM users

**Answer: B** — The `aws:SecureTransport: false` condition in a bucket policy deny statement blocks any request that arrives over plain HTTP. The AWS CLI uses HTTPS by default, but if `--no-verify-ssl` or an HTTP endpoint is configured, the request is rejected. The developer should verify the CLI is using HTTPS (`https://` endpoints, no HTTP override). IAM roles don't have separate HTTPS permissions (A).

---

**Q16.**
A company wants to ensure that all EBS volumes created in their account are encrypted. New developers are frequently creating unencrypted volumes. What is the simplest solution?

A) Create an IAM policy denying `ec2:CreateVolume` without encryption
B) Enable EBS encryption by default in the account settings — all new volumes are automatically encrypted with the default KMS key
C) Use AWS Config with a remediation rule to encrypt volumes after creation
D) Create an SCP denying unencrypted volume creation

**Answer: B** — Enabling EBS encryption by default in EC2 settings (per region) automatically encrypts all new EBS volumes and snapshot copies, even if the user doesn't explicitly choose encryption. It's the simplest, most frictionless enforcement. An IAM policy (A) requires complex condition syntax. Config remediation (C) is reactive — volumes are created unencrypted first, then encrypted. SCPs (D) are for account-level control and harder to target specifically.

---

**Q17.**
A company uses EventBridge to route events from multiple AWS services to different targets. A developer creates a rule that sends all `aws.s3` events to a Lambda function. The Lambda function is never triggered even though S3 operations are occurring. What is most likely missing?

A) EventBridge cannot receive events from S3
B) S3 event notifications must be configured on the bucket to send events to EventBridge
C) Lambda needs a resource-based policy to allow EventBridge to invoke it
D) Both B and C may be missing

**Answer: D** — Two things are needed: (1) S3 event notifications must be enabled on the bucket with EventBridge as the destination (`aws:s3:SendNotifications` to EventBridge — done via S3 bucket properties); (2) EventBridge needs permission to invoke Lambda, which requires a Lambda resource-based policy (`events.amazonaws.com` as principal). Either missing piece breaks the pipeline.

---

**Q18.**
A company wants to analyze their CloudTrail logs to detect when any user disables CloudTrail logging. Which combination achieves real-time detection?

A) Athena query scheduled daily
B) CloudTrail → CloudWatch Logs → metric filter on `StopLogging` → CloudWatch Alarm → SNS notification
C) GuardDuty → EventBridge → SNS
D) Config rule checking CloudTrail enabled status every hour

**Answer: B** — CloudTrail logs all API calls. Sending logs to CloudWatch Logs allows metric filters to match specific API calls (e.g., `eventName = "StopLogging"`). A CloudWatch Alarm on this metric triggers an SNS notification for near-real-time alerting. Athena (A) is for ad-hoc queries, not real-time alerts. GuardDuty (C) does detect `Stealth:IAMUser/CloudTrailLoggingDisabled` but with some delay. Config (D) checks configurations, not API events, and polls hourly.

---

**Q19.**
A company uses AWS Lake Formation to manage data access for their data lake in S3. A data analyst reports they cannot query a specific S3 prefix via Athena even though their IAM policy allows S3 read access. What is the cause?

A) Athena requires a separate license for Lake Formation-governed data
B) Lake Formation has a separate permission model — IAM S3 access is not sufficient. The analyst needs Lake Formation table/column permissions granted
C) The analyst needs to assume an Athena service role
D) S3 access control lists (ACLs) are blocking the analyst

**Answer: B** — When Lake Formation governs a data lake, access is controlled at the Lake Formation level (database, table, column, row filter) rather than S3 bucket policies alone. An IAM policy allowing S3 reads is not sufficient — Lake Formation enforces its own permissions on top. The data lake admin must grant the analyst specific table or column permissions in Lake Formation. This is a frequently tested gotcha.

---

**Q20.**
A company's application uses SQS FIFO queues to ensure ordered processing of customer orders per customer ID. They use `CustomerID` as the message group ID. How does FIFO ordering work with multiple message groups?

A) All messages across all groups are processed in strict global order
B) Messages within the same message group ID are processed in order, but messages in different groups can be processed in parallel and out of order relative to each other
C) FIFO queues process one message at a time globally — no parallelism
D) Message group IDs are only used for deduplication, not ordering

**Answer: B** — SQS FIFO queues guarantee ordering within a message group (same `CustomerID`). Messages in different groups are independent and can be processed concurrently by different consumers. This allows parallelism across customers while maintaining per-customer order integrity. Global strict ordering (A) would require a single message group ID and eliminates all parallelism. Message group IDs control both ordering and parallelism (D is wrong).

---

## Score Log

| Date | Score | Notes |
|---|---|---|
| | /20 | |
