# Cheatsheet: Compute

*Fill in as you complete Week 3. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| Spot interruption notice | 2 minutes |
| Lambda max duration | 15 minutes |
| Lambda concurrency default | 1,000/region (soft limit) |
| Lambda max memory | 10,240 MB (CPU scales with memory) |
| Lambda reserved concurrency | Caps max + guarantees capacity (free) |
| Lambda provisioned concurrency | Pre-warms environments (extra cost) |
| Lambda SQS visibility timeout | Must be ≥ 6x function timeout |
| Lambda alias traffic shifting | Split traffic between 2 versions (canary) |
| ASG cooldown default | 300 seconds |
| ALB: static IP | No (use NLB) |
| NLB: static IP | Yes (Elastic IP) |
| Placement group (Spread) | 7 instances per AZ |

---

## Must-Know Distinctions

- **ALB** → L7, path/host routing, Lambda targets | **NLB** → L4, static IP, PrivateLink
- **Reserved** → commitment discount | **Savings Plans** → flexible commitment
- **Spot** → interruptible, cheapest | **On-Demand** → no commitment, full price
- **Dedicated Host** → BYOL, socket control | **Dedicated Instance** → single-tenant, no socket control
- **ECS Task execution role** → infrastructure | **ECS Task role** → app code
- **Lambda Provisioned Concurrency** → no cold starts, costs money | **Reserved Concurrency** → cap max, free config
- **Lambda Destinations** → success + failure routing, richer payload | **DLQ** → failure only, original event only
- **Lambda Version** → immutable snapshot | **Alias** → named pointer, can split traffic
- **Bisect batch on error** → Kinesis/DynamoDB poison pill fix | **Partial batch response** → SQS individual message retry

---

## Exam Traps (Quick List)

- Lambda in VPC adds cold start latency — use Provisioned Concurrency
- Reserved concurrency = 0 disables the function (emergency kill switch)
- SQS → Lambda: without `ReportBatchItemFailures`, one bad message retries entire batch
- Kinesis → Lambda: poison record blocks shard — enable bisect batch on error
- Never point production at $LATEST — use an alias to a specific version
- Destinations preferred over DLQ (richer payload, handles success too)
- ALB preserves source IP only in headers (X-Forwarded-For); NLB preserves natively
- Fargate doesn't support privileged containers
- Cross-zone LB: ALB = on/free; NLB = off/extra cost
- Beanstalk Immutable = safest rollback

---

## Related Files

`concepts/compute.md` · `labs/compute-asg-alb.md` · `diagrams/compute.md` · `mock-exams/phase2.md`
