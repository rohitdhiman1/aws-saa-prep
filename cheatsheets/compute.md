# Cheatsheet: Compute

*Fill in as you complete Week 3. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| Spot interruption notice | 2 minutes |
| Lambda max duration | 15 minutes |
| Lambda concurrency default | 1,000/region |
| Lambda max memory | 10,240 MB |
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
- **Lambda Provisioned Concurrency** → no cold starts | **Reserved Concurrency** → cap max

---

## Exam Traps (Quick List)

- Lambda in VPC adds cold start latency
- ALB preserves source IP only in headers (X-Forwarded-For); NLB preserves source IP natively
- Fargate doesn't support privileged containers
- Cross-zone LB: ALB = on/free; NLB = off/extra cost
- Beanstalk Immutable = safest rollback

---

## Related Files

`concepts/compute.md` · `labs/compute-asg-alb.md` · `diagrams/compute.md` · `mock-exams/phase2.md`
