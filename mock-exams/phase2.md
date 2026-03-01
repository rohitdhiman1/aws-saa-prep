# Practice Questions: Phase 2 — Compute & Storage

*20 questions · Target score: ≥ 70% · Complete end of Week 4*

---

## Instructions

Answer each question before reading the answer. Write your choice, then compare.
Log wrong answers in `docs/bugbase.md` with the correct reasoning.

---

**Q1.**
A company runs a batch processing workload that can be interrupted. It must run on EC2 and minimize cost. Which purchasing option should the solutions architect choose?

A) On-Demand Instances
B) Reserved Instances (1-year, no upfront)
C) Spot Instances
D) Dedicated Hosts

**Answer: C** — Spot Instances offer up to 90% discount over On-Demand and are ideal for fault-tolerant, interruptible workloads like batch jobs. On-Demand (A) is for unpredictable needs with no interruption tolerance. Reserved (B) saves ~40–60% but requires commitment and is best for steady-state. Dedicated Hosts (D) are for licensing or compliance requirements and are the most expensive.

---

**Q2.**
An Auto Scaling Group is configured with a minimum of 2, desired of 4, and maximum of 8 instances. A scale-out policy triggers due to high CPU. The policy adds 2 instances. The cooldown period has not yet elapsed. What happens?

A) The new instances are launched immediately, ignoring the cooldown
B) The scale-out action is blocked until the cooldown period expires
C) The ASG terminates 2 existing instances first, then launches 2 new ones
D) The ASG launches 4 instances instead of 2 to compensate for the delay

**Answer: B** — The cooldown period (default 300 seconds) prevents the ASG from launching or terminating additional instances after a scaling activity, giving time for metrics to stabilize. No new actions are taken until the cooldown expires.

---

**Q3.**
A web application uses an Application Load Balancer in front of EC2 instances. The application needs to know the real client IP address, but instances only see the ALB's IP. Where is the real client IP available?

A) In the `X-Real-IP` HTTP header
B) In the `X-Forwarded-For` HTTP header
C) In the EC2 instance's network interface metadata
D) In VPC Flow Logs only

**Answer: B** — ALBs add `X-Forwarded-For` headers containing the original client IP. `X-Real-IP` (A) is an Nginx convention, not an ALB header. The instance's network interface (C) shows the ALB's IP. Flow Logs (D) capture the data but are not accessible to the application at runtime.

---

**Q4.**
A company needs to run a containerized microservice that scales from zero to hundreds of tasks on demand. The ops team does not want to manage EC2 instances. Which compute option is most appropriate?

A) ECS on EC2 with Auto Scaling
B) ECS on Fargate
C) EC2 with Docker installed manually
D) Lambda with container image support

**Answer: B** — ECS Fargate is serverless container compute — AWS manages the underlying infrastructure. Tasks scale up and down without the ops team managing EC2 capacity. ECS on EC2 (A) requires managing an EC2 cluster. Lambda (D) supports containers but has a 15-minute max execution time.

---

**Q5.**
A Lambda function is invoked by SQS. The function occasionally fails. After exhausting retries, messages should be preserved for manual inspection. What should the solutions architect configure?

A) Set Lambda reserved concurrency to 0 to pause processing
B) Configure a Dead Letter Queue on the SQS queue with a `maxReceiveCount`
C) Enable Lambda Destinations with an OnFailure SQS target
D) Increase the SQS visibility timeout to 1 hour

**Answer: B** — For SQS-triggered Lambda, the SQS queue's own DLQ (configured via `maxReceiveCount` on the source queue's redrive policy) captures messages that fail after the configured number of retries. Lambda Destinations (C) work for async invocations (S3, SNS), not SQS event source mappings. Visibility timeout (D) only delays retries.

---

**Q6.**
A company stores user profile images in S3. Images are accessed frequently in the first 30 days, rarely after that. After 1 year they can be deleted. What is the most cost-effective lifecycle policy?

A) Transition to S3 Glacier after 30 days, delete after 1 year
B) Transition to S3 Standard-IA after 30 days, transition to S3 Glacier Flexible Retrieval after 90 days, delete after 1 year
C) Keep all objects in S3 Standard, delete after 1 year
D) Transition to S3 One Zone-IA immediately, delete after 1 year

**Answer: B** — Standard-IA is appropriate after 30 days (infrequent access, but still occasionally retrieved). Glacier Flexible after 90 days (cold archival). Standard-IA has a 30-day minimum storage duration and Glacier Flexible has a 90-day minimum — these timings are correct. Moving straight to Glacier at day 30 (A) skips the IA middle tier. Standard (C) is expensive for rarely-accessed data.

---

**Q7.**
An EC2 instance's root EBS volume needs to be encrypted. The instance is currently running with an unencrypted volume. What is the correct approach?

A) Modify the EBS volume in-place to enable encryption
B) Create a snapshot, copy the snapshot with encryption enabled, create a new volume from the snapshot, swap the volumes
C) Enable EBS encryption by default — existing volumes are automatically encrypted
D) Terminate and re-launch the instance with an encrypted AMI

**Answer: B** — EBS encryption cannot be changed in-place. The standard approach: snapshot → copy with encryption → create volume → swap. Enabling default encryption (C) only affects new volumes. Terminating the instance (D) destroys state unnecessarily.

---

**Q8.**
An application stores session state in memory on EC2 instances behind an ALB. Users lose their session on each request because they hit different instances. What is the recommended fix?

A) Enable sticky sessions on the ALB target group
B) Move session state to ElastiCache or DynamoDB and make the application stateless
C) Use a Network Load Balancer instead of an ALB
D) Add more EC2 instances to improve the odds of hitting the same one

**Answer: B** — The architecturally correct answer is to externalize session state to a shared store (ElastiCache or DynamoDB), making EC2 instances stateless. Sticky sessions (A) solve the symptom but create uneven load distribution and fail if the pinned instance goes down. NLB (C) doesn't help with session affinity.

---

**Q9.**
A company stores financial records in S3. Compliance requires records cannot be deleted or modified for 7 years. Which S3 feature should the solutions architect enable?

A) S3 Versioning with MFA Delete
B) S3 Object Lock with Compliance mode and a 7-year retention period
C) S3 Replication to a separate account
D) S3 Object Lock with Governance mode and a 7-year retention period

**Answer: B** — Object Lock Compliance mode prevents anyone — including the root user — from deleting or overwriting objects during the retention period. Governance mode (D) can be overridden by users with special IAM permissions — not suitable for hard compliance. MFA Delete (A) adds a step but doesn't prevent deletion. Replication (C) provides redundancy, not immutability.

---

**Q10.**
A Lambda function runs in response to an S3 `PutObject` event and fails. What is the default retry behavior for this asynchronous invocation?

A) Lambda retries once immediately, then drops the event
B) Lambda retries twice (3 total attempts) with delays between retries
C) Lambda does not retry asynchronous invocations
D) S3 retries the event every 60 seconds for 24 hours

**Answer: B** — For asynchronous Lambda invocations (S3, SNS, EventBridge), Lambda retries failed executions twice after the initial attempt (3 total), with delays. After all retries, the event goes to the DLQ (if configured) or is discarded. S3 fires the event once — Lambda owns the retry logic.

---

**Q11.**
A company runs an e-commerce application on EC2 in two regions. Users in Europe should be routed to eu-west-1 and users in the US to us-east-1, with automatic failover. Which service handles this?

A) ALB with path-based routing rules
B) Route 53 with geolocation routing and health checks
C) CloudFront with multiple origins configured by geography
D) Global Accelerator with endpoint groups per region

**Answer: B** — Route 53 geolocation routing directs users based on geographic location. Health checks enable automatic failover if a region becomes unhealthy. ALB (A) operates within a single region. Global Accelerator (D) routes by lowest latency/proximity, not strict geolocation control.

---

**Q12.**
An EC2 instance needs to share a file system with 10 other EC2 instances across three Availability Zones, with concurrent read/write access. Which storage solution is most appropriate?

A) EBS Multi-Attach
B) S3
C) EFS
D) Instance Store

**Answer: C** — Amazon EFS is a managed NFS file system mountable concurrently by thousands of EC2 instances across multiple AZs. EBS Multi-Attach (A) is limited to a single AZ and requires the application to handle concurrent write coordination. S3 (B) is object storage, not a POSIX file system. Instance Store (D) is ephemeral and local to one instance.

---

**Q13.**
A company wants to replicate S3 objects from us-east-1 to eu-west-1 for DR. Existing objects must also be replicated. Which statements are correct? (Select TWO)

A) CRR automatically replicates existing objects when configured
B) S3 Batch Operations can be used to copy existing objects to trigger replication
C) Versioning must be enabled on both source and destination buckets
D) CRR only works between different AWS accounts
E) Delete markers are replicated by default

**Answer: B, C** — CRR only replicates new objects after the rule is created. S3 Batch Operations with a replication job is the way to replicate existing objects. Versioning is a hard requirement for both source and destination. CRR works within the same account or across accounts (D is wrong). Delete markers are NOT replicated by default — you must explicitly enable it (E is wrong).

---

**Q14.**
A Lambda function that processes images has a 3–5 second cold start. It runs occasionally throughout the day. What is the most cost-effective way to eliminate cold starts?

A) Increase Lambda memory to the maximum (10,240 MB)
B) Use Provisioned Concurrency to keep execution environments initialized
C) Switch to ECS Fargate
D) Set reserved concurrency to 10

**Answer: B** — Provisioned Concurrency keeps Lambda environments pre-initialized and eliminates cold starts. You pay for the provisioned concurrency while allocated, but it's cheaper than running a persistent container service. Increasing memory (A) speeds up execution but doesn't eliminate initialization time. Reserved concurrency (D) caps concurrent executions but doesn't prevent cold starts.

---

**Q15.**
An application writes large sequential data to an EBS volume. Data is written once and never modified. Which EBS volume type provides the best throughput-to-cost ratio?

A) gp3
B) io2
C) st1 (Throughput Optimized HDD)
D) sc1 (Cold HDD)

**Answer: C** — `st1` is designed for large sequential workloads (log processing, big data) with high throughput at lower cost than SSDs. `gp3` (A) is general purpose SSD — good balance but costs more per GB than HDD. `io2` (B) is high-IOPS SSD — overkill for sequential throughput. `sc1` (D) has the lowest throughput and is for cold sequential access, not active writes.

---

**Q16.**
A company's ASG uses a launch template. A new AMI is available with security patches. How should the architect update ASG instances with minimal disruption?

A) Update the launch template — the ASG automatically replaces all instances
B) Update the launch template, then trigger an Instance Refresh with a minimum healthy percentage
C) Terminate all instances — the ASG launches new ones with the old AMI
D) Create a new ASG with the new AMI and redirect the ALB to it

**Answer: B** — Updating the launch template does NOT automatically replace running instances. Instance Refresh performs a rolling replacement, respecting `minHealthyPercentage` to avoid downtime. Terminating all instances (C) causes an outage. Blue/green with a new ASG (D) is valid but more complex than needed.

---

**Q17.**
A Docker container running on ECS Fargate needs access to an S3 bucket. How should the solutions architect grant access?

A) Store AWS access keys in the container's environment variables
B) Attach an IAM task role to the ECS task definition with S3 permissions
C) Attach an IAM instance profile to the underlying Fargate host
D) Configure an S3 bucket policy to allow all Fargate IP addresses

**Answer: B** — ECS task roles are the correct mechanism. The task role is specified in the task definition, and ECS provides temporary credentials via the container metadata endpoint automatically picked up by the AWS SDK. Environment variable keys (A) are a security anti-pattern. Fargate is serverless — there's no EC2 host to attach an instance profile to (C). Fargate IPs are dynamic — an IP-based bucket policy (D) is not maintainable.

---

**Q18.**
An S3 bucket hosts a static website with public read access. Users can access the site via the S3 URL but get `Access Denied` via the CloudFront URL. What is the most likely cause?

A) CloudFront doesn't support S3 static websites as origins
B) The CloudFront distribution uses OAC and is configured to the S3 REST endpoint, but the bucket policy only grants public read — not the OAC principal
C) The S3 bucket and CloudFront are in different regions
D) The bucket policy doesn't include CloudFront's IP ranges

**Answer: B** — When a CloudFront distribution uses OAC, it accesses S3 via the REST API endpoint (`bucket.s3.amazonaws.com`). If the bucket policy only allows public reads and doesn't include the OAC principal (`cloudfront.amazonaws.com`), CloudFront gets `Access Denied`. The S3 direct URL works because it matches the public read policy. CloudFront is global — region doesn't matter (C). CloudFront uses many IPs dynamically — IP-based policies (D) don't work.

---

**Q19.**
An ALB health check is set to check `/health` every 30 seconds with an unhealthy threshold of 2. An instance fails one health check. What happens?

A) The instance is immediately marked unhealthy and terminated
B) The instance continues receiving traffic until the second consecutive failed check, then the ALB stops sending traffic to it
C) The ASG immediately launches a replacement instance
D) The instance is deregistered and the ASG terminates it after the first failure

**Answer: B** — The ALB marks a target unhealthy only after the configured number of consecutive failed checks (threshold = 2). After one failure, traffic continues. After the second, the ALB marks it unhealthy and stops routing to it. The ASG may eventually replace it, but that's a separate process from ALB health check evaluation.

---

**Q20.**
An IAM role generates pre-signed S3 URLs for users. The role has a 1-hour session duration. The pre-signed URL is set to expire in 6 hours. What is the effective expiry?

A) 6 hours as configured in the URL
B) 1 hour — when the role's session expires, the pre-signed URL becomes invalid
C) 12 hours — AWS extends expiry to align with the next credential rotation
D) Indefinitely — pre-signed URLs are validated at creation time

**Answer: B** — Pre-signed URL validity is bounded by the credentials used to sign the URL. If the IAM role session expires in 1 hour, the pre-signed URL also becomes invalid at that point, regardless of the configured 6-hour expiry. This is a classic exam trap — the shorter of (URL expiry, credential expiry) wins.

---

## Score Log

| Date | Score | Notes |
|---|---|---|
| | /20 | |
