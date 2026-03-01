# Cheatsheet: Cost Optimisation

*Build this during Week 12 review. Key exam domain: Cost Optimised (20%).*

---

## Compute

| Scenario | Cost-Optimal Choice |
|---|---|
| Predictable 24/7 workload, 1-year commitment | Reserved Instance or Savings Plans |
| Flexible workload, any instance type/region | Compute Savings Plans |
| Fault-tolerant batch | Spot Instances (up to 90% off) |
| Dev/test (off during nights/weekends) | Schedule: stop instances or use Auto Scaling with scheduled actions |
| Container workload | Fargate Spot (up to 70% off Fargate price) |

## Storage

| Scenario | Cost-Optimal Choice |
|---|---|
| Infrequently accessed data, known pattern | S3 Standard-IA (30-day minimum) |
| Unknown/variable access pattern | S3 Intelligent-Tiering (auto-tiers) |
| Archive, rarely accessed | S3 Glacier Deep Archive |
| Block storage, general | gp3 (cheaper + more flexible than gp2) |
| Shared Linux file storage | EFS with Intelligent-Tiering lifecycle |

## Networking

| Scenario | Cost Consideration |
|---|---|
| Data egress | Minimise cross-AZ and inter-region transfers |
| S3 access from EC2 | Gateway endpoint (free, avoids NAT GW cost) |
| NAT Gateway | One per AZ; avoid cross-AZ NAT traffic |
| CloudFront | Reduces origin load + data transfer cost |

## Database

| Scenario | Cost-Optimal Choice |
|---|---|
| Variable DB load | Aurora Serverless v2 (pay per ACU-second) |
| Read scaling | Aurora replicas < RDS Read Replicas cost |
| NoSQL, variable traffic | DynamoDB on-demand |
| Data warehouse, short jobs | Redshift Serverless or pause/resume cluster |

## Key Cost Principles

1. **Pay for what you use** — serverless where possible.
2. **Commit for discounts** — Reserved/Savings Plans for steady-state.
3. **Spot for fault-tolerant** — stateless workloads.
4. **Right-size** — use AWS Compute Optimizer recommendations.
5. **S3 lifecycle policies** — automatically tier down to cheaper classes.
6. **Delete unused resources** — snapshots, AMIs, unused EBS volumes.

---

## Related Files

`docs/roadmap.md` · `mock-exams/exam3.md` · All cheatsheets (cross-reference)
